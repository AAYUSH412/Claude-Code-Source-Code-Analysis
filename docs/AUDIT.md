# 🔍 Claude Code — Source Code Audit

> A comprehensive technical audit of Claude Code's TypeScript source covering models, architecture, intelligence, security, extensibility, and infrastructure.

> All findings backed by source file references from the `@anthropic-ai/claude-code` npm package.

---

## Table of Contents

1. [Models & Pricing](#1-models--pricing)
2. [Architecture](#2-architecture)
3. [Intelligence](#3-intelligence)
4. [Security](#4-security)
5. [Extensibility](#5-extensibility)
6. [Infrastructure](#6-infrastructure)

---

## 1. Models & Pricing

### Complete Model Registry

Every model ID, label, and provider string hardcoded in the binary:

**Source:** `src/utils/model/configs.ts:9-99`

| Family     | Versions                | Model IDs                          |
| ---------- | ----------------------- | ---------------------------------- |
| **Opus**   | 4.6, 4.5, 4.1, 4.0      | `claude-opus-4-6-20260XXX`, etc.   |
| **Sonnet** | 4.6, 4.5, 4.0, 3.7, 3.5 | `claude-sonnet-4-6-20260XXX`, etc. |
| **Haiku**  | 4.5, 3.5                | `claude-haiku-4-5-20251001`, etc.  |

**Provider strings:** `firstParty`, `bedrock`, `vertex`, `foundry`

**Model aliases:**

| Alias      | Maps to                        |
| ---------- | ------------------------------ |
| `opus`     | Latest Opus                    |
| `sonnet`   | Latest Sonnet                  |
| `haiku`    | Latest Haiku                   |
| `best`     | Opus 4.6                       |
| `opusplan` | Opus in plan mode, else Sonnet |

**Fallback:** `--fallback-model` flag for automatic fallback on overloaded primaries

---

### Fast Mode Internals

**Source:** `src/utils/fastMode.ts:38-147`

| Property             | Value                                       |
| -------------------- | ------------------------------------------- |
| Supported model      | Opus 4.6 only                               |
| Beta header          | `fast-mode-2026-02-01`                      |
| Enabled by default   | Yes (for paid users)                        |
| Disable              | `CLAUDE_CODE_DISABLE_FAST_MODE=1`           |
| Provider restriction | 1P only (not Bedrock/Vertex/Foundry)        |
| Requires             | `extra-usage` billing capability            |
| Free accounts        | Excluded                                    |
| Rate limit           | Tracks cooldown with reset timestamps       |
| Network bypass       | `CLAUDE_CODE_SKIP_FAST_MODE_NETWORK_ERRORS` |
| **Pricing**          | **$30/$150 per MTok (input/output)**        |
| Normal pricing       | $5/$25 per MTok                             |
| **Markup**           | **6× premium**                              |

Internal codename: **Penguin Mode** (`penguinModeOrgEnabled` in config)

---

### Context Window & Token Architecture

**Source:** `src/utils/context.ts:8-25, 71, 86, 89`

| Model      | Default Context | Max Context     | Default Output | Max Output |
| ---------- | --------------- | --------------- | -------------- | ---------- |
| Opus 4.6   | 200K            | 1M (via `[1m]`) | 64K            | 128K       |
| Sonnet 4.6 | 200K            | 1M (via `[1m]`) | 32K            | 128K       |
| Others     | 200K            | 200K            | 32K            | 64K        |

- 1M enabled via beta header `context-1m-2025-08-07`
- HIPAA disable: `CLAUDE_CODE_DISABLE_1M_CONTEXT`
- Constants: `MAX_OUTPUT_TOKENS_DEFAULT`, `CAPPED_DEFAULT_MAX_TOKENS`, `ESCALATED_MAX_TOKENS`

---

### Model Selection & Routing Logic

**Source:** `src/utils/model/model.ts:61-98`

Five-layer priority cascade:

```
Priority 1: /model session override        (highest)
Priority 2: --model CLI flag
Priority 3: ANTHROPIC_MODEL env var
Priority 4: settings file
Priority 5: Built-in default               (lowest)
```

**Defaults by user tier:**

| User Type                  | Default Model |
| -------------------------- | ------------- |
| Max/Team subscribers       | Opus 4.6      |
| Standard users             | Sonnet 4.6    |
| Anthropic employees (Ants) | Opus 4.6 [1m] |

**Override env vars:**

- `ANTHROPIC_DEFAULT_OPUS_MODEL`
- `ANTHROPIC_DEFAULT_SONNET_MODEL`
- `ANTHROPIC_SMALL_FAST_MODEL` — for internal lightweight queries

---

### Internal Codename Map

**Source:** `src/utils/model/antModels.ts:4-42`

| Codename          | Meaning                      | Notes                                                    |
| ----------------- | ---------------------------- | -------------------------------------------------------- |
| **Capybara**      | Active internal model        | Encoded as `charCodes` to evade build-time string filter |
| **Fennec**        | Retired Opus codename        | Migrated via `migrateFennecToOpus.ts`                    |
| **Turtle Carbon** | UltraThink feature gate      | GrowthBook: `tengu_turtle_carbon`                        |
| **Pewter Ledger** | Plan file structure A/B test | GrowthBook: `tengu_pewter_ledger`                        |
| **Birch Mist**    | First-message optimization   | Experiment codename                                      |

- `scripts/excluded-strings.txt` blocklist strips codenames from external builds
- Ant model override: `tengu_ant_model_override` with `alwaysOnThinking`, custom `contextWindow`

---

## 2. Architecture

### Tool System Architecture

**Source:** `src/Tool.ts` (793 lines)

Every tool implements an extensive interface:

```typescript
{
  call(); // Execute the tool
  description(); // Human-readable description
  prompt(); // LLM-facing prompt text
  inputSchema; // Zod input validation
  outputSchema; // Zod output validation (optional)
  checkPermissions(); // Permission gating
  isConcurrencySafe(); // Default: false
  isReadOnly(); // Default: false
  isDestructive(); // Default: false
  shouldDefer; // Lazy loading control
  alwaysLoad; // Force load control
  searchHint; // 3-10 word capability phrase for ToolSearch
  buildTool(); // Factory merges defaults with overrides
}
```

---

### Complete Tool Inventory

**Source:** `src/tools.ts:1-150`, `src/tools/` (46 directories)

| Category             | Tools                                                                                                              |
| -------------------- | ------------------------------------------------------------------------------------------------------------------ |
| **Core (always on)** | Agent, Bash, FileRead, FileEdit, FileWrite, Glob, Grep, WebFetch, WebSearch, NotebookEdit, TodoWrite, Skill, Brief |
| **Task tools**       | TaskCreate, TaskGet, TaskUpdate, TaskList, TaskOutput, TaskStop, EnterPlanMode, ExitPlanModeV2                     |
| **MCP**              | MCPTool wrapper, ListMcpResources, ReadMcpResource, McpAuth                                                        |
| **Multi-agent**      | TeamCreate, TeamDelete, SendMessage, AskUserQuestion                                                               |
| **Feature-gated**    | Sleep, Cron tools, Monitor, WebBrowser, PushNotification, SubscribePR, Workflow, Snip, TerminalCapture             |
| **ANT-only**         | ConfigTool, REPLTool, TungstenTool (virtual terminal), SuggestBackgroundPR                                         |

---

### CLAUDE.md Loading Hierarchy

**Source:** `src/utils/claudemd.ts:1-26`

```
Priority (lowest → highest):
├── /etc/claude-code/CLAUDE.md        System-wide defaults
├── ~/.claude/CLAUDE.md               User-level config
├── project/CLAUDE.md                 Project root
│   project/.claude/CLAUDE.md         Alternative location
│   project/.claude/rules/*.md        Rule files
└── CLAUDE.local.md                   Local overrides (highest)
```

**Features:**

- `@include` directive: `@path`, `@./relative`, `@~/home`, `@/absolute`
- Frontmatter `paths:` for glob-based applicability filtering
- HTML comment stripping for `<!-- ... -->` blocks
- Circular reference prevention via content tracking
- `contentDiffersFromDisk` flag with `rawContent` preserved
- `MEMORY.md` truncated by line/byte caps

---

## 3. Intelligence

### UltraThink Enhanced Reasoning

**Source:** `src/utils/thinking.ts:10-162`

| Property          | Details                                                                    |
| ----------------- | -------------------------------------------------------------------------- |
| Modes             | `adaptive` (model decides), `enabled` (fixed budget), `disabled`           |
| Supported models  | Opus 4.6 and Sonnet 4.6 (adaptive), Opus 4+ / Sonnet 4+ (basic)            |
| Trigger detection | `hasUltrathinkKeyword()` — regex `/\bultrathink\b/i`                       |
| Visual            | 7-color rainbow shimmer (red, orange, yellow, green, blue, indigo, violet) |
| Dual-gated        | Compile-time `feature('ULTRATHINK')` + runtime `tengu_turtle_carbon`       |
| 3P limitation     | Bedrock/Vertex limited to basic thinking for Opus 4+ / Sonnet 4+ only      |

**Default enabled:** Unless explicitly disabled via `settings.alwaysThinkingEnabled = false` or `MAX_THINKING_TOKENS=0`

---

### AutoCompact & Context Management

**Source:** `src/services/compact/autoCompact.ts:28-91`

| Parameter                   | Value                                      |
| --------------------------- | ------------------------------------------ |
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000                                     |
| Trigger point               | ~187K of 200K context                      |
| Circuit breaker             | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` |
| MicroCompact targets        | Bash, FileRead, Grep output only           |
| Post-compact restore        | Up to 5 files (5K tokens each, 25K total)  |
| Warning threshold           | 20,000 buffer tokens                       |
| Manual compact buffer       | 3,000 tokens                               |
| Background extraction       | Key insights persisted to memory files     |

---

### Prompt Cache Optimization

**Source:** `src/services/api/promptCacheBreakDetection.ts:19-99`

A significant competitive differentiator — granular two-phase cache break detection:

**Pre-call:** `recordPromptState()` captures:

- System prompt text
- Tool schemas (per-tool hash)
- Model, betas, effort
- Cache scope/TTL

**Post-call:** `checkResponseForCacheBreak()` compares:

- `cache_read` tokens vs expected
- System prompt char delta
- Per-tool schema hash changes
- Scope/TTL flips
- Effort value changes
- Beta header adds/removals
- Auto-mode toggles

**Heuristics:**

- `>5min` gap → suggests TTL expiry
- `<5min` gap → suggests server issue
- `MIN_CACHE_MISS_TOKENS = 2K` → avoid false positives
- Handles intentional `cache_edits` deletions
- `MAX_TRACKED_SOURCES = 10` per agent
- Debug: writes unified diffs to temp dir for Ant debugging

---

### Scratchpad System

**Source:** `src/utils/permissions/filesystem.ts`

| Property       | Value                                                       |
| -------------- | ----------------------------------------------------------- |
| Path           | `/tmp/claude-{uid}/{sanitized-cwd}/{sessionId}/scratchpad/` |
| Permissions    | `0o700` (owner-only)                                        |
| Path traversal | Prevention via normalization                                |
| Scope          | Session-scoped with automatic cleanup                       |
| Gate           | `tengu_scratch`                                             |
| Usage          | Cross-worker knowledge exchange in coordinator mode         |

---

### Plan Mode V2

**Source:** `src/utils/planModeV2.ts:5-60`

5-phase interview workflow:

```
Interview → Implementation Planning → Execution → Verification → Review
```

| Property             | Value                                                                |
| -------------------- | -------------------------------------------------------------------- |
| Exploration agents   | 3 parallel (Max/Enterprise), 1 (default)                             |
| Agent count override | `CLAUDE_CODE_PLAN_V2_AGENT_COUNT` env var                            |
| Experiment           | `tengu_pewter_ledger` — plan size variants (trim, cut, cap)          |
| Access               | Ants always-on; external gated via `tengu_plan_mode_interview_phase` |

---

## 4. Security

### Permission System Deep Dive

**Source:** `src/types/permissions.ts:16-38`

| Mode                | Behavior              |
| ------------------- | --------------------- |
| `default`           | Interactive prompting |
| `plan`              | Explicit approval     |
| `bypassPermissions` | Full bypass           |
| `dontAsk`           | Auto-deny             |
| `acceptEdits`       | Auto-accept edits     |
| `auto`              | ML classifier decides |

**Rule sources** (8, in priority order):

1. `userSettings`
2. `projectSettings`
3. `localSettings`
4. `flagSettings`
5. `policySettings`
6. `cliArg`
7. `command`
8. `session`

**Bash security:**

- Two-stage classifier: fast static (AST) + thinking (Claude evaluates)
- Risk levels: `LOW` / `MEDIUM` / `HIGH`
- Sleep blocking: `sleep N` where N ≥ 2s blocked; sub-2s allowed
- YOLO classifier: `classifyYoloAction()` — Claude evaluates its own tool use safety

---

### Upstream Proxy Security (CCR)

**Source:** `src/upstreamproxy/upstreamproxy.ts:7-79`

Serious security engineering for containerized environments:

```
1. Read session token from /run/ccr/session_token
2. Set prctl(PR_SET_DUMPABLE, 0) → block ptrace from same-UID processes
3. Download CCR CA certificate → concatenate with system bundle
4. Start local CONNECT-to-WebSocket relay → inject auth into proxied requests
5. Unlink token file → token exists only in heap memory after this point
```

- `NO_PROXY` exclusions: Anthropic, GitHub, package registries, IMDS
- Sets `HTTPS_PROXY` and `SSL_CERT_FILE` for subprocess inheritance

---

### Prompt Injection Defenses (6 Layers)

**Source:** `src/constants/prompts.ts`, `src/utils/deepLink/parseDeepLink.ts`

| Layer | Defense                                                               |
| ----- | --------------------------------------------------------------------- |
| 1     | System prompt instructs model to flag suspected injection             |
| 2     | `partiallySanitizeUnicode()` strips hidden chars in deep links        |
| 3     | Subprocess env protection prevents secret exfiltration                |
| 4     | `toolResultStorage.ts` budgets prevent overflow attacks               |
| 5     | Trusted (hooks) vs untrusted (tool results) separation                |
| 6     | ALL hooks require workspace trust dialog; SDK mode has implicit trust |

**Deep link hardening:**

- ASCII control chars (0x00-0x1F, 0x7F) rejected outright
- Path validation: absolute paths only, ≤ 4096 chars
- Query length: max 5000 chars (Windows cmd.exe safety margin)
- Long prefills get explicit scroll warning above `LONG_PREFILL_THRESHOLD`

---

## 5. Extensibility

### MCP Integration

**Source:** `src/services/mcp/` (24 files)

| Property              | Details                                                          |
| --------------------- | ---------------------------------------------------------------- |
| **Transports**        | stdio, SSE, HTTP, WebSocket, SDK, CloudAI proxy                  |
| **Server scopes**     | local, user, project, dynamic, enterprise, claudeai, managed     |
| **Auth**              | OAuth with callback ports, XAA (Cross-App Access), token refresh |
| **Tool naming**       | `mcp__<server>__<tool>`                                          |
| **Connection states** | connected, failed, needs-auth, pending, disabled                 |
| **Pool ordering**     | Built-ins first, then MCP tools (sorted for cache stability)     |
| **Code size**         | 24 files, ~500KB                                                 |

---

### Skills System

**Source:** `src/skills/bundledSkills.ts:43-115`

| Type           | Loading                       | Count                                                                                |
| -------------- | ----------------------------- | ------------------------------------------------------------------------------------ |
| **Bundled**    | Compiled-in                   | ~19 (batch, claudeApi, debug, keybindings, loop, remember, simplify, skillify, etc.) |
| **Disk-based** | `.claude/skills/` directories | Variable                                                                             |
| **MCP-based**  | MCP server skills             | Variable                                                                             |

**Skill definition:**

```typescript
{
  name: string
  description: string
  aliases: string[]
  whenToUse: string
  argumentHint: string
  allowedTools: string[]       // Restrict available tools
  model: string                // Override model
  executionContext: 'inline' | 'fork'
}
```

---

### Hooks System

**Source:** `src/utils/hooks.ts:286-291`

20 hook events:

| Event                | Timeout            | Trust    |
| -------------------- | ------------------ | -------- |
| `Setup`              | 10min              | Required |
| `SessionStart`       | 10min              | Required |
| `SessionEnd`         | 1.5s (overridable) | Required |
| `PreToolUse`         | 10min              | Required |
| `PostToolUse`        | 10min              | Required |
| `PostToolUseFailure` | 10min              | Required |
| `StopHook`           | 10min              | Required |
| `PreCompact`         | 10min              | Required |
| `PostCompact`        | 10min              | Required |
| `SubagentStart`      | 10min              | Required |
| `SubagentStop`       | 10min              | Required |
| `TeammateIdle`       | 10min              | Required |
| `TaskCreated`        | 10min              | Required |
| `TaskCompleted`      | 10min              | Required |
| `ConfigChange`       | 10min              | Required |
| `CwdChanged`         | 10min              | Required |
| `FileChanged`        | 10min              | Required |
| `UserPromptSubmit`   | 10min              | Required |
| `PermissionRequest`  | 10min              | Required |
| `PermissionDenied`   | 10min              | Required |

- **Exit code 2** = blocking error
- **Async hooks:** Background execution via `shellCommand.background()`
- **asyncRewake:** Enqueues `task-notification` on completion

---

### Plugin System

**Source:** `src/plugins/builtinPlugins.ts:46-108`

| Property       | Details                                          |
| -------------- | ------------------------------------------------ |
| Structure      | `BUILTIN_PLUGINS` Map with enable/disable toggle |
| Each provides  | Skills, Hooks, MCP servers                       |
| User toggle    | `settings.enabledPlugins[pluginId]`              |
| Source tagging | `builtin@builtin` vs `marketplace@npm`           |
| Availability   | `isAvailable()` for conditional loading          |
| Default        | `defaultEnabled` flag with settings override     |

---

### Deep Link Protocol

**Source:** `src/utils/deepLink/parseDeepLink.ts:23-100`

URI scheme: `claude-cli://open?q=<query>&cwd=<path>&repo=<owner/repo>`

| Validation          | Rule                                             |
| ------------------- | ------------------------------------------------ |
| ASCII control chars | 0x00-0x1F, 0x7F → rejected                       |
| Unicode             | `partiallySanitizeUnicode()` strips hidden chars |
| Path                | Absolute paths only, ≤ 4096 chars                |
| Query length        | Max 5000 chars                                   |
| Windows fallback    | WT.exe → PowerShell → cmd.exe → fallback         |
| Long prefills       | Scroll warning above `LONG_PREFILL_THRESHOLD`    |

---

## 6. Infrastructure

### Bridge & CCR

**Source:** `src/bridge/bridgeMain.ts:1-79`, `src/bridge/jwtUtils.ts:72-256`

| Property       | Details                                            |
| -------------- | -------------------------------------------------- |
| Architecture   | Poll-based work fetching from Anthropic servers    |
| Heartbeat      | Connection health monitoring with backoff          |
| Multi-session  | Gated via GrowthBook                               |
| Auth           | OAuth via Claude AI login                          |
| JWT refresh    | Every 3h55m (tokens last ~4hrs)                    |
| Token storage  | Trusted device tokens separate from session tokens |
| Session runner | Spawns child Claude processes for work execution   |

---

### Cost Tracking & Billing

**Source:** `src/cost-tracker.ts:50-278`

| Tracked Metric | Details                                     |
| -------------- | ------------------------------------------- |
| Tokens         | Input, output, cache read/write per model   |
| Web searches   | Separate request count                      |
| Cost           | `calculateUSDCost()` with per-model pricing |
| Code changes   | Lines added/removed                         |
| Duration       | API duration vs wall clock                  |
| Storage        | `.claude/config.json` indexed by session ID |
| Display        | Breakdown by model, precision for < $0.50   |
| Visibility     | Only if `hasConsoleBillingAccess()`         |

---

### Vim Mode

**Source:** `src/vim/types.ts`

Full state machine implementation:

| Feature          | Details                                                |
| ---------------- | ------------------------------------------------------ |
| **Modes**        | INSERT and NORMAL with stateful command parsing        |
| **Operators**    | `d` (delete), `c` (change), `y` (yank), `>/<` (indent) |
| **Motions**      | `h/j/k/l`, `w/b/e`, `0/^/$`                            |
| **Text objects** | 11: `w, W, ", ', (, ), {, }, [, ], <, >`               |
| **Find**         | `f/F/t/T` with repeat                                  |
| **Dot-repeat**   | Full recorded change types                             |
| **Registers**    | Unnamed register with linewise flag                    |
| **Max count**    | `MAX_VIM_COUNT = 10,000`                               |

---

### Migration System

**Source:** `src/migrations/`

11 active migrations, each logging `tengu_migrate_*` telemetry events:

| Migration                                    | Purpose                              |
| -------------------------------------------- | ------------------------------------ |
| `migrateFennecToOpus`                        | Remap retired Fennec codename → Opus |
| `migrateLegacyOpusToCurrent`                 | Update old Opus model IDs            |
| `migrateOpusToOpus1m`                        | Enable 1M context for Opus users     |
| `migrateSonnet45ToSonnet46`                  | Sonnet version bump                  |
| `migrateAutoUpdatesToSettings`               | Auto-update config migration         |
| `migrateBypassPermissionsAcceptedToSettings` | Consent tracking                     |

---

### Worktree Isolation

**Source:** `src/utils/worktree.ts:66+`

| Feature          | Implementation                                   |
| ---------------- | ------------------------------------------------ |
| Creation         | `git worktree add`                               |
| Optimization     | Symlinks `node_modules` from main repo           |
| Sparse checkout  | `settings.worktree.sparsePaths`                  |
| Path safety      | Alphanumeric/dots/dashes/underscores, ≤ 64 chars |
| Session tracking | `WorktreeSession` metadata                       |
| Cache            | Clears memory file caches on entry               |
| Cleanup          | `executeWorktreeRemoveHook()` on session end     |

---

_This document is AI-generated analysis of publicly available source code. Information may be inaccurate, incomplete, or outdated. Feature details and timelines are speculative and may never ship._

_Source: `@anthropic-ai/claude-code` · Analysis date: 2026-03-31_
