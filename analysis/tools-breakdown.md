# Claude Code Tool System — Classification & Permission Model

## Overview

Claude Code exposes **~40 tools** through a unified `buildTool()` pattern defined in `/Tool.ts`. Each tool implements a standard interface with:
- **Input/Output schemas** (Zod-based validation)
- **Permission checks** (`checkPermissions()` — auto-allow, user-ask, or deny)
- **Metadata flags**: `isReadOnly`, `isDestructive`, `isConcurrencySafe`, `shouldDefer`
- **Rendering methods** for CLI and IDE display

Tools are registered in `/tools.ts` and filtered at runtime by deny rules, feature gates, and platform constraints.

---

## Permission Model

| Level | Description | Examples |
|-------|-------------|---------|
| **Auto-allow** | No user prompt needed; safe read-only ops | `Read`, `Glob`, `Grep`, `WebSearch`, `TaskList` |
| **User approval required** | Prompt shown before execution | `Bash`, `Edit`, `Write`, `WebFetch` |
| **Rule-based** | Allow/deny per hostname, path, or command pattern | `WebFetch` (preapproved hosts), `Bash` (AST-parsed commands) |
| **Feature-gated** | Requires server-side entitlement or feature flag | `CronCreate`, `TeamCreate`, `RemoteTrigger`, `BriefTool` |
| **Computed dynamically** | Permission depends on input content | `Bash` (read-only commands auto-allowed), `Config` (get vs set) |

### Key permission infrastructure:
- **`ToolPermissionContext`** — carries mode (plan/auto/normal), deny/allow/ask rules, working directories
- **Auto-classifier** — in auto-mode, a lightweight classifier fast-approves safe operations
- **Plan mode** — all tool use requires explicit approval; entered via `EnterPlanMode`
- **Deny rules** — blanket deny rules filter tools before the model even sees them (`filterToolsByDenyRules`)

---

## Tool Classification Table

### File Operations

| # | Tool Name | Registered As | Permission | Read-Only | Destructive | Key Parameters | Description |
|---|-----------|---------------|------------|-----------|-------------|----------------|-------------|
| 1 | FileReadTool | `Read` | Read permission | Yes | No | `file_path`, `limit`, `offset`, `pages` | Read files (text, images, PDFs, notebooks). 100KB text limit, 20 PDF page limit. Blocks `/dev/zero` etc. |
| 2 | FileEditTool | `Edit` | Write permission | No | Yes | `file_path`, `old_string`/`new_string` or `new_contents` | String-based or line-based edits. UTF-8 BOM preservation. 1 GiB max. |
| 3 | FileWriteTool | `Write` | Write permission | No | Yes | `file_path`, `content` | Create or overwrite entire files. Auto-creates parent dirs. |
| 4 | NotebookEditTool | `NotebookEdit` | Write permission | No | Yes | `notebook_path`, `cell_id`, `new_source`, `edit_mode` | Edit Jupyter notebook cells (replace/insert/delete). |

### Search & Discovery

| # | Tool Name | Registered As | Permission | Read-Only | Destructive | Key Parameters | Description |
|---|-----------|---------------|------------|-----------|-------------|----------------|-------------|
| 5 | GlobTool | `Glob` | Read permission | Yes | No | `pattern`, `path` | Fast file pattern matching. Returns up to 100 results. |
| 6 | GrepTool | `Grep` | Read permission | Yes | No | `pattern`, `path`, `glob`, `output_mode`, `-A/-B/-C`, `-i`, `multiline` | Full ripgrep integration. Content/files/count modes. Default 250 result limit. |
| 7 | LSPTool | `LSP` | Read permission | Yes | No | `operation`, `filePath`, `line`, `character` | Code intelligence: go-to-definition, find-references, hover, symbols, call hierarchy. |
| 8 | ToolSearchTool | `ToolSearch` | Auto-allow | Yes | No | `query`, `max_results` | Discovers deferred tools by keyword or `select:<name>`. |

### Command Execution

| # | Tool Name | Registered As | Permission | Read-Only | Destructive | Key Parameters | Description |
|---|-----------|---------------|------------|-----------|-------------|----------------|-------------|
| 9 | BashTool | `Bash` | User approval (AST-checked) | Computed | Computed | `command`, `timeout`, `run_in_background` | Shell execution with security AST parsing. Read-only commands auto-allowed. Destructive ops flagged. |
| 10 | PowerShellTool | `PowerShell` | User approval | Computed | Computed | `command`, `timeout`, `run_in_background` | Windows PowerShell execution. Same security model as Bash. |

