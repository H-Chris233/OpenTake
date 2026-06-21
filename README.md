<div align="center">
<img src="./assets/opentake-logo.png" alt="OpenTake Logo" width="120" />

# OpenTake

**Agent 原生的视频剪辑工作引擎 —— 让 Agent 内化、学习并蒸馏各大视频博主的剪辑能力与手法。**
</div>

---

OpenTake 是 [Palmier Pro](https://github.com/palmier-io/palmier-pro) 的跨平台社区分支 —— 在 **Rust 核心（Tauri 2 + React）** 上重建，媒体引擎采用 **FFmpeg**（编解码）+ **wgpu**（合成），忠实复刻其编辑逻辑，并内置更强的 Agent 上下文信号系统和工作流插件引擎。目标平台：**macOS / Windows / Linux**。

> 🌟 **核心理念：不让 Agent 去读技能文件，而是让软件主动向 Agent 发送剪辑指引。** 我们把影视飓风等专业视频博主的剪辑方法论内化为软件端的"信号发射器"——Agent 操作时间线时，软件直接告诉它"这条轨道是干什么的、这个素材该用什么手法剪、这一段该匹配哪类 B-roll"。Agent 从来不需要自己翻技能文档。

> ⚠️ **当前状态：早期设计阶段。** 本仓库已包含从上游逐模块拆解得出的架构设计、路线图、模块移植地图、Agent 上下文信号系统设计，以及工作流插件系统规划。代码即将落地。

## 核心特性

### 🔧 Rust 核心引擎

从上游 Swift/AVFoundation 栈**忠实复刻**全部编辑逻辑：
- **领域模型**：Timeline / Track / Clip / Keyframe / Transform —— 全部纯函数式值语义，1:1 移植到 Rust
- **编辑算法**：OverwriteEngine / RippleEngine / SnapEngine（含波纹拒绝语义、A/V 链接组联动、分割关键帧边界保持）
- **命令层**：UI 手势、Agent、MCP 三个前端归一到一个 `EditCommand` 枚举，撤销/校验/遥测只写一遍
- **撤销系统**：整树快照栈（Timeline derive Clone），每步操作自动压栈

### 🎬 跨平台媒体引擎

- **FFmpeg**（`ffmpeg-next`）：解码 / 编码 / 缩略图 / 波形 —— 成熟 Rust 绑定，代替 AVFoundation
- **wgpu**：自写帧合成器 —— 多轨叠加 + 逐帧属性采样 + 仿射/裁剪/混合，代替 AVVideoComposition 声明式合成器
- **cpal**：跨平台音频播放
- **whisper-rs**：端侧语音转写（word/segment 时间戳）
- **candle / ort**：本地语义搜索（SigLIP2 图文双编码器）

### 🧠 Agent 上下文信号系统

OpenTake 的核心创新：**软件主动向 Agent 发送剪辑指引**，而不是让 Agent 自己去读技能文件。

当 Agent 通过 MCP 操作时间线和轨道时，软件在每次工具返回中附带 `context_signal`：
- **视频类型判定**：口播 / Vlog / 混剪 / 采访 / 短剧 / 长视频 —— 自动识别，自动套用对应剪辑骨架
- **轨道角色标注**：主画面 / B-roll / 旁白 / BGM / SFX / 文字 —— Agent 看到轨道就知道该怎么做
- **剪辑阶段指引**：当前阶段 + 下一步建议 + 规则校验（气口三规则、B-roll 五注意、时钟理论、波峰制等）

这套系统的知识来源是 [ClipSkills](https://github.com/appergb/ClipSkills) 技能套件（MIT 许可）—— 12 册专业剪辑知识内核，融合了影视飓风《剪辑全能必修课》等专业课程的方法论，被内化为软件端的"信号发射器"。

详见 [docs/AGENT-CONTEXT-SIGNAL.md](docs/AGENT-CONTEXT-SIGNAL.md)。

### 🔌 MCP Server（31 个工具）

与上游 Palmier Pro 兼容的 MCP server（`127.0.0.1:19789`），对外暴露 31 个工具：

| 工具分组 | 数量 | 功能 |
|---|---|---|
| 读 / 内省 | 7 | get_timeline, get_media, inspect_media, search_media 等 |
| 时间线编辑 | 11 | add_clips, split_clip, set_clip_properties, set_keyframes, ripple_delete_ranges 等 |
| 媒体生成 / 导入 | 5 | generate_video, generate_image, generate_audio, import_media 等 |
| 库组织 | 7 | create_folder, move_to_folder, rename_media 等 |
| MCP Resources | 2 | models/video, models/image |

同时支持应用内 Agent chat（via Anthropic API / BYOK），与 MCP 共享同一套工具定义和系统提示词。

### 🌐 自建 / BYOK 生成式 AI

上游的生成式 AI 云服务为闭源，**不在本次分支中**。OpenTake 提供双模运行：

| 模式 | 说明 |
|---|---|
| **BYOK 模式** | 用户自带 fal.ai / Replicate / OpenAI key，本地直连厂商，**零后端、零运营成本** |
| **托管模式（可选）** | 自建轻量代理（opentake-gen-proxy），持厂商 key + 积分计费 |

### 📋 工作流插件系统

社区可为特定视频类型（科普、评测、游戏、婚礼……）编写 JSON + Markdown 轻量插件。每个插件封装了该类型视频的专业剪辑方法论，Agent 激活后自动获得完整的剪辑决策链。详见 [docs/WORKFLOW-PLUGIN-SYSTEM.md](docs/WORKFLOW-PLUGIN-SYSTEM.md)。

## 技术栈

| 关注点 | 选型 |
|---|---|
| 核心语言 | Rust（workspace，多 crate） |
| 桌面外壳 | Tauri 2 |
| 前端 | React + TypeScript + Vite |
| 状态管理 | Zustand（前端只读镜像） |
| 编解码 | FFmpeg（`ffmpeg-next`） |
| 帧合成 | wgpu（自写合成器） |
| 音频播放 | cpal |
| 转写 | whisper-rs（whisper.cpp） |
| 语义搜索 | candle / ort + SigLIP2 |
| MCP server | rmcp（streamable-http-server） |
| LLM 客户端 | reqwest + eventsource-stream |
| 生成代理 | axum（可选） |
| 密钥存储 | keyring |
| 序列化 | serde / serde_json |

## 架构一览

```
┌───────────────────────────────────────────────┐
│ React + TypeScript 前端                         │
│ TimelineView · Preview · Inspector · MediaPanel│
│ Zustand: Timeline 只读镜像 + UI-only 态         │
└───────────────┬───────────────────────────────┘
                │ Tauri invoke + event
┌───────────────▼───────────────────────────────┐
│ Rust core（真相源 + 全部领域逻辑）               │
│ domain · ops · project · render · media        │
│ agent · gen · core                              │
│         ▲               │                      │
│       MCP server         调用                   │
│  in-app agent            ▼                     │
│       media: FFmpeg + wgpu + cpal + whisper-rs │
└────────────────────────────────────────────────┘
```

详见 [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)。

## 文档

| 文档 | 内容 |
|---|---|
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | 目标架构、分层、crate 布局、命令层、渲染管线 |
| [ROADMAP.md](docs/ROADMAP.md) | 阶段实施路线图（Phase 0 - Phase 10），含验证标准与风险登记 |
| [MODULE-PORT-MAP.md](docs/MODULE-PORT-MAP.md) | 20 个上游模块逐项移植规格、核心算法、移植判定 |
| [AGENT-CONTEXT-SIGNAL.md](docs/AGENT-CONTEXT-SIGNAL.md) | Agent 上下文信号系统（视频类型判定 / 轨道角色标注 / 剪辑规则内化） |
| [WORKFLOW-PLUGIN-SYSTEM.md](docs/WORKFLOW-PLUGIN-SYSTEM.md) | 工作流插件系统设计（JSON + Markdown 轻量插件） |
| [ADVANCED-FEATURES.md](docs/ADVANCED-FEATURES.md) | 对标剪映的进阶能力设计（特效 / 转场 / 调色 / 蒙版 / AI 扣像） |
| [CAPCUT-GAP.md](docs/CAPCUT-GAP.md) | 与剪映模块 1-5 的 33 项特性差距分析 |
| [MOTION-GRAPHICS-PLUGIN.md](docs/MOTION-GRAPHICS-PLUGIN.md) | Web 动效模块与插件系统设计 |
| [DECISIONS.md](DECISIONS.md) | 技术栈 / 许可 / 品牌决策记录（ADR） |
| [docs/_analysis/](docs/_analysis/) | 4 份上游横切分析报告 + 原始子 Agent 输出 |

## 与 Palmier Pro 的关系

OpenTake 是**独立的社区分支**，与 Palmier, Inc. 无隶属关系，亦未获其赞助或背书。

"Palmier" / "Palmier Pro" 是其各自所有者的名称/商标，此处仅用于说明 OpenTake 的来源（指明性合理使用），并非 OpenTake 自身品牌。

OpenTake 依据 **GPL-3.0** 开源（与上游一致）。上游的生成式 AI 云服务为闭源、不属于本分支，相关能力由 OpenTake 自建。详见 [LICENSE](LICENSE) 与 [NOTICE](NOTICE)。

## 开发与贡献

### 本地开发（Phase 0+）

```bash
# Rust core
cargo build
cargo test
cargo clippy

# 前端
cd web && pnpm install && pnpm build

# 启动 Tauri 开发模式
cargo tauri dev
```

当前代码尚未产生，请参考 [ROADMAP.md](docs/ROADMAP.md) 了解当前阶段。欢迎在 [Issues](https://github.com/appergb/OpenTake/issues) 中提交建议或设计讨论。

### 上游参考

本项目的上游参考代码位于同目录 `palmier-pro-upstream/`。编辑逻辑的移植应首先查阅该目录的 Swift 源码（`Sources/PalmierPro/`）。

## 致谢与引用

OpenTake 建立在以下优秀开源项目的肩膀之上：

| 项目 | 许可证 | 用途 |
|---|---|---|
| [Palmier Pro](https://github.com/palmier-io/palmier-pro) | GPL-3.0 | 本项目为其社区分支。编辑逻辑和领域模型均在此基础上移植。 |
| [ClipSkills](https://github.com/appergb/ClipSkills) | MIT | 剪辑知识内核（12 册技能参考），融合影视飓风等专业课程的方法论，被内化为 OpenTake 的 Agent 上下文信号系统。 |
| [FFmpeg](https://ffmpeg.org) | LGPL-2.1+ / GPL-2.0+ | 媒体编解码引擎。通过 `ffmpeg-next` Rust 绑定动态链接，与 GPL-3.0 兼容。 |
| [Tauri](https://tauri.app) | MIT / Apache 2.0 | 跨平台桌面应用框架。 |
| [wgpu](https://wgpu.rs) | MIT / Apache 2.0 | GPU 渲染引擎，用于自写帧合成器。 |

以上所有引用的开源项目均按其各自的许可证条款使用。OpenTake 整体适用 **GPL-3.0**。

## 许可证

Copyright (C) 2026 OpenTake contributors

OpenTake 是自由软件：您可以依据自由软件基金会发布的 **GNU 通用公共许可证第三版（GPLv3）** 或（由您选择）任何更新版本的条款，再分发和/或修改本软件。

分发本软件是希望它有用，但**没有任何担保**；甚至没有适销性或特定用途适用性的默示担保。详见 [GNU 通用公共许可证](LICENSE)。

本程序基于 [Palmier Pro](https://github.com/palmier-io/palmier-pro)（Copyright (C) 2026 Palmier, Inc.），亦以 GPLv3 许可分发。详见 [NOTICE](NOTICE)。
