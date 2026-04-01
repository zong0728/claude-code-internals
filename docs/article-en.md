# Inside Claude Code: A Deep Dive Into the Leaked Source

Claude Code is Anthropic's AI-powered coding assistant — a CLI tool that reads your codebase, runs commands, edits files, and manages complex multi-step engineering tasks. In early 2025, its full source code was inadvertently exposed. Here's what's inside.

## How the Leak Happened

Like many modern web applications, Claude Code ships as a minified JavaScript bundle. But the production build included **source maps** — `.map` files that map minified code back to the original TypeScript source. Source maps are standard debugging aids, but they're meant for development only. When included in production, anyone who downloads the bundle can reconstruct the original file tree, complete with comments, variable names, and module structure. That's exactly what happened: the source maps shipped alongside the minified output, and the community extracted the full codebase.

This isn't a novel attack. Source map leaks have exposed the internals of many production apps. The takeaway for engineering teams: always strip source maps from production builds, or serve them only to authenticated debugging sessions.

## Scale & Scope

The extracted codebase is substantial:

| Metric | Value |
|--------|-------|
| Source files (TS/TSX/JS/JSX) | ~1,900 |
| Total lines of code | ~513,000 |
| Top-level modules | 36 |
| Tools (LLM-callable) | 39 |
| React hooks | 85+ |
| UI components | 146+ |
| Slash commands | 100+ |

The architecture spans terminal UI rendering (custom Ink.js fork), LLM orchestration, a full tool system with permission gating, IDE bridge protocols, background task management, persistent memory, plugin/skill extensibility, and multi-agent coordination.

## Architecture Deep-Dive

### The Tool System

Claude Code's 39 tools are the primary interface between the LLM and the outside world. Each tool is built via `buildTool()` (`/Tool.ts`), implementing a standard contract:

- **Input/output schemas** (Zod validation)
- **Permission checks** (`checkPermissions()` → auto-allow / ask user / deny)
- **Metadata flags**: `isReadOnly`, `isDestructive`, `isConcurrencySafe`, `shouldDefer`
- **Rendering methods** for both CLI and IDE display

Tools span 10 categories: file operations (`Read`, `Edit`, `Write`, `Glob`, `Grep`), command execution (`Bash`, `PowerShell`), multi-agent spawning (`Agent`, `SendMessage`, `TeamCreate`), task management (`TaskCreate/Get/List/Update/Stop`), web access (`WebFetch`, `WebSearch`), MCP integration (`MCPTool`, `McpAuth`), mode control (`EnterPlanMode`, `EnterWorktree`, `Config`), and more.

The permission model is notably sophisticated. `Bash` commands are **AST-parsed** to determine if they're read-only (auto-approved) or destructive (requires confirmation). `WebFetch` uses hostname-based rules with a preapproved list. Some tools compute permissions dynamically from input content. A lightweight classifier provides fast auto-approval for safe operations in auto-mode.

### The Query Engine

The heart of Claude Code is `queryLoop()` in `/query.ts` — an async generator that implements the agentic turn loop:

1. **Pre-API processing**: microcompaction, autocompaction, context collapse, tool result budgeting
2. **Streaming LLM call**: `queryModelWithStreaming()` with retry logic, fallback models, and rate limit handling
3. **Streaming tool execution**: `StreamingToolExecutor` runs concurrency-safe tools in parallel while the LLM is still generating
4. **Post-streaming tool execution**: remaining tools run via `toolOrchestration.ts`, partitioned into concurrent (read-only) and serial (write) batches
5. **Error recovery**: prompt-too-long → context collapse drain → reactive compact; max output tokens → escalation to 64K → multi-turn recovery (up to 3 retries)
6. **Stop hooks**: memory extraction, skill discovery, task completion evaluation
7. **Loop**: tool results + attachment messages (memory, skills, commands) fed back; loop continues

The system handles 7+ distinct continuation paths: streaming fallback, collapse drain retry, reactive compact retry, max output token escalation/recovery, stop hook blocking, token budget continuation, and normal next-turn.

### IDE Bridge

The bridge (`/bridge/`) enables bidirectional communication between the CLI and IDE extensions (VS Code, JetBrains). It supports two protocols:

