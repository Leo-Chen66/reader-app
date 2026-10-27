# 电子书阅读器 技术设计文档（TECH_DESIGN）

> 版本：v1.0 ｜ 日期：2026-10-27 ｜ 对应 PRD：v1.0（MVP）

## 一、技术选型

### 1.1 桌面框架：Electron
| 候选 | 优势 | 劣势 | 结论 |
|------|------|------|------|
| **Electron** | 生态最成熟、教程与资料多、调试方便（DevTools）、打包工具链完善 | 内存占用偏高、安装包大 | ✅ 选用 |
| Tauri | 体积小、内存低 | Rust 工具链学习成本高，前端需适配 WebView 差异 | 备选（v2 评估） |

理由：MVP 阶段优先「跑通 + 快速迭代」，Electron 的开发效率与资料丰富度最重要；性能优化不是本期瓶颈（纯文本渲染）。

### 1.2 各层选型
| 层 | 选型 | 说明 |
|----|------|------|
| 界面 | 原生 HTML/CSS/JS（无框架） | 界面极简，引入 React 收益为负 |
| 文件读取 | `<input type="file">` + FileReader / Node fs | Web MVP 用 FileReader，Electron 版用 fs |
| 编码识别 | UTF-8 优先探测，失败回退 GBK | 依据 BOM 与解码失败特征判断 |
| 进度存储 | localStorage（Web）/ electron-store（桌面） | 键：文件名，值：{scrollTop, ratio, fontSize, updatedAt} |
| 打包 | electron-builder | 产出 Windows 安装包 |

## 二、项目结构

```
reader-app/
├── AGENTS.md              # AI 代理开发约定
├── PRD.md                 # 产品需求文档
├── TECH_DESIGN.md         # 本文档
├── reader.html            # MVP Web 版（单文件，可直接打开）
├── main.js                # Electron 主进程（窗口管理、文件对话框）
├── preload.js             # 预加载脚本（安全暴露 IPC 能力）
├── renderer/
│   ├── index.html         # Electron 渲染层入口
│   ├── style.css
│   └── app.js             # 阅读逻辑（与 Web 版共用核心）
├── package.json
└── tests/                 # 手工测试记录
```

## 三、数据模型

```jsonc
// localStorage key: "reader:progress:<fileName>"
{
  "scrollTop": 12480,        // 滚动像素位置
  "ratio": 0.352,            // 阅读比例（0~1，用于跨窗口尺寸恢复）
  "fontSize": 18,            // 正文字号
  "updatedAt": 1730000000000 // 最后阅读时间戳
}
// 全局键 "reader:settings": { "fontSize": 18 }
```

**恢复策略**：优先按 `ratio` 恢复（对窗口尺寸变化鲁棒），`scrollTop` 作为同尺寸下的精确值。

## 四、关键技术点

### 4.1 TXT 编码识别（FR-06）
1. 读原始 ArrayBuffer，检查 BOM：`EF BB BF` → UTF-8；`FF FE` → UTF-16LE
2. 无 BOM 时先按 UTF-8 严格解码（`fatal: true`），抛错则回退 GBK（`TextDecoder('gbk')`）
3. 两者都失败 → 提示「无法识别的文件编码」

### 4.2 大文件渲染性能（10MB 约数百万字）
- 全量 innerHTML 拼接会卡顿 → 采用**分块渲染**：正文按段落切片，仅渲染可视区前后各一屏（虚拟滚动思路的轻量实现）
- MVP 先验证直渲性能，超过阈值（首帧 > 1s）再启用分块，避免过早优化

### 4.3 进度保存节流
- `scroll` 事件高频触发 → 500ms 节流写 localStorage，`beforeunload` 时强制补写一次

### 4.4 安全约定（Electron）
- `contextIsolation: true`，`nodeIntegration: false`
- 渲染进程不直接碰 fs，全部经 preload 暴露的白名单 IPC（`openFile`、`saveProgress`）

## 五、风险与对策

| 风险 | 影响 | 对策 |
|------|------|------|
| GBK 冷僻字解码异常 | 个别字符乱码 | 解码失败字符用 `U+FFFD` 兜底并在状态栏提示 |
| 超大文件（>50MB） | 渲染卡顿 | MVP 明确不支持并在 PRD 声明；v1.1 分块加载 |
| Electron 打包体积大 | 分发不便 | MVP 不做优化，v1.2 评估 Tauri 迁移 |

## 六、测试要点

1. 三种编码样本：UTF-8 带 BOM / UTF-8 无 BOM / GBK（各准备一份测试 TXT）
2. 进度恢复：不同窗口尺寸下重开，位置偏差 < 1 屏
3. 字号持久化：调节后重启保持
4. 空文件 / 二进制改后缀 .txt：友好报错不崩溃
