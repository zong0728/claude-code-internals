# Hidden Features Analysis — BUDDY & KAIROS

## 1. BUDDY System — Deterministic Companion Generation

**Location:** `/buddy/` (6 files: `types.ts`, `companion.ts`, `sprites.ts`, `CompanionSprite.tsx`, `useBuddyNotification.tsx`, `prompt.ts`)

### How It Works

BUDDY generates a unique virtual companion for each user, deterministically derived from their `userId`. The same user always gets the same companion — no randomness, no server calls.

**Generation pipeline:**
```
userId + SALT ("friend-2026-401")
  → hashString() (FNV-1a 32-bit hash, or Bun.hash if available)
  → mulberry32(seed) (lightweight 32-bit PRNG)
  → rollFrom(rng) (deterministic attribute selection)
```

### Mulberry32 PRNG (`companion.ts:16-25`)

A fast, non-cryptographic 32-bit seeded PRNG:
- Magic constants: `0x6d2b79f5`, shifts of 15, 7, 14
- Produces floats in [0, 1) via unsigned right-shift and division by 2^32
- Key property: same seed → same sequence, every time

### Species & Rarity

**18 Species:** Duck, Goose, Blob, Cat, Dragon, Octopus, Owl, Penguin, Turtle, Snail, Ghost, Axolotl, Capybara, Cactus, Robot, Rabbit, Mushroom, Chonk

Species names are encoded as hex character codes (e.g., `0x64,0x75,0x63,0x6b` = "duck") to evade string detection in bundle scans.

**5 Rarity Tiers (weighted distribution):**

| Rarity | Weight | Probability | Stars | Stat Floor |
|--------|--------|-------------|-------|------------|
| Common | 60 | 60% | ★ | 5 |
| Uncommon | 25 | 25% | ★★ | 15 |
| Rare | 10 | 10% | ★★★ | 25 |
| Epic | 4 | 4% | ★★★★ | 35 |
| Legendary | 1 | 1% | ★★★★★ | 50 |

**Shiny trait:** 1% chance (`rng() < 0.01`), independent of rarity.

### Companion Attributes ("Bones")

Regenerated from userId hash on every read (not persisted — prevents config editing exploits):

| Attribute | Source | Values |
|-----------|--------|--------|
| `species` | Weighted random | 18 species |
| `rarity` | Weighted random | 5 tiers |
| `eye` | Random pick | `·`, `✦`, `×`, `◉`, `@`, `°` |
| `hat` | Random pick (rarity-gated) | none, crown, tophat, propeller, halo, wizard, beanie, tinyduck |
| `shiny` | 1% chance | boolean |
| `stats` | Roll with rarity floor | 5 stats (0-100) |

**Stat system** — 5 stats: DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK
- One **peak stat**: floor + 50 + rand(0-30), capped at 100
- One **dump stat**: floor - 10 + rand(0-15), min 1
- Three **neutral stats**: floor + rand(0-40)

### Companion Soul (Model-Generated, Persisted)

After bones are rolled, the LLM generates a "soul":
- `name`: String (model picks a fitting name)
- `personality`: String (model writes a personality description)
- `hatchedAt`: Timestamp

Only the soul is persisted to `config.companion` — bones are always regenerated from userId, making it impossible to fake a legendary companion by editing config.

### Visual Rendering (`sprites.ts`, `CompanionSprite.tsx`)

- **ASCII art sprites**: 3 animation frames per species, 5 lines tall × 12 chars wide
- `{E}` placeholder substituted with the companion's eye character
- Hats rendered as single-line overlays on frame 0
- **Animation**: 500ms tick interval, idle sequence `[0,0,0,0,1,0,0,0,-1,0,0,2,0,0,0]`
- **Speech bubbles**: 10s display (20 ticks), fade in last 3s
- **Pet reaction**: 2.5s heart burst animation with 5 floating heart frames
- **Responsive**: Full sprite at 100+ columns, collapses to single-line face in narrow terminals

### Teaser / Rollout (`useBuddyNotification.tsx`)

- Rainbow `/buddy` notification on startup when no companion hatched
- **Teaser window**: April 1-7, 2026 only (local timezone, 24h rolling wave across timezones)
- `/buddy` command stays available permanently after the teaser period
- Feature-gated: `feature('BUDDY')`

---

