<div align="center">

# 🔍 Claude Code — Source Code Analysis

**Reverse-engineered internals of Anthropic's Claude Code CLI**

[![Source](https://img.shields.io/badge/source-@anthropic--ai/claude--code-blueviolet)](https://www.npmjs.com/package/@anthropic-ai/claude-code)
[![Analysis Date](https://img.shields.io/badge/analysis-March%202026-blue)]()
[![TypeScript](https://img.shields.io/badge/TypeScript-512K%20lines-3178C6?logo=typescript&logoColor=white)]()
[![Tools](https://img.shields.io/badge/built--in%20tools-43-green)]()

_Unreleased features, secret flags, architecture internals & more_

> ⚠️ **Disclaimer**: This repository contains AI-assisted analysis of publicly available source code from the `@anthropic-ai/claude-code` npm package (source maps were accidentally published). Information may be inaccurate, incomplete, or outdated. Feature details and timelines are speculative and may never ship. This is an **unofficial community resource** and is **not affiliated with or endorsed by Anthropic**.

---

[Architecture](#-architecture) · [Tools](#-built-in-tools-43) · [Commands](#-slash-commands) · [Hidden Features](#-unreleased-features) · [Secret Flags](#-secret-cli-flags) · [Security](#-security--permissions) · [Environment Variables](#-environment-variables)

</div>

---

## 📊 Key Numbers

| Metric                  | Count  |
| ----------------------- | ------ |
| Source files            | 2,000+ |
| Lines of TypeScript     | ~512K  |
| Built-in tools          | 43     |
| CLI commands            | 100+   |
| Service modules         | 39     |
| React hooks             | 85     |
| UI components           | 144    |
| MCP integration files   | 24     |
| Utility files           | 564    |
| Permission system files | 24     |

---

## 🏗 Architecture

### Folder Structure

```
src/                          2,000+ files · ~512K lines
├── entrypoints/              CLI bootstrap & entry points
│   ├── cli.tsx               Fast-path bootstrap (39KB)
│   ├── init.ts               Initialization sequence
│   ├── mcp.ts                MCP server mode entry
│   └── sdk/                  Agent SDK type definitions
├── commands/                 101 dirs · 207 files · CLI command handlers
│   ├── commit.ts             Git commit command
│   ├── config/               Settings management
│   ├── memory/               Memory CRUD commands
│   └── ...                   (100+ more) doctor, upgrade, login, etc.
├── tools/                    43 tools · 184 files
│   ├── BashTool/             Shell command execution
│   ├── FileReadTool/         Filesystem reading
│   ├── FileEditTool/         Precise string replacement
│   ├── AgentTool/            Subagent spawning & orchestration
│   ├── MCPTool/              Dynamic MCP tool wrapper
│   ├── WebFetchTool/         HTTP fetch + HTML→markdown
│   └── ...                   (37+ more)
├── services/                 39 services · 130 files
│   ├── api/                  Claude API client (Anthropic SDK)
│   ├── mcp/                  MCP protocol (24 files)
│   ├── tools/                StreamingToolExecutor, registry
│   ├── compact/              Context compaction service
│   ├── analytics/            Telemetry & event tracking
│   ├── oauth/                Authentication flows
│   └── plugins/              Community plugin system
├── components/               144 files · Ink terminal UI components
├── hooks/                    85 React hooks (useStream, useTools…)
├── ink/                      Custom terminal renderer
│   ├── reconciler.ts         React reconciler for TTY
│   ├── layout/               Yoga flexbox engine bindings
│   └── termio/               ANSI escape sequence parser
├── utils/                    564 files · Helper functions
│   ├── permissions/          Permission system (24 files)
│   ├── bash/                 Bash AST parser (safety checks)
│   ├── model/                Model selection & routing
│   ├── settings/             Settings read/write layer
│   └── git.ts                Git porcelain operations
├── state/                    Centralized application state
├── bridge/                   Remote control protocol (31 files, ~400KB)
├── cli/                      Transport layer (WS/SSE)
├── memdir/                   Memory persistence layer
├── tasks/                    Background task system
├── coordinator/              Multi-agent orchestration
├── assistant/                KAIROS persistent mode
├── buddy/                    AI Companion Pet system
├── voice/                    Voice input pipeline
├── vim/                      Full Vim emulation
└── schemas/                  JSON validation schemas
```

### Boot Sequence

What happens when you type `claude`:

```
1. cli.tsx loads         → Fast-path: --version exits with zero imports (<50ms)
2. Feature flags         → bun:bundle eliminates dead code at build time
3. main.tsx initializes  → Commander.js parses argv, routes to handler
4. Config loaded         → settings.json + CLAUDE.md files + .envrc via direnv
5. Auth checked          → OAuth token or ANTHROPIC_API_KEY validated
6. GrowthBook init       → Remote feature flags fetched async
7. Tools assembled       → 43 built-in tools loaded from registry
8. MCP servers connected → stdio/SSE/WebSocket negotiated in parallel
9. System prompt built   → 10+ component sources assembled
10. REPL launched        → Ink renders terminal UI via React reconciler + Yoga
11. Query loop begins    → Event loop idles, background tasks on timers
```

### Query Loop — The Core Cycle

```
User types message → createUserMessage()
    ↓
Append to conversation history (in-memory array)
    ↓
Build system prompt (CLAUDE.md + tools + context + memory)
    ↓
Stream to Claude API (Anthropic SDK · SSE)
    ↓
Parse response tokens (incremental terminal render)
    ↓
if tool_use blocks found → findToolByName() → canUseTool()
    ↓
StreamingToolExecutor (concurrent-safe, parallel-capable)
    ↓
Collect tool results → loop back to API
    ↓
Display final response (markdown rendered)
    ↓
Post-sampling hooks (auto-compact · memory extract · dream)
    ↓
Wait for next input (cycle restarts)
```

**Key implementation details:**

- **Streaming by default** — tokens render as they arrive
- **Parallel tool execution** — `StreamingToolExecutor` runs concurrent tools
- **Agentic loop** — cycles until Claude emits no `tool_use` blocks
- **React rendering** — Ink reconciler + Yoga flexbox, ANSI diffs minimize redraws
- **Interrupt handling** — `Ctrl+C` aborts executor, preserves conversation
- **Post-sampling hooks** — compaction check, memory extraction, dream consolidation

---

## 🔧 Built-in Tools (43)

### File Operations & Execution

| Tool               | Description                      | Category |
| ------------------ | -------------------------------- | -------- |
| `FileReadTool`     | Read files from filesystem       | Core     |
| `FileEditTool`     | Precise string replacement edits | Core     |
| `FileWriteTool`    | Write new files                  | Core     |
| `GlobTool`         | File pattern matching            | Core     |
| `GrepTool`         | Content search (ripgrep-backed)  | Core     |
| `NotebookEditTool` | Jupyter notebook editing         | Core     |
| `BashTool`         | Shell command execution          | Core     |
| `PowerShellTool`   | Windows PowerShell execution     | Platform |

### Agents, Planning & MCP

| Tool                   | Description                       | Category    |
| ---------------------- | --------------------------------- | ----------- |
| `AgentTool`            | Subagent spawning & orchestration | Core        |
| `SendMessageTool`      | Inter-agent communication         | Multi-agent |
| `TaskCreateTool`       | Create background tasks           | Task        |
| `TaskGetTool`          | Retrieve task status              | Task        |
| `TaskListTool`         | List all tasks                    | Task        |
| `TaskUpdateTool`       | Update task progress              | Task        |
| `TaskStopTool`         | Stop running tasks                | Task        |
| `TaskOutputTool`       | Get task output                   | Task        |
| `EnterPlanModeTool`    | Enter planning phase              | Planning    |
| `ExitPlanModeV2Tool`   | Exit planning phase               | Planning    |
| `EnterWorktreeTool`    | Create git worktree               | Worktree    |
| `ExitWorktreeTool`     | Exit git worktree                 | Worktree    |
| `MCPTool`              | Dynamic MCP tool wrapper          | MCP         |
| `ListMcpResourcesTool` | List MCP resources                | MCP         |
| `ReadMcpResourceTool`  | Read MCP resources                | MCP         |

### Search & Fetch

| Tool             | Description                | Category      |
| ---------------- | -------------------------- | ------------- |
| `WebFetchTool`   | HTTP fetch + HTML→markdown | Core          |
| `WebSearchTool`  | Web search                 | Core          |
| `ToolSearchTool` | Search available tools     | Feature-gated |

### System & Utility

| Tool                  | Description           | Category |
| --------------------- | --------------------- | -------- |
| `AskUserQuestionTool` | Prompt user for input | Core     |
| `TodoWriteTool`       | Write TODO files      | Core     |
| `SkillTool`           | Invoke skills         | Core     |
| `BriefTool`           | Brief/summary output  | Core     |
| `ConfigTool`          | Settings management   | ANT-only |
| `TungstenTool`        | Virtual terminal      | ANT-only |

### Feature-Gated / Experimental

These tools are compiled into the binary but only surface when the corresponding feature flag is enabled:

| Tool                      | Gate                      | Description             |
| ------------------------- | ------------------------- | ----------------------- |
| `SleepTool`               | `PROACTIVE` / `KAIROS`    | Pause between actions   |
| `WebBrowserTool`          | `WEB_BROWSER_TOOL`        | Browser automation      |
| `TerminalCaptureTool`     | `TERMINAL_PANEL`          | Terminal screen capture |
| `CtxInspectTool`          | `CONTEXT_COLLAPSE`        | Context inspection      |
| `SnipTool`                | `HISTORY_SNIP`            | Snippet compression     |
| `WorkflowTool`            | `WORKFLOW_SCRIPTS`        | Workflow automation     |
| `ListPeersTool`           | `UDS_INBOX`               | List peer sessions      |
| `OverflowTestTool`        | `OVERFLOW_TEST_TOOL`      | Overflow testing        |
| `MonitorTool`             | `MONITOR_TOOL`            | Monitoring              |
| `VerifyPlanExecutionTool` | `CLAUDE_CODE_VERIFY_PLAN` | Plan verification       |
| `SuggestBackgroundPRTool` | `USER_TYPE=ant`           | Suggest background PRs  |
| `REPLTool`                | `USER_TYPE=ant`           | REPL mode               |

---

## ⌨️ Slash Commands

### Publicly Available Commands

| Command              | Description                  |
| -------------------- | ---------------------------- |
| `/help`              | Show help text               |
| `/compact`           | Compact conversation context |
| `/clear`             | Clear conversation           |
| `/config`            | Settings management          |
| `/cost`              | Show session cost            |
| `/doctor`            | Diagnostic checks            |
| `/init`              | Initialize project           |
| `/login` / `/logout` | Authentication               |
| `/memory`            | Memory management            |
| `/model`             | Change model                 |
| `/mcp`               | MCP server management        |
| `/resume`            | Resume session               |
| `/vim`               | Toggle vim mode              |
| `/theme`             | Change theme                 |
| `/plan`              | Toggle plan mode             |
| `/permissions`       | Manage permissions           |
| `/diff`              | Show file changes            |
| `/review`            | Code review                  |
| `/export`            | Export transcript            |
| `/upgrade`           | Upgrade CLI                  |
| `/status`            | Show status                  |
| `/hooks`             | Manage hooks                 |
| `/plugin`            | Plugin management            |
| `/skills`            | Skills management            |
| `/agents`            | Agent management             |
| `/fast`              | Toggle fast mode             |

### Hidden / Internal Slash Commands (26)

> Not shown in `--help`, discovered via source analysis

| Command              | Description                | Access        |
| -------------------- | -------------------------- | ------------- |
| `/ctx-viz`           | Context visualizer         | Internal      |
| `/btw`               | Side question (quick note) | Available     |
| `/good-claude`       | Easter egg                 | Internal      |
| `/teleport`          | Session teleport           | Internal      |
| `/share`             | Session sharing            | Internal      |
| `/summary`           | Summarize conversation     | Internal      |
| `/ultraplan`         | Advanced 30-min planning   | Feature-gated |
| `/subscribe-pr`      | PR webhooks                | Feature-gated |
| `/autofix-pr`        | Auto PR fix                | Internal      |
| `/ant-trace`         | Internal trace             | ANT-only      |
| `/perf-issue`        | Performance report         | Internal      |
| `/debug-tool-call`   | Debug tool calls           | Internal      |
| `/bughunter`         | Bug debugging              | Internal      |
| `/force-snip`        | Force snippet              | Feature-gated |
| `/mock-limits`       | Mock rate limits           | Internal      |
| `/bridge-kick`       | Bridge test                | Internal      |
| `/backfill-sessions` | Backfill sessions          | Internal      |
| `/break-cache`       | Invalidate cache           | Internal      |
| `/agents-platform`   | Agent platform             | ANT-only      |
| `/onboarding`        | Setup flow                 | Internal      |
| `/oauth-refresh`     | Token refresh              | Internal      |
| `/env`               | Environment inspect        | Internal      |
| `/reset-limits`      | Reset rate limits          | Internal      |
| `/dream`             | Memory consolidation       | Feature-gated |
| `/version`           | Internal version           | Internal      |
| `/init-verifiers`    | Verifier setup             | Internal      |

---

## 🔮 Unreleased Features

### 🐾 BUDDY — AI Companion Pet

> _Easter Egg · Unreleased_

Every user gets a unique virtual pet generated deterministically from their account ID. The buddy system includes species, rarity tiers, stats, cosmetics, and animations.

**Verified in source:** `src/buddy/types.ts`, `src/buddy/companion.ts`, `src/buddy/CompanionSprite.tsx`

| Property    | Details                                                                                                                                |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **Release** | Teaser: Apr 1–7, 2026 · Live: May 2026                                                                                                 |
| **Species** | duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk |
| **Rarity**  | Common (60%), Uncommon (25%), Rare (10%), Epic (4%), Legendary (1%)                                                                    |
| **Eyes**    | `·  ✦  ×  ◉  @  °`                                                                                                                     |
| **Hats**    | none, crown, tophat, propeller, halo, wizard, beanie, tinyduck                                                                         |
| **Stats**   | DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK                                                                                              |
| **Soul**    | Generated once per user, stored forever in config                                                                                      |
| **Shiny**   | ~1% chance of shiny variant                                                                                                            |

> **Fun fact:** The species names are encoded as `String.fromCharCode()` calls in the source to evade build-time string filters (e.g. "capybara" collides with a model codename).

---

### 🔮 KAIROS — Persistent Assistant

> _Unreleased · Feature gate: `feature('KAIROS') + tengu_kairos`_

An always-on mode where Claude remembers everything across sessions with daily logs and overnight "dream" memory consolidation.

| Property            | Details                                                        |
| ------------------- | -------------------------------------------------------------- |
| Daily Logs          | `~/.claude/.../logs/YYYY/MM/DD.md`                             |
| Log mode            | append-only                                                    |
| Dream bash          | Read-only                                                      |
| Blocking budget     | 15s max — auto-backgrounds                                     |
| Status modes        | normal · proactive                                             |
| **Exclusive Tools** | `SendUserFile`, `PushNotification`, `SubscribePR`, `SleepTool` |

**Dream Phases:** Orient → Gather → Consolidate → Prune

---

### ⚡ ULTRAPLAN — 30-Min Remote Planning

> _Unreleased · `src/commands/ultraplan.tsx` (66KB)_

For complex tasks, Claude spins up a separate cloud instance that explores and plans for up to 30 minutes. Users review and approve the plan in their browser.

| Property      | Details                                                |
| ------------- | ------------------------------------------------------ |
| Model         | Opus 4.6 via `tengu_ultraplan_model`                   |
| Poll interval | 3s                                                     |
| Flow          | Poll → ExitPlanMode → approve/reject → loop or execute |
| Teleport      | "Teleport to terminal" archives remote, runs locally   |

---

### ⚙️ Coordinator Mode — Multi-Agent

> _ENV activated: `CLAUDE_CODE_COORDINATOR_MODE=1`_

Claude becomes a manager, breaking tasks into pieces and assigning each to parallel worker agents.

**Verified in source:** `src/coordinator/`, `src/tools.ts` lines 120-134

| Property            | Details                           |
| ------------------- | --------------------------------- |
| Protocol            | `<task-notification>` XML         |
| Isolation           | Scratch dirs via `tengu_scratch`  |
| Continue            | via `SendMessage`                 |
| Notification fields | status, summary, tokens, duration |

---

### 📬 UDS Inbox — Cross-Session IPC

> _Unreleased · Feature gate: `UDS_INBOX`_

Multiple Claude sessions on the same machine can send messages to each other like a team chat.

| Property     | Details                                     |
| ------------ | ------------------------------------------- |
| Teammate     | `to: "researcher"`                          |
| Local socket | `to: "uds:/.../sock"`                       |
| Remote       | `to: "bridge:..."`                          |
| Discovery    | `ListPeersTool` reads `~/.claude/sessions/` |

---

### 📱 Bridge — Remote Control

> _Unreleased · Feature gate: `BRIDGE_MODE`_

Run Claude locally but control it from your phone or claude.ai in the browser.

**Verified in source:** `src/bridge/` (31 files, ~400KB total)

| Property         | Details                                   |
| ---------------- | ----------------------------------------- |
| Command          | `claude remote-control`                   |
| Init API         | `POST /v1/environments/bridge`            |
| Transport        | poll → WebSocket                          |
| Control messages | `initialize`, `set_model`, `can_use_tool` |
| JWT refresh      | Every 3h55m (tokens last ~4hrs)           |

---

### 👾 Daemon Mode — Session Supervisor

> _Unreleased · Feature gate: `DAEMON`_

Run Claude sessions in the background like system services.

| Property   | Details                                  |
| ---------- | ---------------------------------------- |
| Background | `claude --bg <prompt>` in tmux           |
| On exit    | detach (session persists)                |
| Commands   | `daemon`, `ps`, `logs`, `attach`, `kill` |

---

### 🌙 Auto-Dream — Memory Consolidation

> _Unreleased · Feature gate: `tengu_onyx_plover`_

Between sessions, Claude reviews what it learned and organizes notes into structured memory files.

| Property     | Details                               |
| ------------ | ------------------------------------- |
| Trigger      | ≥24h + ≥5 sessions since last dream   |
| Output limit | <25KB                                 |
| Phases       | Orient → Gather → Consolidate → Prune |

---

## 🚩 Secret CLI Flags

| Flag                     | Description                | Status     |
| ------------------------ | -------------------------- | ---------- |
| `--bare`                 | No hooks/plugins/memory    | Available  |
| `--dump-system-prompt`   | Print system prompt & exit | Available  |
| `--daemon-worker=<k>`    | Daemon subprocess          | Unreleased |
| `--computer-use-mcp`     | Computer Use MCP server    | Unreleased |
| `--claude-in-chrome-mcp` | Chrome MCP                 | Unreleased |
| `--chrome-native-host`   | Chrome native messaging    | Unreleased |
| `--bg`                   | Background tmux session    | Available  |
| `--spawn`                | Spawn mode                 | Unreleased |
| `--capacity <n>`         | Parallel worker limit      | Unreleased |
| `--worktree / -w`        | Git worktree isolation     | Available  |

---

## 🔬 Build-Time Feature Flags

32 feature flags compiled into the binary via `bun:bundle`. Dead code elimination removes disabled features from external builds.

```typescript
import { feature } from "bun:bundle";

// Example: conditionally load a tool
const SleepTool =
  feature("PROACTIVE") || feature("KAIROS")
    ? require("./tools/SleepTool/SleepTool.js").SleepTool
    : null;
```

| Flag                  | Description                     |
| --------------------- | ------------------------------- |
| `KAIROS`              | Persistent assistant mode       |
| `PROACTIVE`           | Background sleeping agents      |
| `COORDINATOR_MODE`    | Multi-agent orchestration       |
| `BRIDGE_MODE`         | Remote control                  |
| `DAEMON`              | Session supervisor              |
| `BG_SESSIONS`         | Background sessions             |
| `ULTRAPLAN`           | 30-min cloud planning           |
| `BUDDY`               | AI companion pet                |
| `TORCH`               | ??? (opaque)                    |
| `WORKFLOW_SCRIPTS`    | Workflow automation             |
| `VOICE_MODE`          | Voice input · `/voice` works    |
| `TEMPLATES`           | Job templates                   |
| `CHICAGO_MCP`         | Computer use (live for Max/Pro) |
| `UDS_INBOX`           | Unix socket IPC                 |
| `REACTIVE_COMPACT`    | Realtime compaction             |
| `CONTEXT_COLLAPSE`    | Smart context collapse          |
| `HISTORY_SNIP`        | Snippet compression             |
| `CACHED_MICROCOMPACT` | Cached micro-compaction         |
| `TOKEN_BUDGET`        | Per-turn token budgets          |
| `EXTRACT_MEMORIES`    | Background memory extraction    |
| `OVERFLOW_TEST_TOOL`  | Overflow testing                |
| `TERMINAL_PANEL`      | Terminal screen capture         |
| `WEB_BROWSER_TOOL`    | Browser automation              |
| `FORK_SUBAGENT`       | Agent forking                   |
| `DUMP_SYS_PROMPT`     | Print system prompt             |
| `ABLATION_BASE`       | Research mode                   |
| `BYOC_RUNNER`         | Bring-your-own-compute runner   |
| `SELF_HOSTED`         | Self-hosted mode                |
| `MONITOR_TOOL`        | Monitoring                      |
| `CCR_AUTO`            | Auto cloud compute              |
| `MEM_SHAPE_TEL`       | Memory analytics                |
| `SKILL_SEARCH`        | Experimental skill search       |

---

## 🌍 Environment Variables

### 🐛 Debug & Profiling

| Variable                       | Description               |
| ------------------------------ | ------------------------- |
| `CLAUDE_CODE_PERFETTO_TRACE`   | Chrome trace via Perfetto |
| `CLAUDE_CODE_PROFILE_STARTUP`  | Startup timing profiler   |
| `CLAUDE_CODE_FRAME_TIMING_LOG` | Frame timing log output   |
| `CLAUDE_CODE_VCR_RECORD`       | Record HTTP interactions  |
| `CLAUDE_CODE_DEBUG_REPAINTS`   | Visualize UI repaints     |

### 🎛️ Runtime Overrides

| Variable                         | Description                  |
| -------------------------------- | ---------------------------- |
| `CLAUDE_CODE_OVERRIDE_DATE`      | Inject fake date             |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | Override context window      |
| `MAX_THINKING_TOKENS`            | Override thinking budget     |
| `CLAUDE_CODE_EXTRA_BODY`         | Inject extra API params      |
| `AUTOCOMPACT_PCT_OVERRIDE`       | Override compact threshold   |
| `IDLE_THRESHOLD_MINUTES`         | Idle threshold (75m default) |

### ⚠️ Safety Bypass (Dangerous)

| Variable                          | Description                          |
| --------------------------------- | ------------------------------------ |
| `DISABLE_COMMAND_INJECTION_CHECK` | Skip injection guard — **DANGEROUS** |
| `CLAUDE_CODE_ABLATION_BASELINE`   | Disable ALL safety features          |
| `DISABLE_INTERLEAVED_THINKING`    | Disable interleaved thinking         |

### 🔑 Anthropic Internal

| Variable                       | Description                  |
| ------------------------------ | ---------------------------- |
| `USER_TYPE=ant`                | Unlock all internal features |
| `CLAUDE_INTERNAL_FC_OVERRIDES` | Override feature flags       |
| `CLAUDE_MORERIGHT`             | "More right" layout          |
| `CLAUDE_CODE_UNDERCOVER`       | Undercover mode              |
| `CLAUBBIT`                     | Internal testing             |

---

## 🔒 Security & Permissions

### 3-Layer Permission Model

```
Layer 1: Tool Registry Filter
  └── filterToolsByDenyRules() removes denied tools BEFORE Claude sees them
  └── Claude cannot call tools blocked at this layer

Layer 2: Per-call Permission Check
  └── canUseTool() checks allow/deny rules per invocation
  └── Checks tool name, arguments, and working directory

Layer 3: Interactive User Prompt
  └── If no rule matches → user prompted
  └── Options: allow once · allow always · deny
```

**Permission Modes:**

- `default` — interactive prompting
- `plan` — requires explicit approval
- `auto` — AI classifier decides
- `bypassPermissions` — full bypass
- `dontAsk` — auto-deny
- `acceptEdits` — auto-accept file edits

### Bash Safety — AST-Level Analysis

The Bash tool includes a full shell AST parser in `utils/bash/`. Before executing any command, it parses the AST to detect dangerous patterns:

- `rm -rf /` — destructive deletion
- Fork bombs
- `curl | bash` — remote code execution
- `sudo` escalation
- TTY injection
- History manipulation
- `sleep N` where N >= 2s is blocked

### Prompt Injection Defenses (6 Layers)

1. **System prompt warning** — instructs model to flag suspected injection
2. **Unicode smuggling prevention** — `partiallySanitizeUnicode()` strips hidden chars
3. **Subprocess env protection** — prevents secret exfiltration via untrusted content
4. **Tool result budgeting** — `toolResultStorage.ts` prevents overflow attacks
5. **Trusted/untrusted separation** — hooks = trusted, tool results = potentially untrusted
6. **Hook trust gates** — ALL hooks require workspace trust dialog acceptance

### Settings.json Rule Format

```json
{
  "permissions": {
    "allow": ["Bash(git *)"],
    "deny": ["Bash(rm *)", "Write"]
  }
}
```

---

## 🧠 Models & Intelligence

### Complete Model Registry

| Family     | Versions                | Providers                    |
| ---------- | ----------------------- | ---------------------------- |
| **Opus**   | 4.6, 4.5, 4.1, 4.0      | 1P, Bedrock, Vertex, Foundry |
| **Sonnet** | 4.6, 4.5, 4.0, 3.7, 3.5 | 1P, Bedrock, Vertex, Foundry |
| **Haiku**  | 4.5, 3.5                | 1P, Bedrock, Vertex, Foundry |

**Model aliases:** `opus`, `sonnet`, `haiku`, `best` (Opus 4.6), `opusplan` (Opus in plan mode else Sonnet)

### Model Selection Priority (5 layers)

```
1. /model session override     (highest)
2. --model flag
3. ANTHROPIC_MODEL env
4. settings file
5. built-in default            (lowest)
```

- Max/Team subscribers → Opus 4.6
- Standard users → Sonnet 4.6
- Anthropic employees (Ants) → Opus 4.6 [1m]

### Fast Mode

**Verified in source:** `src/utils/fastMode.ts`

| Property        | Details                                        |
| --------------- | ---------------------------------------------- |
| Supported model | Opus 4.6 only                                  |
| Beta header     | `fast-mode-2026-02-01`                         |
| Pricing         | $30/$150 per MTok vs $5/$25 normal (6× markup) |
| Availability    | 1P only (not Bedrock/Vertex/Foundry)           |
| Disable         | `CLAUDE_CODE_DISABLE_FAST_MODE=1`              |
| Default         | Enabled for paying users                       |

### UltraThink Enhanced Reasoning

**Verified in source:** `src/utils/thinking.ts`

| Property | Details                                                              |
| -------- | -------------------------------------------------------------------- |
| Modes    | adaptive (model decides), enabled (fixed budget), disabled           |
| Models   | Opus 4.6 and Sonnet 4.6 support adaptive thinking                    |
| Trigger  | `hasUltrathinkKeyword()` detects "ultrathink" in user input          |
| Visual   | 7-color rainbow shimmer animation                                    |
| Gate     | Compile-time `feature('ULTRATHINK')` + runtime `tengu_turtle_carbon` |

### Context Window & Tokens

| Property          | Value                               |
| ----------------- | ----------------------------------- |
| Default context   | 200,000 tokens                      |
| Extended context  | 1M via `[1m]` model suffix          |
| 1M gate           | Beta header `context-1m-2025-08-07` |
| Opus 4.6 output   | 64K default, 128K upper limit       |
| Sonnet 4.6 output | 32K default, 128K upper limit       |
| Other models      | 32K default, 64K upper limit        |
| HIPAA disable     | `CLAUDE_CODE_DISABLE_1M_CONTEXT`    |

### AutoCompact & Context Management

**Verified in source:** `src/services/compact/autoCompact.ts`

| Property          | Value                                            |
| ----------------- | ------------------------------------------------ |
| Buffer tokens     | 13,000                                           |
| Trigger           | ~187K of 200K (~80% usage)                       |
| Circuit breaker   | 3 consecutive failures                           |
| MicroCompact      | Bash, FileRead, Grep output only                 |
| Post-compact      | Restores top-5 files (5K tokens each, 25K total) |
| Warning threshold | 20,000 buffer tokens                             |
| Manual compact    | 3,000 buffer tokens                              |

---

## 🌱 GrowthBook Feature Gates

Gradual rollout gates in the `tengu_*` namespace:

| Gate                      | Function                     |
| ------------------------- | ---------------------------- |
| `tengu_malort_pedway`     | Computer use                 |
| `tengu_onyx_plover`       | Auto-dream                   |
| `tengu_kairos`            | Assistant mode               |
| `tengu_ultraplan_model`   | Planning model               |
| `tengu_cobalt_raccoon`    | Auto-compact                 |
| `tengu_portal_quail`      | Memory extract               |
| `tengu_harbor`            | MCP allowlist                |
| `tengu_scratch`           | Worker scratch dirs          |
| `tengu_herring_clock`     | Team memory                  |
| `tengu_chomp_inflection`  | Prompt suggestions           |
| `tengu_turtle_carbon`     | UltraThink                   |
| `tengu_pewter_ledger`     | Plan file A/B test           |
| `tengu_marble_sandcastle` | Fast mode binary requirement |

---

## 🧪 API Beta Headers

`anthropic-beta` header values used by Claude Code:

| Feature              | Header Date  |
| -------------------- | ------------ |
| Interleaved Thinking | `2025-05-14` |
| 1M Context Window    | `2025-08-07` |
| Structured Outputs   | `2025-12-15` |
| Advanced Tool Use    | `2025-11-20` |
| Tool Search          | `2025-10-19` |
| Effort Levels        | `2025-11-24` |
| Task Budgets         | `2026-03-13` |
| Fast Mode            | `2026-02-01` |
| Prompt Cache Scoping | `2026-01-05` |
| CLI Internal (ant)   | `2026-02-09` |

---

## 🔌 Extension Points

### MCP Integration

**24 files, ~500KB of MCP client code** in `src/services/mcp/`

| Property          | Details                                                      |
| ----------------- | ------------------------------------------------------------ |
| Transports        | stdio, SSE, HTTP, WebSocket, SDK, CloudAI proxy              |
| Server scopes     | local, user, project, dynamic, enterprise, claudeai, managed |
| Auth              | OAuth with callback ports, XAA, token refresh                |
| Tool naming       | `mcp__<server>__<tool>`                                      |
| Connection states | connected, failed, needs-auth, pending, disabled             |

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/me/projects"
      ]
    },
    "remote-api": {
      "url": "https://api.example.com/mcp",
      "transport": "sse"
    }
  }
}
```

### Hooks System (20 Events)

**Verified in source:** `src/utils/hooks.ts` (5,023 lines, 159KB)

| Event                                    | Description                 |
| ---------------------------------------- | --------------------------- |
| `Setup`                                  | Initial setup               |
| `SessionStart` / `SessionEnd`            | Session lifecycle           |
| `PreToolUse` / `PostToolUse`             | Before/after tool execution |
| `PostToolUseFailure`                     | After tool failure          |
| `StopHook`                               | Stop condition              |
| `PreCompact` / `PostCompact`             | Before/after compaction     |
| `SubagentStart` / `SubagentStop`         | Subagent lifecycle          |
| `TeammateIdle`                           | Teammate idle notification  |
| `TaskCreated` / `TaskCompleted`          | Task lifecycle              |
| `ConfigChange`                           | Configuration changed       |
| `CwdChanged`                             | Working directory changed   |
| `FileChanged`                            | File modification           |
| `UserPromptSubmit`                       | User submits prompt         |
| `PermissionRequest` / `PermissionDenied` | Permission events           |

- **Tool hook timeout:** 10 minutes
- **SessionEnd timeout:** 1.5s (overridable via env)
- **Trust:** ALL hooks require workspace trust dialog
- **Async:** `asyncRewake` hooks enqueue `task-notification` on completion
- **Exit code 2** = blocking error

### CLAUDE.md Loading Hierarchy

```
Priority (low → high):
1. /etc/claude-code/CLAUDE.md          (system-wide)
2. ~/.claude/CLAUDE.md                 (user-level)
3. project/CLAUDE.md                   (project-level)
   project/.claude/CLAUDE.md
   project/.claude/rules/*.md
4. CLAUDE.local.md                     (local override, highest)
```

- `@include` directive: `@path`, `@./relative`, `@~/home`, `@/absolute`
- Frontmatter `paths:` field for glob-based applicability
- HTML comment stripping for `<!-- ... -->` blocks
- Circular reference prevention via tracking

### Skills System

| Type           | Description                                                                                             |
| -------------- | ------------------------------------------------------------------------------------------------------- |
| **Bundled**    | ~19 compiled-in skills (batch, claudeApi, debug, keybindings, loop, remember, simplify, skillify, etc.) |
| **Disk-based** | Loaded from `.claude/skills/` directories                                                               |
| **MCP-based**  | Skills from MCP servers                                                                                 |

### Plugin System

- `BUILTIN_PLUGINS` map with per-plugin enable/disable
- Each plugin provides: Skills, Hooks, MCP servers
- Source tagging: `builtin@builtin` vs `marketplace@npm`
- User toggle via `settings.enabledPlugins[pluginId]`

---

## 🏗️ Infrastructure Internals

### Multi-Agent Swarm System

| Backend              | Description                                                  |
| -------------------- | ------------------------------------------------------------ |
| **InProcessBackend** | Same Node.js process via `AsyncLocalStorage` (zero overhead) |
| **TmuxBackend**      | Separate tmux panes/windows                                  |
| **ITermBackend**     | iTerm2 split panes                                           |

- Teams: `.claude/teams/<name>.json` with lead/member config
- Identity: `agentName@teamName`
- Tool restrictions: agents cannot use `TaskOutput`, `ExitPlanMode`, `AskUserQuestion`, `Workflow`

### Memory System (4 Types)

| Type        | Scope            | Default |
| ----------- | ---------------- | ------- |
| `user`      | Always private   | Private |
| `feedback`  | Default private  | Private |
| `project`   | Bias toward team | Team    |
| `reference` | Usually team     | Team    |

Storage format: YAML frontmatter + Markdown content

### Vim Mode

Full state machine with INSERT and NORMAL modes:

- **Operators:** `d` (delete), `c` (change), `y` (yank), `>/<` (indent)
- **Motions:** `h/j/k/l`, `w/b/e`, `0/^/$`
- **Text objects:** 11 types — `w, W, ", ', (, ), {, }, [, ], <, >`
- **Find:** `f/F/t/T` with repeat
- **Dot-repeat** with full change recording
- **Max count:** 10,000

### Cost Tracking

**Verified in source:** `src/cost-tracker.ts`

Tracks per-session:

- Input/output/cache tokens per model
- Web search request count
- USD cost via `calculateUSDCost()` with per-model pricing
- Lines of code added/removed
- API duration vs wall clock duration
- Stored in `.claude/config.json` indexed by session ID

### Internal Codename Map

| Codename          | Meaning                                                       |
| ----------------- | ------------------------------------------------------------- |
| **Capybara**      | Active internal model (encoded as charCodes to evade filters) |
| **Fennec**        | Retired Opus codename                                         |
| **Turtle Carbon** | UltraThink feature gate                                       |
| **Pewter Ledger** | Plan file structure A/B test                                  |
| **Birch Mist**    | First-message optimization experiment                         |
| **Penguin Mode**  | Fast mode (internal name)                                     |

---

## 🗝️ Hardcoded SDK API Keys

> These SDK keys are baked into the binary for specific Anthropic environments. They are **not** user credentials.

| Environment | Key Prefix             |
| ----------- | ---------------------- |
| Ant prod    | `sdk-xRVcrliHIlrg4og4` |
| Ant dev     | `sdk-yZQvlplybuXjYh6L` |
| External    | `sdk-zAZezfDKGoZuXXKe` |

---

## 📚 Source References

Key source files for deeper exploration:

| File                            | Description                | Size  |
| ------------------------------- | -------------------------- | ----- |
| `src/main.tsx`                  | Main application           | 803KB |
| `src/tools.ts`                  | Tool registry              | 17KB  |
| `src/commands.ts`               | Command registry           | 25KB  |
| `src/Tool.ts`                   | Tool interface (793 lines) | 29KB  |
| `src/QueryEngine.ts`            | Query engine               | 46KB  |
| `src/utils/hooks.ts`            | Hooks system               | 159KB |
| `src/utils/messages.ts`         | Message handling           | 193KB |
| `src/utils/sessionStorage.ts`   | Session storage            | 180KB |
| `src/bridge/bridgeMain.ts`      | Bridge orchestrator        | 115KB |
| `src/bridge/replBridge.ts`      | Bridge REPL                | 100KB |
| `src/utils/teleport.tsx`        | Teleport system            | 175KB |
| `src/utils/attachments.ts`      | Attachments                | 127KB |
| `src/utils/ansiToPng.ts`        | ANSI to PNG export         | 214KB |
| `src/commands/ultraplan.tsx`    | UltraPlan impl             | 66KB  |
| `src/commands/insights.ts`      | Usage insights             | 115KB |
| `src/utils/fastMode.ts`         | Fast mode impl             | 17KB  |
| `src/utils/thinking.ts`         | UltraThink                 | 5KB   |
| `src/cost-tracker.ts`           | Cost tracking              | 10KB  |
| `src/buddy/types.ts`            | Buddy pet types            | 3KB   |
| `src/buddy/CompanionSprite.tsx` | Pet rendering              | 45KB  |

---

## 🙏 Credits

- **Original source:** [`@anthropic-ai/claude-code`](https://www.npmjs.com/package/@anthropic-ai/claude-code) npm package (source maps accidentally published)
- **Analysis inspiration:** [ccleaks.com](https://www.ccleaks.com/) by [Abhishek Tiwari](https://abhishektiwari.co/) ([@baanditeagle](https://x.com/baanditeagle))
- **Initial discovery:** [@Fried_rice on X](https://x.com/Fried_rice/status/2038894956459290963)

---

## ⚖️ Legal Notice

This repository is for **educational and research purposes only**. All code and analysis is based on publicly available source code that was accidentally published by Anthropic via npm source maps. No proprietary systems were accessed. Feature details are speculative and may never be released.

If you are from Anthropic and would like this repository taken down, please open an issue or contact the repository owner.

---

<div align="center">

**⭐ Star this repo if you found it useful!**

_Last updated: March 31, 2026_

</div>
