# Claude Code Internals

Deep dive into Claude Code's architecture based on the [leaked source code](https://github.com/anthropics/claude-code) — tools system, query engine, IDE bridge, hidden features & more.

基于泄露源码的 Claude Code 架构深度拆解 — 工具系统、Query Engine、IDE 桥接、隐藏功能分析。

## Background / 背景

On March 31, 2026, [Chaofan Shou discovered](https://x.com/shoucccc) that Anthropic's Claude Code CLI had its full source code exposed via source maps bundled in the npm package. The source maps — normally a development-only debugging aid — were accidentally included in the production build, allowing anyone to reconstruct the original TypeScript source (~1,900 files, ~513,000 lines of code).

This repo contains **only my own analysis and documentation** — no leaked source code is included.

## Contents / 目录

### Analysis / 分析文档

| File | Description |
|------|-------------|
| [`analysis/tools-breakdown.md`](analysis/tools-breakdown.md) | Tool system classification — 39 tools, 10 categories, permission model |
| [`analysis/hidden-features.md`](analysis/hidden-features.md) | Hidden features: BUDDY (deterministic companion) & KAIROS (proactive messaging) |
| [`analysis/extra-modules.md`](analysis/extra-modules.md) | Supplementary modules: coordinator, tasks, memdir, moreright, etc. |

### Diagrams / 架构图 (Mermaid)

| File | Description |
|------|-------------|
| [`diagrams/architecture-overview.mmd`](diagrams/architecture-overview.mmd) | High-level architecture — 36 modules, dependency graph |
| [`diagrams/query-engine-flow.mmd`](diagrams/query-engine-flow.mmd) | Query engine sequence diagram — agentic turn loop |
| [`diagrams/ide-bridge.mmd`](diagrams/ide-bridge.mmd) | IDE bridge protocol — v1/v2, reconnection, crash recovery |

### Articles / 文章

| File | Description |
|------|-------------|
| [`docs/article-en.md`](docs/article-en.md) | English article (~1,800 words) — for GitHub / dev.to |
| [`docs/article-zh.md`](docs/article-zh.md) | 中文文章（~2,300 字）— 面向掘金 / 知乎读者 |

## Key Findings / 关键发现

- **39 tools** with a multi-layer permission system (AST-parsed commands, dynamic permission computation, auto-classifier)
- **Query engine** with 7+ error recovery paths, streaming tool execution, and 4 context compaction strategies
- **IDE bridge** supporting dual protocols (WebSocket/SSE), crash recovery, and bounded UUID deduplication
- **BUDDY**: A deterministic virtual companion system using Mulberry32 PRNG — 18 species, 5 rarity tiers, 1% shiny chance
- **KAIROS**: A proactive messaging system with append-only logging and checkpoint-based communication
- **moreright/**: A mysterious 25-line stub hiding an internal-only Anthropic feature behind an always-false gate

## License

This analysis is original work. The analyzed source code belongs to Anthropic.
