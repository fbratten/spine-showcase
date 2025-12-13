# Tiered Enforcement Protocol

Not every task needs the full multi-agent orchestra. This protocol helps match task complexity to the right tooling - balancing capability against overhead (tokens, latency, complexity).

**Version:** 1.0.0
**Status:** Mandatory for SPINE-orchestrated projects

---

## The Three Tiers

| Tier | Task Type | Enforcement | Examples |
|------|-----------|-------------|----------|
| **Tier 1** | Simple | None needed | Fix typo, add docstring |
| **Tier 2** | Medium | Recommended | New API endpoint, bug investigation |
| **Tier 3** | Complex | Mandatory | System design, dashboard implementation |

---

## Tier 1: Simple Tasks

Single-file edits, typo fixes, small bug fixes, adding comments, simple refactoring within one file, quick questions about code.

**What to do:** Just do it. No subagents, no MCP, no documentation required.

**Examples:**
- "Fix the typo in README.md"
- "Add a docstring to this function"
- "Rename this variable"
- "What does this function do?"

**Cost:** ~500-2,000 tokens, 2-5 seconds, 1 API call

---

## Tier 2: Medium Tasks

Multi-file changes (2-5 files), new features with clear scope, bug fixes that need investigation, refactoring across files, integration work.

**What to do:**
- Use `Task(subagent_type="Explore")` for codebase exploration
- Use `browser-mcp` for UI changes
- Use `next-conductor` for task status
- If you skip any of these, document why

**Examples:**
- "Add a new API endpoint with tests"
- "Implement dark mode toggle"
- "Fix the authentication bug"
- "Refactor the utils module"

**Skip template:**
```
SPINE Usage (Tier 2):
- Subagents: [Used/Skipped - reason]
- browser-mcp: [Used/Skipped - reason]
- next-conductor: [Used/Skipped - reason]
```

**Cost:** ~5,000-15,000 tokens, 30-90 seconds, 2-5 API calls

---

## Tier 3: Complex Tasks

Architecture decisions, research tasks, UI-heavy implementations, multi-component features (6+ files), system design, performance optimization.

**What to do:**
- Use the right subagents: `code-architect` for design, `research-coordinator` for research, `visual-tester` + `browser-mcp` for UI
- Take screenshots for visual verification
- Document capability usage
- Use SPINE Python logging when it makes sense

**Examples:**
- "Design the authentication system architecture"
- "Research caching best practices for this stack"
- "Implement the dashboard with charts and filters"
- "Optimize database query performance"

**Documentation template:**
```
SPINE Usage (Tier 3):
- Subagents used:
  - [subagent_type]: [purpose]
- MCP servers used:
  - [server]: [purpose]
- Screenshots: [location]
- SPINE logs: [experiment ID]
```

**Cost:** ~20,000-100,000+ tokens, 2-10 minutes, 5-20+ API calls

---

## Why Bother with Tiers?

### The cost of over-engineering

| Factor | Impact |
|--------|--------|
| Token cost | Parallel subagents = 2-6x cost increase |
| Latency | Subagent spawn adds 10-30 seconds |
| Over-engineering | Simple tasks don't need orchestration |
| Context fragmentation | Subagents don't share full conversation context |
| Debugging | More layers = harder to trace failures |

### The cost of under-engineering

| Factor | Impact |
|--------|--------|
| Missed requirements | Incomplete exploration |
| Rework | Architectural mistakes caught late |
| Quality issues | Insufficient verification |
| Technical debt | Quick fixes that don't scale |

---

## Decision Flow

```
New Task
    │
    ├─ 1 file? ──────────────────► TIER 1
    │
    │ 2-5 files
    ▼
    ├─ Design decisions needed? ─► TIER 3 (if yes)
    │
    │ No
    ▼
    ├─ Research/UI-heavy/arch? ──► TIER 3 (if yes)
    │
    │ No
    ▼
    TIER 2
```

---

## Try It Out

The [Tier Classifier Demo](../demos/tier-classifier/) walks through the classification with four questions:

1. **Scope** - how many files/systems affected?
2. **Clarity** - how clear are the requirements?
3. **Decisions** - what design decisions needed?
4. **Verification** - what verification required?

---

## Checklists

### Before starting (all tiers)
- [ ] Classify task tier
- [ ] If Tier 2/3: identify which capabilities apply

### Tier 2 completion
- [ ] If subagents skipped: documented reason
- [ ] If browser-mcp skipped for UI: documented reason

### Tier 3 completion
- [ ] Appropriate subagents used
- [ ] browser-mcp used for UI verification
- [ ] Screenshots captured
- [ ] Capability usage documented

---

## Examples: Right vs Wrong

### UI Feature

**Task:** "Add a settings page with theme toggle"

**Wrong:**
```
- Created settings.tsx
- Added theme toggle
- Done
```

**Right:**
```
- Used Explore to find existing theme patterns
- Created settings.tsx
- Used browser-mcp to navigate and screenshot
- Used visual-tester to verify rendering

SPINE Usage (Tier 3):
- Subagents: Explore (found ThemeContext), visual-tester (verified UI)
- MCP: browser-mcp (screenshots)
- Screenshots: screenshots/settings-theme-toggle.png
```

### Research Task

**Task:** "Research caching strategies for our API"

**Wrong:**
```
- Read some articles
- Recommended Redis
- Done
```

**Right:**
```
- Used research-coordinator to investigate:
  - Current caching patterns in codebase
  - Industry best practices
  - Framework-specific recommendations
- Synthesized findings with trade-offs

SPINE Usage (Tier 3):
- Subagents: research-coordinator, code-architect
- MCP: research-agent-mcp
```

### Simple Bug Fix

**Task:** "Fix null pointer in getUserName()"

**Fine:**
```
- Found the issue: missing null check
- Added null check with fallback
- Done
```

No extra documentation needed for Tier 1.

---

## How Enforcement Works

This is instruction-based. Claude reads the protocol, classifies tasks, follows requirements, and documents usage. There's no automated gate - that may come later.

---

## Related Docs

- [Architecture Overview](architecture.md) - system design and components
- [Pattern Guide](patterns.md) - fan-out and pipeline usage

[← Back to Main](../README.md)
