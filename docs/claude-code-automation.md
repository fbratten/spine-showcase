# Claude Code Session Automation

> How to configure Claude Code for uninterrupted, automatic operation

This guide covers settings and hooks to eliminate permission prompts and maintain context across conversation compaction.

---

## Problem Statement

By default, Claude Code:
1. **Prompts for permission** on every tool use ("Yes, allow Claude to access...")
2. **Loses context** after conversation compaction (when token window fills)

Both issues interrupt autonomous workflows. Here's how to fix them.

---

## 1. Disable Permission Prompts

### Option A: Settings File (Permanent)

Add to `.claude/settings.local.json` (project) or `~/.claude/settings.json` (global):

```json
{
  "permissions": {
    "defaultMode": "bypassPermissions"
  }
}
```

This is the settings equivalent of `--dangerously-skip-permissions`.

### Option B: CLI Flag (Per Session)

```bash
claude --dangerously-skip-permissions
```

### Option C: Keyboard Shortcut (Quick Toggle)

Press **`Shift+Tab`** during a session to toggle auto-accept mode.

### Option D: Granular Permissions (Safer)

Pre-approve specific tools instead of bypassing all:

```json
{
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(python:*)",
      "Edit",
      "Write",
      "Read"
    ]
  }
}
```

---

## 2. Auto-Reload Context After Compaction

When conversations compact, Claude loses in-conversation context. Use **hooks** to automatically re-inject critical files.

### SessionStart Hook with Compact Matcher

Add to `.claude/settings.local.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo '=== POST-COMPACTION CONTEXT RELOAD ===' && echo '' && echo 'Please read: CLAUDE.md, NEXT.md, and the latest handover doc below:' && echo '' && LATEST=$(ls -t ai-memory/handover/*.md 2>/dev/null | head -1) && if [ -n \"$LATEST\" ]; then echo \"=== Latest Handover: $LATEST ===\"; echo ''; cat \"$LATEST\"; else echo 'No handover docs found in ai-memory/handover/'; fi"
          }
        ]
      }
    ]
  }
}
```

**How it works:**
- `"matcher": "compact"` - Only fires after compaction events
- The command finds the most recently modified `.md` file in `ai-memory/handover/`
- Output is injected into Claude's context

### Hook Matcher Options

| Matcher | Fires When |
|---------|------------|
| `startup` | First session start |
| `resume` | `/resume`, `--resume`, or `--continue` |
| `clear` | `/clear` command |
| `compact` | Auto-compaction or `/compact` |

---

## 3. CLAUDE.md Imports (Alternative)

For files that should ALWAYS be available (not just after compaction), use imports in `CLAUDE.md`:

```markdown
# Project Name

See @ai-memory/handover/latest-session.md
See @ai-memory/shared/project-context.md
See @NEXT.md

... rest of CLAUDE.md ...
```

The `@` syntax imports files when CLAUDE.md is read. These survive compaction automatically.

---

## 4. Complete Configuration Example

**File:** `.claude/settings.local.json`

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo '=== POST-COMPACTION ===' && LATEST=$(ls -t ai-memory/handover/*.md 2>/dev/null | head -1) && if [ -n \"$LATEST\" ]; then cat \"$LATEST\"; fi"
          }
        ]
      }
    ]
  },
  "permissions": {
    "defaultMode": "bypassPermissions",
    "allow": [
      "Bash(git:*)",
      "Bash(python:*)",
      "Edit",
      "Write",
      "Read"
    ]
  },
  "enableAllProjectMcpServers": true
}
```

---

## 5. What Survives Compaction

| Item | Survives? | Mechanism |
|------|-----------|-----------|
| **CLAUDE.md** | Yes | Reloaded every session |
| **CLAUDE.local.md** | Yes | Always reloaded |
| **Hooks** | Yes | Config persists in settings |
| **MCP servers** | Yes | .mcp.json is reloaded |
| **Conversation history** | No | Compacted/summarized |
| **In-conversation instructions** | No | Lost after compaction |

---

## 6. SPINE Integration

For SPINE-managed projects, combine automation with context persistence:

### Recommended ai-memory Structure

```
ai-memory/
├── shared/           # Cross-session shared context
│   ├── project-context.md
│   ├── architecture.md
│   └── decisions.md
├── handover/         # Session handover documents
│   └── session-YYYY-MM-DD-topic.md
├── architect/        # Role-specific context
├── implementer/
├── reviewer/
└── backlog/          # Future work tracking
```

### Post-Compaction Hook for SPINE Projects

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo '=== SPINE CONTEXT RELOAD ===' && echo 'Read: CLAUDE.md, NEXT.md' && echo '' && LATEST=$(ls -t ai-memory/handover/*.md 2>/dev/null | head -1) && if [ -n \"$LATEST\" ]; then echo \"Latest handover: $LATEST\"; cat \"$LATEST\"; fi"
          }
        ]
      }
    ]
  }
}
```

---

## 7. Safety Considerations

With `bypassPermissions` enabled:
- Claude runs uninterrupted without confirmation
- **Always commit to git** before running complex tasks
- Use `archived/` folders for deleted files (never hard-delete)
- Consider running in trusted projects only

---

## Quick Reference

| Goal | Setting |
|------|---------|
| No permission prompts | `"defaultMode": "bypassPermissions"` |
| Auto-reload after compact | `SessionStart` hook with `"matcher": "compact"` |
| Specific tool approval | `"allow": ["Edit", "Bash(git:*)"]` |
| Quick session toggle | Press `Shift+Tab` |

---

[← Back to Docs](README.md)
