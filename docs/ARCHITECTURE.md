# 🏗️ Claude Code — Architecture Deep Dive

> A detailed technical analysis of Claude Code's internal architecture, based on the TypeScript source code from the `@anthropic-ai/claude-code` npm package.

---

## Table of Contents

- [System Overview](#system-overview)
- [Tool System Architecture](#tool-system-architecture)
- [System Prompt Architecture](#system-prompt-architecture)
- [Ink Terminal Rendering Engine](#ink-terminal-rendering-engine)
- [MCP Integration](#mcp-integration)
- [Permission System](#permission-system)
- [Context Management](#context-management)
- [Multi-Agent Swarm System](#multi-agent-swarm-system)
- [Bridge & CCR System](#bridge--ccr-system)
- [Memory System](#memory-system)
- [Worktree Isolation](#worktree-isolation)
- [Migration System](#migration-system)
- [Server / DirectConnect System](#server--directconnect-system)
- [Prompt Cache Optimization](#prompt-cache-optimization)

---

## System Overview

Claude Code is a ~512K-line TypeScript application built with [Bun](https://bun.sh/) and rendered using a custom React reconciler for the terminal (built on [Ink](https://github.com/vadimdemedes/ink)). It implements a full agentic loop where user prompts are streamed to the Claude API, tool calls are executed locally, and results are fed back in a cycle until completion.

### Core Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                    CLI Entry (cli.tsx)                │
│          Fast-path check → Commander.js routing      │
└─────────────┬───────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────┐
│              Initialization Pipeline                  │
│  Config → Auth → GrowthBook → Tools → MCP → Prompt  │
└─────────────┬───────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────┐
│              REPL (Ink + Yoga Flexbox)               │
│                                                      │
│  ┌──────────┐  ┌───────────┐  ┌──────────────────┐ │
│  │ Input    │→ │ Query     │→ │ API Stream       │ │
│  │ Handler  │  │ Engine    │  │ (Anthropic SDK)  │ │
│  └──────────┘  └───────────┘  └────────┬─────────┘ │
│                                         │           │
│  ┌──────────────────────────────────────▼─────────┐ │
│  │          StreamingToolExecutor                   │ │
│  │   ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐   │ │
│  │   │Bash │ │File │ │Web  │ │Agent│ │MCP  │   │ │
│  │   │Tool │ │Edit │ │Fetch│ │Tool │ │Tool │   │ │
│  │   └─────┘ └─────┘ └─────┘ └─────┘ └─────┘   │ │
│  └───────────────────────────┬─────────────────────┘ │
│                               │                      │
│  ┌────────────────────────────▼────────────────────┐ │
│  │           Post-Sampling Pipeline                 │ │
│  │   AutoCompact → Memory Extract → Dream Mode     │ │
│  └──────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

---

## Tool System Architecture

**Source:** `src/Tool.ts` (793 lines, 29KB)

Every tool implements a comprehensive interface with permissions, concurrency safety, schema validation, deferred loading, and search hints.

### Tool Interface

```typescript
interface Tool {
  name: string;
  description(): string;
  prompt(): string;

  // Schema
  inputSchema: ZodSchema;
  outputSchema?: ZodSchema;

  // Core
  call(input: unknown, context: ToolUseContext): Promise<ToolResult>;

  // Permissions
  checkPermissions(): PermissionCheck;

  // Safety
  isConcurrencySafe(): boolean; // default: false
  isReadOnly(): boolean; // default: false
  isDestructive(): boolean; // default: false

  // Loading
  shouldDefer: boolean;
  alwaysLoad: boolean;

  // Search
  searchHint: string; // 3-10 word capability phrase

  // Lifecycle
  isEnabled(): boolean;
}
```

### Tool Registry (Source: `src/tools.ts`)

Tools are assembled via `getAllBaseTools()` which is the single source of truth for all tools. The registry respects:

1. **Feature flags** — `feature()` from `bun:bundle` for compile-time elimination
2. **User type** — `process.env.USER_TYPE === 'ant'` for internal-only tools
3. **Environment** — `process.env.ENABLE_LSP_TOOL` etc.
4. **Mode settings** — worktree, REPL, coordinator mode

### Tool Pool Assembly

```
getAllBaseTools()                    Source of truth for all tools
    ↓
filterToolsByDenyRules()            Strip denied tools (Claude never sees them)
    ↓
getTools()                          Mode-aware filtering (simple/REPL/coordinator)
    ↓
assembleToolPool()                  Merge with MCP tools, deduplicate, sort for cache stability
```

**Cache stability:** Built-in tools form a contiguous sorted prefix, MCP tools are appended after. This prevents prompt cache invalidation when MCP tools change.

---

## System Prompt Architecture

**Source:** `src/utils/systemPrompt.ts` (4.8KB)

### 6-Layer Prompt Construction

| Priority    | Layer       | Description                         |
| ----------- | ----------- | ----------------------------------- |
| 1 (highest) | Override    | `--system-prompt` flag or loop mode |
| 2           | Coordinator | Dedicated coordinator system prompt |
| 3           | Agent       | Agent-specific prompt               |
| 4           | Custom      | CLAUDE.md files                     |
| 5           | Default     | Built-in system prompt              |
| 6 (lowest)  | Append      | Additional context                  |

### Caching Strategy

```typescript
// Memoized sections — cached until /clear or /compact
systemPromptSection();

// Volatile sections that BREAK prompt cache
DANGEROUS_uncachedSystemPromptSection();

// Dynamic boundary separating cacheable from per-session content
SYSTEM_PROMPT_DYNAMIC_BOUNDARY;
```

### System Prompt Components (10+)

1. Role definition
2. Tool schemas (all registered tools)
3. CLAUDE.md content (project instructions)
4. Memory files
5. Git context (branch, status)
6. OS/platform information
7. Working directory
8. Permission rules
9. Custom agent context
10. Feature-specific sections

---

## Ink Terminal Rendering Engine

**Source:** `src/ink/` directory

Claude Code uses a **full React reconciler for the terminal** — not just a text-based UI, but actual component-based rendering with flexbox layout.

### Stack

| Component             | Description                                                         |
| --------------------- | ------------------------------------------------------------------- |
| **React Reconciler**  | `reconciler.ts` — custom reconciler for component-based terminal UI |
| **Yoga Layout**       | Native TypeScript binding for flexbox in the terminal               |
| **Double Buffering**  | `render-to-screen.ts` — diff computation for minimal redraws        |
| **ANSI Optimization** | `StylePool`, `CharPool`, `HyperlinkPool` for string deduplication   |

### Terminal Features

- Kitty keyboard protocol support
- Mouse tracking
- iTerm2 progress indicators
- BIDI text support
- Hit testing
- Selection & search highlighting

### Export Capabilities

| Export     | Source         | Size  |
| ---------- | -------------- | ----- |
| ANSI → PNG | `ansiToPng.ts` | 214KB |
| ANSI → SVG | `ansiToSvg.ts` | 8KB   |

---

## MCP Integration

**Source:** `src/services/mcp/` (24 files, ~500KB)

### Transport Types (6)

1. **stdio** — subprocess-based communication
2. **SSE** — HTTP Server-Sent Events
3. **HTTP** — Standard HTTP requests
4. **WebSocket** — Full-duplex communication
5. **SDK** — Direct SDK integration
6. **CloudAI proxy** — Cloud-proxied connections

### Server Scopes (7)

| Scope        | Description           |
| ------------ | --------------------- |
| `local`      | Machine-specific      |
| `user`       | User-specific         |
| `project`    | Project-specific      |
| `dynamic`    | Runtime-added         |
| `enterprise` | Enterprise-managed    |
| `claudeai`   | Claude.ai integration |
| `managed`    | Platform-managed      |

### Connection Lifecycle

```
Configure → Connect → Capability Exchange → Tool Discovery
    ↓          ↓              ↓                    ↓
settings   transport    server info           tool schemas
  .json    negotiation  + capabilities       merged into pool
```

### Tool Wrapping

MCP tools are wrapped with a naming convention:

```
mcp__<server>__<tool>
Example: mcp__claude-in-chrome__tabs_context_mcp
```

Built-in tools take precedence on name conflicts. MCP tools are sorted for prompt cache stability.

---

## Permission System

**Source:** `src/utils/permissions/` (24 files), `src/types/permissions.ts`

### Permission Modes (6)

| Mode                | Behavior                                    |
| ------------------- | ------------------------------------------- |
| `default`           | Interactive prompting for risky operations  |
| `plan`              | Explicit approval required before execution |
| `bypassPermissions` | Full bypass (dangerous)                     |
| `dontAsk`           | Auto-deny everything                        |
| `acceptEdits`       | Auto-accept file edits only                 |
| `auto`              | ML classifier decides safety                |

### Permission Behaviors

- `allow` — permit the action
- `deny` — block the action
- `ask` — prompt the user
- `passthrough` — delegate to next layer

### Rule Sources (8)

Priority order:

1. `userSettings`
2. `projectSettings`
3. `localSettings`
4. `flagSettings`
5. `policySettings`
6. `cliArg`
7. `command`
8. `session`

### Bash Security

Two-stage classifier:

```
Stage 1: Fast static analysis (AST parsing)
    ↓
Stage 2: Thinking classifier (Claude evaluates safety)
    ↓
Risk Levels: LOW → MEDIUM → HIGH
```

The YOLO classifier (`classifyYoloAction()`) uses Claude itself to evaluate its own tool use safety in auto mode.

---

## Context Management

**Source:** `src/services/compact/autoCompact.ts`

### Token Budget Architecture

```
┌──────────────────────────────────────────────────────┐
│                200K Context Window                    │
│                                                      │
│ ┌──────┐┌────┐┌─────────────────┐┌──────────┐┌────┐│
│ │System││Mem ││  Conversation   ││  Tool    ││Free ││
│ │Prompt││ory ││  History        ││  Results ││    ││
│ │5-15K ││2-10││  (grows)        ││  (grows) ││    ││
│ └──────┘└────┘└─────────────────┘└──────────┘└────┘│
│                                                      │
│ ⚡ Auto-compact triggers at ~80% (187K tokens)       │
└──────────────────────────────────────────────────────┘

After compaction:
┌──────────────────────────────────────────────────────┐
│ ┌──────┐┌────┐┌───────┐┌──────┐┌──────────────────┐│
│ │System││Mem ││Summary││Recent││    ~60% freed     ││
│ │Prompt││ory ││       ││      ││                   ││
│ └──────┘└────┘└───────┘└──────┘└──────────────────┘│
└──────────────────────────────────────────────────────┘
```

### Key Constants

| Constant                               | Value  | Purpose                     |
| -------------------------------------- | ------ | --------------------------- |
| `AUTOCOMPACT_BUFFER_TOKENS`            | 13,000 | Buffer before context limit |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3      | Circuit breaker             |
| `WARNING_THRESHOLD_BUFFER_TOKENS`      | 20,000 | Early warning               |
| `MANUAL_COMPACT_BUFFER_TOKENS`         | 3,000  | Blocking limit              |

### MicroCompact

- **Scope:** Only Bash, FileRead, and Grep tool outputs
- **Behavior:** Incremental compression of individual tool results
- **Post-compact:** Restores up to 5 files (5K tokens each, 25K total)

---

## Multi-Agent Swarm System

**Source:** `src/utils/swarm/`, `src/tasks/InProcessTeammateTask/`

### Backend Types

| Backend              | Isolation           | Overhead                    |
| -------------------- | ------------------- | --------------------------- |
| **InProcessBackend** | `AsyncLocalStorage` | Zero (same Node.js process) |
| **TmuxBackend**      | Tmux panes/windows  | Process spawn               |
| **ITermBackend**     | iTerm2 split panes  | Process spawn               |

### Team Configuration

Teams are defined in `.claude/teams/<name>.json`:

```json
{
  "lead": "coordinator",
  "members": [
    {
      "name": "researcher",
      "model": "sonnet"
    },
    {
      "name": "implementer",
      "model": "opus"
    }
  ]
}
```

### Agent Identity & Restrictions

- Agents addressed as `agentName@teamName`
- **Coordinator-only tools:** `Agent`, `TaskStop`, `SendMessage`, `SyntheticOutput`
- **Agent restrictions:** Cannot use `TaskOutput`, `ExitPlanMode`, `AskUserQuestion`, `Workflow`

---

## Bridge & CCR System

**Source:** `src/bridge/` (31 files)

### Key Files

| File                  | Size  | Description                |
| --------------------- | ----- | -------------------------- |
| `bridgeMain.ts`       | 115KB | Main orchestrator          |
| `replBridge.ts`       | 100KB | REPL bridge implementation |
| `remoteBridgeCore.ts` | 39KB  | Remote bridge core         |
| `initReplBridge.ts`   | 23KB  | REPL bridge initialization |
| `sessionRunner.ts`    | 18KB  | Session runner             |
| `jwtUtils.ts`         | 9KB   | JWT management             |

### Bridge Architecture

```
┌─────────────┐       ┌──────────────┐       ┌─────────────┐
│   Client    │       │   Bridge     │       │  Local CLI  │
│ (Phone/Web) │◄─────►│   Server     │◄─────►│  (Claude)   │
└─────────────┘       └──────────────┘       └─────────────┘
                         │
                   ┌─────┴─────┐
                   │ poll →    │
                   │ WebSocket │
                   └───────────┘
```

### Security

| Feature           | Implementation                                       |
| ----------------- | ---------------------------------------------------- |
| Session token     | Read from `/run/ccr/session_token`                   |
| Ptrace protection | `prctl(PR_SET_DUMPABLE, 0)`                          |
| CA certificate    | Download + concatenate with system bundle            |
| Token cleanup     | File unlinked before agent loop — token only in heap |
| NO_PROXY          | Anthropic, GitHub, package registries, IMDS          |
| JWT refresh       | Every 3h55m (tokens expire in ~4hrs)                 |

---

## Memory System

**Source:** `src/memdir/memoryTypes.ts`, `src/services/extractMemories/`

### Memory Types

| Type        | Scope              | Default Visibility |
| ----------- | ------------------ | ------------------ |
| `user`      | Personal           | Always private     |
| `feedback`  | Behavioral         | Default private    |
| `project`   | Project context    | Bias toward team   |
| `reference` | Reference material | Usually team       |

### Storage Format

```markdown
---
name: project-conventions
description: Code style conventions for the project
type: project
---

# Project Conventions

- Use TypeScript strict mode
- Prefer functional patterns
- All files must have tests
```

### Team Memory

- Feature-gated behind `TEAMMEM`
- Shared across project team members
- Synced via `src/services/teamMemorySync/`

### Auto-Memory

- `~/.claude/MEMORY.md` — user-scoped
- Project-scoped `MEMORY.md` files
- Truncation by line and byte caps
- Background extraction of key insights

---

## Worktree Isolation

**Source:** `src/utils/worktree.ts` (49KB)

### Features

| Feature            | Implementation                                           |
| ------------------ | -------------------------------------------------------- |
| Creation           | `git worktree add`                                       |
| Symlinks           | `node_modules` linked from main repo                     |
| Sparse checkout    | `settings.worktree.sparsePaths`                          |
| Path protection    | Alphanumeric/dots/dashes/underscores only, ≤64 chars     |
| Session tracking   | `WorktreeSession` metadata with tmux name, branch, time  |
| Cache invalidation | Clears memory caches and system prompt sections on entry |
| Auto-cleanup       | `executeWorktreeRemoveHook()` on session end             |

---

## Migration System

**Source:** `src/migrations/`

11 active migrations, each logging `tengu_migrate_*` telemetry events:

| Migration                                    | Purpose                               |
| -------------------------------------------- | ------------------------------------- |
| `migrateFennecToOpus`                        | Remap retired Fennec codename to Opus |
| `migrateLegacyOpusToCurrent`                 | Update old Opus model IDs             |
| `migrateOpusToOpus1m`                        | Enable 1M context for Opus users      |
| `migrateSonnet45ToSonnet46`                  | Sonnet version bump                   |
| `migrateAutoUpdatesToSettings`               | Config migration                      |
| `migrateBypassPermissionsAcceptedToSettings` | Consent tracking                      |

---

## Server / DirectConnect System

**Source:** `src/server/directConnectManager.ts` (213 lines)

### Message Types

| Type                | Direction       | Description           |
| ------------------- | --------------- | --------------------- |
| `SDK messages`      | Bidirectional   | Standard SDK protocol |
| `Control requests`  | Server → Client | Permission checks     |
| `Control responses` | Client → Server | Permission results    |
| `Keep-alive`        | Bidirectional   | Connection health     |
| `Interrupt`         | Client → Server | Cancel operations     |

### Key Feature: `updatedInput`

The server supports "allow with modifications" — approving a tool call while changing its input parameters. This enables middleware-like behavior.

---

## Prompt Cache Optimization

**Source:** `src/services/api/promptCacheBreakDetection.ts`

### Two-Phase Detection

```
Pre-call:  recordPromptState()
    ↓
    Captures: system prompt, tools, model, betas,
              effort, cache scope/TTL

Post-call: checkResponseForCacheBreak()
    ↓
    Compares: cache_read tokens vs expected

Granular Detection:
    - System prompt char delta
    - Per-tool schema hash
    - Scope/TTL flips
    - Effort value changes
    - Extra body params
    - Beta header adds/removals
    - Auto-mode toggles
```

### Thresholds & Heuristics

| Parameter               | Value                                       |
| ----------------------- | ------------------------------------------- |
| `MIN_CACHE_MISS_TOKENS` | 2K (avoid false positives)                  |
| `MAX_TRACKED_SOURCES`   | 10 per agent                                |
| `>5min gap`             | Suggests TTL expiry                         |
| `<5min gap`             | Suggests server issue                       |
| Debug output            | Unified diffs to temp dir for Ant debugging |

---

_This document is AI-generated analysis of publicly available source code. Information may be inaccurate or outdated._