- **v1**: WebSocket inbound + HTTP POST outbound via Session-Ingress
- **v2**: SSE stream inbound + HTTP POST outbound via CCR (Code Compute Runtime)

The server selects the protocol per-session. The bridge handles environment registration, work polling with exponential backoff (2s→120s), message deduplication via bounded UUID sets (capacity: 2000), two reconnection strategies (in-place reuse vs fresh session), heartbeat-based lease extension, flush gating for message ordering, and crash recovery via persistent bridge pointer files.

Control requests flow from the IDE/server to the CLI: `initialize`, `set_model`, `set_permission_mode`, `interrupt` — each with a ~10-14s response deadline before the server kills the connection.

## Hidden Features

### BUDDY: Deterministic Companion System

Hidden in `/buddy/`, this is a virtual pet system. Each user gets a unique ASCII companion deterministically derived from their `userId` via Mulberry32 PRNG seeded with an FNV-1a hash. The system has:

- **18 species** (Duck to Chonk), names hex-encoded to evade bundle string scanning
- **5 rarity tiers**: Common (60%), Uncommon (25%), Rare (10%), Epic (4%), Legendary (1%)
- **1% shiny chance**, independent of rarity
- **5 stats** (Debugging, Patience, Chaos, Wisdom, Snark) with rarity-based floors
- **3-frame ASCII sprite animations** with hats, eyes, speech bubbles, and pet reactions

Only the "soul" (name, personality) is persisted — "bones" (species, rarity, stats) regenerate from userId hash, preventing config editing exploits. The teaser window was April 1-7, 2026.

### KAIROS: Proactive Messaging

KAIROS (`/assistant/` + `/tools/BriefTool/`) fundamentally changes how Claude communicates. All visible output must go through the `SendUserMessage` tool — text outside tool calls goes to a low-visibility detail view. This enables:

- **Proactive status updates** (background task completion, blocker surfacing)
- **Checkpoint-based communication** (ACK → work → result, skip filler)
- **Append-only daily logging** for perpetual assistant sessions, distilled nightly via `/dream`

Multi-layer gating: build-time flags, server-side GrowthBook gates (refreshable every 5 minutes for live kill-switch), and user opt-in.

## Modules Worth Studying

**coordinator/** — Multi-agent orchestration with carefully crafted system prompts. Workers get filtered tool access. The coordinator synthesizes findings rather than just forwarding them.

**memdir/** — Persistent memory with two-level caps (200 lines / 25KB), Sonnet-based relevance selection, path traversal defense, and a clear taxonomy of what should vs shouldn't be memorized.

**moreright/** — A 25-line no-op stub that replaces an internal Anthropic feature in external builds. Gated by `"external" === 'ant'` (always false). The interface (`onBeforeQuery` / `onTurnComplete`) reveals where internal tooling hooks into the main loop.

## Key Takeaways for AI Engineers

1. **Permission design matters deeply.** Claude Code's multi-layer permission system (AST-parsed commands, dynamic computation from input, hostname rules, auto-classifier, plan mode) is arguably more complex than the LLM orchestration itself. If you're building tools that let an LLM execute code, invest heavily here.

2. **Streaming tool execution is a significant optimization.** Running concurrency-safe tools while the LLM is still generating (via `StreamingToolExecutor`) meaningfully reduces latency in multi-tool turns.

3. **Context management is an ongoing battle.** The codebase has four distinct compaction strategies (micro, auto, reactive, context collapse) plus tool result budgeting. Managing the context window is not a solved problem — it's a continuous engineering effort.

4. **Build separation for internal features.** The `moreright/` pattern (always-false string gate + no-op stub) is an elegant way to maintain a single codebase with internal-only features that can never leak behavior to external builds.

5. **Deterministic generation from user IDs.** BUDDY's approach (hash → PRNG seed → attribute rolls) is a clean pattern for generating user-specific content without server calls or persistent state. Only the model-generated parts need storage.

The full analysis, including Mermaid diagrams and detailed module breakdowns, is available in this repository's `/analysis/` and `/diagrams/` directories.
