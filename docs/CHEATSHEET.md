# 🚀 Claude Code — Quick Reference Cheat Sheet

> A fast-lookup reference for Claude Code's hidden features, flags, and internals.

---

## 🔑 Quick Access

### Secret CLI Flags

```bash
claude --bare                    # No hooks/plugins/memory
claude --dump-system-prompt      # Print system prompt & exit
claude --bg "fix the tests"      # Background tmux session
claude --worktree                # Git worktree isolation
claude --capacity 4              # Parallel worker limit
```

### Useful Environment Variables

```bash
# Debug
export CLAUDE_CODE_PERFETTO_TRACE=1        # Chrome trace
export CLAUDE_CODE_PROFILE_STARTUP=1       # Startup timing
export CLAUDE_CODE_DEBUG_REPAINTS=1         # UI repaint visualization

# Overrides
export CLAUDE_CODE_MAX_CONTEXT_TOKENS=1000000  # 1M context
export MAX_THINKING_TOKENS=50000               # Thinking budget
export IDLE_THRESHOLD_MINUTES=120              # Idle timeout

# Fast mode
export CLAUDE_CODE_DISABLE_FAST_MODE=1     # Disable fast mode

# Internal (use with caution)
export USER_TYPE=ant                        # Unlock internal features
```

### Hidden Slash Commands

```
/btw          → Quick side question
/ctx-viz      → Context visualizer
/good-claude  → Easter egg 🐣
/dream        → Memory consolidation
/ultraplan    → 30-min advanced planning
/env          → Environment inspect
/version      → Internal version info
/force-snip   → Force snippet compression
/break-cache  → Invalidate prompt cache
```

---

## 📊 Model Cheat Sheet

| Alias      | Resolves To                        | Who Gets It                |
| ---------- | ---------------------------------- | -------------------------- |
| `best`     | Opus 4.6                           | All users                  |
| `opus`     | Latest Opus                        | Explicit request           |
| `sonnet`   | Latest Sonnet                      | Default for standard users |
| `haiku`    | Latest Haiku                       | Explicit request           |
| `opusplan` | Opus (plan mode) / Sonnet (normal) | Auto                       |

### Default Models by Tier

| Tier            | Default       |
| --------------- | ------------- |
| Max/Team        | Opus 4.6      |
| Standard        | Sonnet 4.6    |
| Anthropic (Ant) | Opus 4.6 [1m] |

---

## ⚡ Fast Mode Quick Facts

- **Model:** Opus 4.6 only
- **Pricing:** 6× normal ($30/$150 vs $5/$25 per MTok)
- **1P only:** Not available on Bedrock/Vertex/Foundry
- **Default:** ON for paid users
- **Internal name:** "Penguin Mode" 🐧
- **Disable:** `CLAUDE_CODE_DISABLE_FAST_MODE=1`

---

## 🧠 Context Window

| Setting              | Tokens                 |
| -------------------- | ---------------------- |
| Default              | 200K                   |
| Extended             | 1M (via `[1m]` suffix) |
| Auto-compact trigger | ~187K (80%)            |
| Post-compact freed   | ~60%                   |
| Buffer               | 13K tokens             |
| Warning              | 20K before limit       |

---

## 🔒 Permission Rules (settings.json)

```json
{
  "permissions": {
    "allow": ["Bash(git *)", "Bash(npm *)", "FileRead", "FileEdit"],
    "deny": ["Bash(rm *)", "Bash(sudo *)", "FileWrite(/etc/*)"]
  }
}
```

---

## 🔌 MCP Server Config

```json
{
  "mcpServers": {
    "my-server": {
      "command": "npx",
      "args": ["-y", "@my/mcp-server"]
    },
    "remote-api": {
      "url": "https://api.example.com/mcp",
      "transport": "sse"
    }
  }
}
```

---

## 📁 CLAUDE.md Priority (low → high)

```
1. /etc/claude-code/CLAUDE.md     (system)
2. ~/.claude/CLAUDE.md            (user)
3. ./CLAUDE.md                    (project)
4. ./.claude/CLAUDE.md            (project alt)
5. ./.claude/rules/*.md           (rules)
6. ./CLAUDE.local.md              (local override)
```

---

## 🪝 Hooks Quick Reference

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "echo 'Bash about to run'"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "FileEdit",
        "command": "eslint --fix $CLAUDE_FILE"
      }
    ]
  }
}
```

**Hook Events:** Setup, SessionStart, SessionEnd, PreToolUse, PostToolUse, PreCompact, PostCompact, SubagentStart, SubagentStop, UserPromptSubmit, PermissionRequest, PermissionDenied, and more.

---

## 🌱 GrowthBook Feature Gates

| Gate                    | What it controls     |
| ----------------------- | -------------------- |
| `tengu_turtle_carbon`   | UltraThink           |
| `tengu_kairos`          | Persistent assistant |
| `tengu_cobalt_raccoon`  | Auto-compact         |
| `tengu_scratch`         | Worker scratch dirs  |
| `tengu_harbor`          | MCP allowlist        |
| `tengu_ultraplan_model` | UltraPlan model      |
| `tengu_onyx_plover`     | Auto-dream           |

---

## 🗂️ Key File Locations

| Path                      | Purpose                 |
| ------------------------- | ----------------------- |
| `~/.claude/settings.json` | User settings           |
| `~/.claude/auth.json`     | OAuth tokens            |
| `~/.claude/CLAUDE.md`     | User-level instructions |
| `~/.claude/MEMORY.md`     | User memory             |
| `~/.claude/agents/`       | Custom agents           |
| `~/.claude/skills/`       | Custom skills           |
| `~/.claude/teams/`        | Team configs            |
| `.claude/config.json`     | Project config          |
| `.claude/settings.json`   | Project settings        |
| `/tmp/claude-{uid}/`      | Session scratchpad      |

---

_Quick reference derived from Claude Code source analysis · March 2026_
