# Claude Code 架构拆解

**[English](README.md)** | **[中文](README_zh.md)**

基于[泄露源码](https://github.com/anthropics/claude-code)的 Claude Code 架构深度拆解 — 工具系统、Query Engine、IDE 桥接、隐藏功能分析。

## 背景

2026 年 3 月 31 日，[Chaofan Shou](https://x.com/shoucccc) 发现 Anthropic 的 Claude Code CLI 将完整源码通过 npm 包中的 source map 文件意外暴露。Source map 本应仅用于开发调试，却被打包进了生产版本，任何人都能据此还原原始 TypeScript 源码（约 1,900 个文件，约 513,000 行代码）。

本仓库 **仅包含我个人的分析与文档**，不包含泄露的源代码。

## 目录

### 分析文档

| 文件 | 说明 |
|------|------|
| [`analysis/tools-breakdown.md`](analysis/tools-breakdown.md) | 工具系统分类 — 39 个工具、10 个类别、权限模型 |
| [`analysis/hidden-features.md`](analysis/hidden-features.md) | 隐藏功能：BUDDY（确定性虚拟宠物）& KAIROS（主动感知助手） |
| [`analysis/extra-modules.md`](analysis/extra-modules.md) | 补充模块分析：coordinator、tasks、memdir、moreright 等 |

### 架构图（Mermaid）

| 文件 | 说明 |
|------|------|
| [`diagrams/architecture-overview.mmd`](diagrams/architecture-overview.mmd) | 高层架构图 — 36 个模块、依赖关系 |
| [`diagrams/query-engine-flow.mmd`](diagrams/query-engine-flow.mmd) | Query Engine 时序图 — Agent 式多轮循环 |
| [`diagrams/ide-bridge.mmd`](diagrams/ide-bridge.mmd) | IDE 桥接协议图 — v1/v2、重连、崩溃恢复 |

### 文章

| 文件 | 说明 |
|------|------|
| [`docs/article-en.md`](docs/article-en.md) | 英文文章（约 1,800 词）— 面向 GitHub / dev.to |
| [`docs/article-zh.md`](docs/article-zh.md) | 中文文章（约 2,300 字）— 面向掘金 / 知乎 |

## 关键发现

- **39 个工具**，多层权限系统（AST 解析命令、动态权限计算、自动分类器）
- **Query Engine** 拥有 7+ 种错误恢复路径、流式工具执行、4 种上下文压缩策略
- **IDE 桥接** 支持双协议（WebSocket/SSE）、崩溃恢复、有界 UUID 去重
- **BUDDY**：基于 Mulberry32 PRNG 的确定性虚拟宠物系统 — 18 个物种、5 个稀有度、1% 闪光概率
- **KAIROS**：主动消息系统，采用追加写入日志和检查点式通信
- **moreright/**：一个 25 行的神秘存根，用永假门控隐藏了 Anthropic 内部功能

## 许可

本分析为原创内容。被分析的源代码归 Anthropic 所有。
