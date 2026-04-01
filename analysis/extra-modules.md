# Supplementary Module Analysis

## 1. coordinator/ — Multi-Agent Orchestration

**What it does:** Implements coordinator mode where Claude acts as an orchestrator rather than a direct code manipulator. The coordinator spawns autonomous worker agents via the Agent tool, synthesizes their findings, and directs follow-up work. Workers execute tasks in parallel with filtered tool access (no team-management tools).

**Key file:** `coordinatorMode.ts` (19KB)

**Key design:**
- Feature-gated via `feature('COORDINATOR_MODE')` and `CLAUDE_CODE_COORDINATOR_MODE` env var
- System prompt teaches the coordinator to NOT thank workers, not serialize work unnecessarily, and to synthesize findings before directing follow-up
- Workers get filtered tool access — can't use TEAM_CREATE, SEND_MESSAGE, or other internal coordination tools
- Session mode matching: when resuming a session, the env var flips dynamically to maintain consistency
- Clear guidance on **continue vs spawn fresh**: reuse a worker if context overlaps, spawn new if the task is orthogonal
- Dual-phase workflow: read-only tasks parallelized, write-heavy tasks serialized

---

## 2. tasks/ — Background Task Execution Framework

**What it does:** Manages the lifecycle of background tasks including subagents, shell commands, backgrounded main sessions, in-process teammates, and dream (auto-planning) tasks. Tasks have rich state tracking (running → pending → completed/failed) with backgrounding, foregrounding, and mid-flight stopping.

**Key files:**
- `types.ts` — Discriminated union `TaskState` covering all task variants
- `LocalMainSessionTask.ts` (480 lines) — Handles Ctrl+B backgrounding of the active query
- `pillLabel.ts` — Compact footer labels ("1 local agent", "◆ ultraplan ready")
- `stopTask.ts` — Shared kill logic with type-specific handlers

**Key design:**
- **Task isolation via symlinks**: backgrounded sessions write to isolated transcript files so `/clear` doesn't corrupt output
- **Abort controller threading**: reuses the existing controller when backgrounding, so aborting the task aborts the actual query
- **Atomic notification guard**: `notified` flag prevents duplicate completion notifications
- **Grace period** (`PANEL_GRACE_MS = 30s`): terminal tasks stay in state for 30s after eviction, avoiding re-render churn
- **Retention model**: `retain: true` blocks eviction (used when viewing transcript), `evictAfter: timestamp` for delayed cleanup
- Rich progress: `recentActivities` (last 5 tool uses), `tokenCount`, `toolUseCount`

---

## 3. memdir/ — Persistent Memory System

**What it does:** Implements Claude Code's persistent memory as MEMORY.md (index) + topic files in `~/.claude/memory/`. Scans memory directories, parses YAML frontmatter, selects relevant memories via Sonnet, and enforces size limits. Supports both individual and team memory hierarchies with path traversal protection.

**Key files:**
- `memdir.ts` (21KB) — Core API: `truncateEntrypointContent()` enforces 200-line / 25KB caps
- `memoryTypes.ts` (22KB) — Four types: `user` (always private), `feedback` (private/team), `project` (team-biased), `reference` (usually team)
- `paths.ts` (10KB) — Path resolution with `CLAUDE_CODE_REMOTE_MEMORY_DIR` env fallback
- `findRelevantMemories.ts` (5KB) — Sonnet-based selection of top 5 relevant files per query
- `teamMemPaths.ts` (11KB) — Team memory layout with path traversal injection defense
- `memoryScan.ts` (3KB) — Parallel file reading with `Promise.allSettled`, up to 200 .md files

**Key design:**
- **Two-level cap on MEMORY.md**: line count (200) then byte count (25KB), with reason tracking for user-visible warnings
- **Path traversal defense** (`teamMemPaths.ts`): rejects null bytes, URL-encoded traversals, Unicode normalization attacks, backslashes, absolute paths
- **Memory types exclude derivable info**: code patterns, git history, debugging solutions are explicitly excluded — only save non-obvious context
- **Why-based retention**: project/feedback memories include "Why:" sections so future queries can judge if the memory still applies
- **Sonnet-based selection**: `findRelevantMemories()` reads MEMORY.md headers, calls Sonnet to pick top 5 relevant files, avoiding budget waste on recently-surfaced files
- **KAIROS compatibility**: append-only daily logs for perpetual sessions vs live MEMORY.md edits for standard sessions

---

## 4. schemas/ — Hook Type Definitions

**What it does:** Defines Zod schemas for Claude Code's hook system (event-based automation). Extracted from `settings/types.ts` to break circular import dependencies between settings and plugins modules.

**Key file:** `hooks.ts` (223 lines)

**Key design:**
- **Four hook types**: `command` (bash), `prompt` (LLM), `http`, `agent` (agentic verifier)
- **Discriminated union** via `type` field for exhaustive pattern matching
- **Conditional execution**: `if` field uses permission rule syntax (e.g., `"Bash(git *)"`) to filter hooks before spawning
- **Per-hook configuration**: `timeout`, `statusMessage`, `once` (run-once-then-delete), plus type-specific fields
- **Agent hooks** support structured I/O with `$ARGUMENTS` placeholder for injecting hook input JSON
- **Lazy schemas**: `lazySchema()` helper defers Zod parse at initialization time, breaking cyclic dependencies
- No `.transform()` allowed — schemas must persist cleanly to JSON