## 2. KAIROS — Proactive Awareness Assistant

**Location:** Dynamic import from `/assistant/` + `/tools/BriefTool/`

### Architecture

KAIROS is Claude Code's **proactive messaging system** — it allows the assistant to send unsolicited updates to the user (background task completion, blockers discovered, status updates).

**Dynamic loading** (`main.tsx:80-81`):
```javascript
const assistantModule = feature('KAIROS') ? require('./assistant/index.js') : null
```
Conditional imports enable dead-code elimination in external builds.

### The SendUserMessage Tool (`BriefTool.ts`)

The core mechanism — a tool that lets Claude send visible messages to the user:

| Parameter | Type | Description |
|-----------|------|-------------|
| `message` | string | Markdown content |
| `attachments` | string[] | Optional file paths |
| `status` | enum | `'normal'` (reply) or `'proactive'` (unsolicited) |

**Key distinction**: Text outside tool calls goes to a detail view (low visibility). Only `SendUserMessage` content appears in the main chat view. This forces the model to explicitly route important content through the visible channel.

### Entitlement & Activation Gates

**Multi-layer gating:**

1. **Build-time feature flags**: `KAIROS`, `KAIROS_BRIEF`, `PROACTIVE`, `KAIROS_CHANNELS`, `KAIROS_GITHUB_WEBHOOKS`
2. **Server-side entitlement**: GrowthBook gate `tengu_kairos_brief` (refreshed every 5 minutes — can kill-switch mid-session)
3. **User opt-in**: `--brief` CLI flag, `defaultView: 'chat'` setting, or `/brief` slash command
4. **Dev bypass**: `CLAUDE_CODE_BRIEF=1` environment variable

### Proactive Trigger Conditions

From the system prompt (`prompt.ts:12-22`):

1. **Every message gets visible output** via SendUserMessage — text outside tool calls is detail-view only
2. **Checkpoint pattern**:
   - ACK first (one-liner if running work): "On it — checking output"
   - Run work (read files, execute commands)
   - Send result via SendUserMessage
   - Skip filler updates — checkpoint only at decision/surprise/phase boundaries
3. **Proactive status** for background work the user didn't directly ask for

### Append-Only Logging Infrastructure

KAIROS sessions use a different memory paradigm than standard sessions:

| Aspect | Standard Session | KAIROS Session |
|--------|-----------------|----------------|
| Memory writes | Live edits to MEMORY.md | Append-only daily logs |
| Log format | Structured frontmatter files | Date-named daily files (YYYY-MM-DD) |
| Distillation | Manual | Nightly `/dream` skill distills logs → MEMORY.md + topic files |
| Team sync | Compatible | Incompatible (needs shared live MEMORY.md) |

**Session history** (`assistant/sessionHistory.ts`):
- Fetches user message events from session API
- Pagination: `HISTORY_PAGE_SIZE = 100`
- API endpoint: `/v1/sessions/{sessionId}/events`

**Transcript format**: JSONL, append-only. Rewinding (ctrl-z) leaves orphaned branches; walk back via `parentUuid`. Pre-filter excises dead forks before parsing.

### Feature Flag Taxonomy

| Flag | Scope | What It Enables |
|------|-------|-----------------|
| `KAIROS` | Full assistant mode | All KAIROS features + implies KAIROS_BRIEF |
| `KAIROS_BRIEF` | Brief tool only | SendUserMessage without full assistant mode |
| `PROACTIVE` | Proactive base | Unified with KAIROS for proactive messaging |
| `KAIROS_CHANNELS` | Channel routing | Channel-based message routing |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub events | GitHub webhook integration |

---

## Key Takeaways

1. **BUDDY is pure fun** — a deterministic gacha system that gives every user a unique ASCII companion. The use of Mulberry32 PRNG ensures reproducibility without server calls, and persisting only the "soul" (not "bones") prevents cheating while allowing safe species/rarity rebalancing.

2. **KAIROS is architectural** — it fundamentally changes how Claude communicates by routing all visible output through an explicit tool call. This enables proactive messaging (status: 'proactive'), checkpoint-based communication, and append-only logging for perpetual assistant sessions.

3. **Both are heavily feature-gated** — external builds never see either feature. BUDDY uses build-time flags; KAIROS uses both build-time flags and server-side GrowthBook gates with live kill-switch capability.