### Multi-Agent & Task Management

| # | Tool Name | Registered As | Permission | Read-Only | Destructive | Key Parameters | Description |
|---|-----------|---------------|------------|-----------|-------------|----------------|-------------|
| 11 | AgentTool | `Agent` | Conditional | No | No | `prompt`, `description`, `subagent_type`, `model`, `run_in_background`, `isolation` | Spawn sub-agents with optional worktree/remote isolation. Background task support. |
| 12 | SendMessageTool | `SendMessage` | User approval (cross-machine) | No | No | `to`, `message`, `summary` | Send messages to teammates, broadcast, or protocol messages (shutdown, plan approval). |
| 13 | TeamCreateTool | `TeamCreate` | Feature-gated | No | No | `team_name`, `description`, `agent_type` | Create a new agent team/swarm. |
| 14 | TeamDeleteTool | `TeamDelete` | Feature-gated | No | No | _(none)_ | Clean up team resources. Validates all members idle. |
| 15 | TaskCreateTool | `TaskCreate` | Auto-allow | No | No | `subject`, `description`, `activeForm`, `metadata` | Create task in V2 task system. Fires TaskCreated hooks. |
| 16 | TaskGetTool | `TaskGet` | Auto-allow | Yes | No | `taskId` | Retrieve task by ID with dependency info. |
| 17 | TaskListTool | `TaskList` | Auto-allow | Yes | No | _(none)_ | List all non-internal tasks with status. |
| 18 | TaskUpdateTool | `TaskUpdate` | Auto-allow | No | No | `taskId`, `status`, `subject`, `description`, `addBlocks`, `addBlockedBy`, `owner` | Update task status/metadata. Fires TaskCompleted hooks. |
| 19 | TaskStopTool | `TaskStop` | Auto-allow | No | No | `task_id` | Stop a running background task. Aliases: `AgentOutputTool`, `BashOutputTool`. |
| 20 | TaskOutputTool | `TaskOutput` | Auto-allow | Yes | No | `task_id`, `block`, `timeout` | Get output from background task. Deprecated — prefer `Read` on output file. |
| 21 | TodoWriteTool | `TodoWrite` | Auto-allow | No | No | `todos` | Legacy V1 todo system. Disabled when V2 tasks enabled. |

### Mode & Environment Control

| # | Tool Name | Registered As | Permission | Read-Only | Destructive | Key Parameters | Description |
|---|-----------|---------------|------------|-----------|-------------|----------------|-------------|
| 22 | EnterPlanModeTool | `EnterPlanMode` | Auto-allow | Yes | No | _(none)_ | Switch to plan mode (all tools require approval). Disabled with `--channels`. |
| 23 | ExitPlanModeTool | `ExitPlanMode` | Prompt-based | No | No | `allowedPrompts` | Exit plan mode, persist plan to disk, grant prompt-based permissions. |
| 24 | EnterWorktreeTool | `EnterWorktree` | Auto-allow | No | No | `name` | Create isolated git worktree. Validates slug format. |
| 25 | ExitWorktreeTool | `ExitWorktree` | Auto-allow | No | Conditional | `action` (keep/remove), `discard_changes` | Exit worktree. Blocks removal if changes exist without explicit `discard_changes:true`. |
| 26 | ConfigTool | `Config` | Read: auto; Set: approval | Computed | No | `setting`, `value` | Get/set Claude Code settings (theme, model, permissions, etc.). |

### Web & External

| # | Tool Name | Registered As | Permission | Read-Only | Destructive | Key Parameters | Description |
|---|-----------|---------------|------------|-----------|-------------|----------------|-------------|
| 27 | WebFetchTool | `WebFetch` | Hostname-based rules | No | No | `url`, `prompt` | Fetch web pages, convert to markdown, apply extraction prompt. Preapproved hosts list. 100K char max. |
| 28 | WebSearchTool | `WebSearch` | Auto-allow | Yes | No | `query`, `allowed_domains`, `blocked_domains` | Web search via Anthropic's search tool. Max 8 searches per call. |

### MCP (Model Context Protocol)