---

## 5. plugins/ — Built-In Plugin Registry

**What it does:** Manages built-in plugins that ship with the CLI and appear in the `/plugin` UI for user enable/disable toggling. Distinguishes from bundled skills (auto-enabled, no UI toggle) — built-in plugins are explicitly user-controllable.

**Key files:**
- `builtinPlugins.ts` (160 lines) — Registry: `registerBuiltinPlugin()`, `getBuiltinPlugins()`, `getBuiltinPluginSkillCommands()`
- `bundled/index.ts` (23 lines) — Scaffolding for migration; currently empty

**Key design:**
- **Plugin ID format**: `{name}@builtin` vs marketplace `{name}@{marketplace}`
- **Enabled state priority**: user setting > plugin default > true
- **Runtime availability guard**: `isAvailable()` can hide plugins (e.g., platform-specific)
- **Skills sourced as 'bundled'**: keeps them in SkillTool listings, exempt from prompt truncation (unlike 'builtin' which means hardcoded /help, /clear)
- **Composite plugins**: a single plugin can provide skills, hooks, AND MCP servers
- **Map-based registry**: O(1) lookup for built-in plugin ID checks

---

## 6. state/ — Global State Management

**What it does:** Implements immutable global AppState with React context provider and simple pub/sub store. Tracks tool permissions, UI focus, task registry, MCP connections, plugin status, remote sessions, and mode transitions.

**Key files:**
- `AppStateStore.ts` (21KB) — AppState type definition (settings, model, UI, tasks, MCP, permissions, remote)
- `AppState.tsx` (23KB) — React provider wrapping MailboxProvider and VoiceProvider
- `store.ts` (35 lines) — Generic `Store<T>` with getState/setState/subscribe
- `onChangeAppState.ts` (100+ lines) — Single choke point for syncing changes to CCR/SDK
- `selectors.ts` (77 lines) — Pure selectors: `getViewedTeammateTask()`, `getActiveAgentForInput()`

**Key design:**
- **Immutable state** (`DeepImmutable` utility type) except for tasks dict and Maps
- **Single onChange choke point**: ANY `setAppState` that changes mode notifies both CCR (via `reportMetadata`) and SDK (via status stream), closing gaps from scattered callsites
- **Dual notification on mode change**: CCR and SDK get separate notifications, with external mode name wrapping
- **Teammate view transitions**: switching agents releases previous one back to stub form (`retain: false`, `evictAfter` set)
- **Store.subscribe** returns unsubscribe function (Set-based cleanup) for React hook lifecycle
- **Discriminated union selectors**: `ActiveAgentForInput` with `'leader' | 'viewed' | 'named_agent'` for type-safe input routing

---

## 7. moreright/ — Internal Feature Stub (PRIORITY ANALYSIS)

**What it does:** A **no-op stub for external builds** that replaces an internal Anthropic feature. The real implementation exists only in internal builds. The module hooks into the query lifecycle with pre-query and post-query callbacks.

**Key file:** `useMoreRight.tsx` (25 lines) — Comment: "the real hook is internal only"

### The Interface

```typescript
{
  onBeforeQuery: (input: string, allMessages: M[], turnNumber: number) => Promise<boolean>
  onTurnComplete: (allMessages: M[], aborted: boolean) => Promise<void>
  render: () => null  // Always renders nothing in external builds
}
```

### How It Integrates

- Imported in `screens/REPL.tsx` (main UI loop)
- **Always disabled in external builds**: gated by `"external" === 'ant'` (string comparison that always evaluates to `false`)
- Even if `CLAUDE_MORERIGHT` env var is set, the string gate prevents activation
- Called at two lifecycle points:
  - `mrOnBeforeQuery()` — before query execution (can intercept/modify input or messages)
  - `mrOnTurnComplete()` — after query finishes (cleanup or post-processing)
- Can also `setMessages()` (whole array or updater) and `setInputValue()`

### Why "moreright"?

The name is an **internal Anthropic codename** — the codebase uses many such codenames for feature gates:
- `tengu_*` prefix (e.g., `tengu_kairos_brief`, `tengu_herring_clock`, `tengu_surreal_dali`)
- Other codenames: "quail", "thimble", "clock"

Given that it hooks into query lifecycle boundaries (before and after), it likely serves as an **observation/instrumentation layer** — possibly A/B testing infrastructure, validation pipeline, or internal model evaluation tooling. The careful stubbing ensures external builds never execute any internal logic, even accidentally.

### Why It Matters

`moreright/` is architecturally interesting because:
1. It demonstrates Anthropic's **internal/external build separation strategy** — stub files with always-false gates
2. The hook interface (`onBeforeQuery` / `onTurnComplete`) reveals **where internal tooling plugs into the main loop**
3. The "no relative imports" constraint shows how they maintain clean boundaries between internal and external code