| # | Tool Name | Registered As | Permission | Read-Only | Destructive | Key Parameters | Description |
|---|-----------|---------------|------------|-----------|-------------|----------------|-------------|
| 29 | MCPTool | `mcp__<server>__<tool>` | Passthrough (MCP server decides) | Varies | Varies | _(dynamic, per MCP server)_ | Generic wrapper for MCP server tools. 100K char result max. |
| 30 | McpAuthTool | `mcp__<server>__authenticate` | Auto-allow | No | No | _(none)_ | Start OAuth flow for unauthenticated MCP servers. |
| 31 | ListMcpResourcesTool | `ListMcpResources` | Auto-allow | Yes | No | `server` (optional) | List available MCP resources. LRU-cached. |
| 32 | ReadMcpResourceTool | `ReadMcpResource` | Auto-allow | Yes | No | `server`, `uri` | Read a specific MCP resource by URI. |

### Communication & Output

| # | Tool Name | Registered As | Permission | Read-Only | Destructive | Key Parameters | Description |
|---|-----------|---------------|------------|-----------|-------------|----------------|-------------|
| 33 | AskUserQuestionTool | `AskUserQuestion` | Auto-allow | Yes | No | `questions` (1-4 items, each with `question`, `options`, `multiSelect`) | Multi-choice questions to user. Supports previews and annotations. |
| 34 | BriefTool | `SendUserMessage` | Auto-allow (feature-gated) | Yes | No | `message`, `attachments`, `status` | Send proactive messages. Requires KAIROS entitlement. |
| 35 | SyntheticOutputTool | `StructuredOutput` | Auto-allow | Yes | No | _(dynamic, per schema)_ | Return structured JSON response validated against provided schema. SDK/CLI use. |

### Scheduling & Automation

| # | Tool Name | Registered As | Permission | Read-Only | Destructive | Key Parameters | Description |
|---|-----------|---------------|------------|-----------|-------------|----------------|-------------|
| 36 | ScheduleCronTool | `CronCreate` | Feature-gated | No | No | `cron`, `prompt`, `recurring`, `durable` | Schedule prompts on cron schedule. Durable mode persists across restarts. |
| 37 | RemoteTriggerTool | `RemoteTrigger` | OAuth + feature-gated | No | No | `action` (list/get/create/update/run), `trigger_id`, `body` | Manage scheduled remote agents via REST API. |

### Utility

| # | Tool Name | Registered As | Permission | Read-Only | Destructive | Key Parameters | Description |
|---|-----------|---------------|------------|-----------|-------------|----------------|-------------|
| 38 | SleepTool | `Sleep` | Auto-allow | Yes | No | _(duration)_ | Wait for specified duration. User-interruptible. |
| 39 | SkillTool | `Skill` | Varies by skill | Varies | Varies | _(dynamic, per skill)_ | Invoke slash-command skills (/commit, /review-pr, etc.). |

---

## Permission Flow Diagram

```
User Input → Model selects tool
  ↓
filterToolsByDenyRules() — remove blanket-denied tools
  ↓
tool.isEnabled() — check feature gates, platform
  ↓
tool.checkPermissions(input) — tool-specific logic
  ├── AUTO_ALLOW → execute immediately
  ├── ASK → prompt user for approval
  │     ├── approved → execute
  │     └── denied → accumulate denial, skip
  └── DENY → skip tool
  ↓
tool.call(input) → result
  ↓
Result size check (> maxResultSizeChars → persist to disk)
```

---

## Key Design Patterns

1. **Fail-closed defaults**: `buildTool()` defaults to `isReadOnly: false`, `isConcurrencySafe: false` — tools must opt-in to less restrictive behavior.
2. **Dynamic permission computation**: `Bash` and `Config` compute permission level from input content (read-only commands vs destructive ops).
3. **Deferred loading**: Tools with `shouldDefer: true` aren't sent to the model until discovered via `ToolSearch`, reducing prompt size.
4. **Feature gates**: Server-side entitlements control access to advanced features (KAIROS, agent swarms, remote triggers, cron).
5. **Sandbox enforcement**: `Bash` and `PowerShell` support sandboxing with opt-out via `dangerouslyDisableSandbox`.
6. **File history tracking**: `Edit`, `Write`, and `NotebookEdit` track file modification history for undo/audit.
7. **MCP passthrough**: MCP tools are dynamically registered wrappers — permission model is delegated to the MCP server.

---

## Source Files

| File | Purpose |
|------|---------|
| `/Tool.ts` | Base `Tool` interface, `ToolDef` type, `buildTool()` factory, `ToolPermissionContext` |
| `/tools.ts` | Tool registry, `filterToolsByDenyRules()`, tool list assembly |
| `/tools/shared/` | Shared utilities: `spawnMultiAgent.ts`, `gitOperationTracking.ts` |
| `/tools/utils.ts` | Tool-level utilities |
| `/tools/<ToolName>/` | Individual tool implementations (one directory per tool) |
