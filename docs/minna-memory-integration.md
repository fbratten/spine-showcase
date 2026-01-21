# Minna Memory Integration (v0.3.22+)

SPINE provides an **optional integration** with **Minna Memory (mem-system-lite-mcp)** for persistent cross-session memory capabilities.

---

## Overview

Minna Memory enables SPINE to **remember** across sessions. Unlike ephemeral in-context memory, Minna provides persistent storage for decisions, patterns, errors, and preferences that survive session boundaries.

**Key principle:** SPINE continues to work normally if Minna is unavailable. The integration uses **graceful degradation**—every memory call has a sensible default (empty list, None) so workflows continue uninterrupted.

---

## What Minna Memory Provides

| Capability | Without Minna | With Minna |
|------------|---------------|------------|
| Entity tracking | No structured storage | Yes (people, projects, concepts) |
| Cross-session persistence | Ephemeral | Yes (SQLite) |
| Confidence decay | No | Yes (configurable half-life) |
| Conflict detection | No | Yes (`memory_detect_conflicts`) |
| Session summaries | Manual NEXT.md updates | Automatic storage |
| Full-text search | Limited | Yes (FTS5) |
| Preference tracking | Scattered configs | Structured (category/key/value) |
| Portable database | Scattered files | Single `.spine/minna.db` |

---

## Architecture: Per-Project Minna Instance

Each SPINE-managed project gets its **own isolated Minna database**:

```
SPINE (backbone)
    │
    ├── .spine/minna.db          ← SPINE's cross-project knowledge
    │
    ├── Project A
    │   ├── .spine/config.json   ← Contains Minna settings
    │   ├── .spine/minna.db      ← Project A's memories
    │   └── ai-coordination/
    │
    ├── Project B
    │   ├── .spine/config.json
    │   ├── .spine/minna.db      ← Project B's memories
    │   └── ai-coordination/
    │
    └── Project C (no Minna)
        └── .spine/config.json   ← minna.enabled: false
```

**Benefits:**

| Benefit | Why It Matters |
|---------|----------------|
| **Project Isolation** | Memories don't cross-contaminate between projects |
| **Portability** | `.spine/minna.db` travels with the project (SQLite) |
| **Optional** | Projects work fine without Minna configured |
| **Per-Machine** | Database is gitignored (`.spine/minna.db` not committed) |

---

## MCP Tools (16)

Minna exposes 16 MCP tools for Claude Code integration:

| Tool | Purpose |
|------|---------|
| `memory_store` | Store a memory about an entity |
| `memory_recall` | Retrieve memories with filters |
| `memory_search` | Full-text search across all memories |
| `memory_who` | Get comprehensive entity profile |
| `memory_add_entity` | Create a new entity (person, project, concept) |
| `memory_get_entity` | Get entity by name or alias |
| `memory_find_entities` | Search entities by type/query |
| `memory_relate` | Create relationship between entities |
| `memory_get_related` | Get related entities |
| `memory_set_preference` | Set/get preferences |
| `memory_get_preferences` | List all preferences |
| `memory_summarize_session` | Store conversation summary |
| `memory_get_context` | Get recent conversation context |
| `memory_stats` | Memory statistics |
| `memory_detect_conflicts` | Find contradicting memories |
| `memory_verify` | Reset confidence decay (re-confirm memory) |

---

## Entity Type Constraints

**Critical:** The MCP server only accepts 5 entity types:

| Type | Use For |
|------|---------|
| `person` | People, contacts, stakeholders |
| `project` | Project records |
| `concept` | **ADRs, errors, tasks, tools, patterns** (use naming prefixes) |
| `topic` | Topics, categories |
| `preference` | Stored preferences |

### Naming Conventions for CONCEPT Type

Use `concept` type with prefixed names for semantic distinction:

| Subtype | Naming Pattern | Example |
|---------|----------------|---------|
| ADRs | `adr:{project}:{id}` | `adr:spine:minna-mandatory` |
| Errors | `error:{project}:{hash}` | `error:spine:12345678` |
| Tasks | `task:{project}:{id}` | `task:spine:task-001` |
| Tools | `tool:{project}:{name}` | `tool:spine:grep` |
| Patterns | `pattern:{project}:{name}` | `pattern:spine:retry-on-failure` |
| Resolutions | `resolution:{project}:{hash}` | `resolution:spine:87654321` |

---

## Use Cases

### 1. Architectural Decision Records (ADRs)

**Problem:** Design decisions scattered across NEXT.md, ai-memory/, human memory.

**Minna Solution:**
```
# After making architectural decision:
memory_add_entity(name="adr:spine:auth-jwt", entity_type="concept")
memory_store(
    entity="adr:spine:auth-jwt",
    attribute="decision",
    value="Chose JWT over sessions for stateless API"
)
memory_store(
    entity="adr:spine:auth-jwt",
    attribute="rationale",
    value="Trade-offs: token size vs server load, chosen for horizontal scaling"
)
memory_store(
    entity="adr:spine:auth-jwt",
    attribute="alternatives",
    value="Sessions (rejected: requires sticky sessions), API keys (rejected: no user context)"
)

# Later, when implementing auth:
memory_search(query="authentication decision")
→ Returns ADR with full context
```

### 2. Cross-Session Error Resolution

**Problem:** Claude encounters the same error repeatedly across sessions.

**Minna Solution:**
```
# When error occurs:
error_hash = hash(error_output[:200])
memory_add_entity(name=f"error:spine:{error_hash}", entity_type="concept")
memory_store(entity=f"error:spine:{error_hash}", attribute="error_pattern", value=error_output)

# When resolved:
memory_store(entity=f"error:spine:{error_hash}", attribute="resolution", value="Fixed by adding retry logic to API calls")

# Future session encounters same error:
prior = memory_search(query=f"error:{error_hash}")
if prior:
    resolution = memory_recall(entity=prior[0].entity, attribute="resolution")
    → "Previously resolved by: Fixed by adding retry logic to API calls"
```

### 3. Tool Learning

**Problem:** Claude doesn't know which tools work best for which task types.

**Minna Solution:**
```
# After successful tool use:
memory_store(
    entity="tool:spine:ripgrep",
    attribute="tool_success",
    value="code_search tasks - fast and accurate",
    context="Used for finding function definitions"
)

# After tool failure:
memory_store(
    entity="tool:spine:find",
    attribute="tool_failure",
    value="large codebases - times out",
    context="Better to use ripgrep or glob patterns"
)

# Before selecting tool:
memory_search(query="code_search tool")
→ Returns tool preference history
```

### 4. Stakeholder Context

**Problem:** Claude needs to re-learn stakeholder preferences each session.

**Minna Solution:**
```
# When learning stakeholder preferences:
memory_add_entity(name="Jack", entity_type="person")
memory_store(entity="Jack", attribute="prefers", value="dark mode code blocks")
memory_store(entity="Jack", attribute="communication", value="concise, no emojis")
memory_store(entity="Jack", attribute="coding_style", value="snake_case, 100 char lines")

# Any session:
memory_who("Jack")
→ Returns complete profile with all preferences
```

### 5. Session Continuity

**Problem:** Context lost when session compacts or ends.

**Minna Solution:**
```
# At session end:
memory_summarize_session(
    session_id="session-2026-01-18",
    summary="Completed Minna integration for SPINE. Added 16 MCP tools, created showcase docs.",
    topics=["minna-memory", "spine-showcase", "documentation"]
)

# Next session:
memory_get_context(limit=5)
→ Returns recent session summaries for continuity
```

---

## Confidence Decay

Minna implements a **half-life model** for memory confidence:

```
Confidence = base_confidence × 2^(-days_since_verified / half_life)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `half_life_days` | 90 | Days for confidence to halve |
| `base_confidence` | 1.0 | Initial confidence (0.0-1.0) |

**Why It Matters:**
- Old memories fade naturally (like human memory)
- Re-verified memories reset their decay timer
- Low-confidence memories can be filtered out
- Prevents stale information from dominating

```
# Recall with minimum confidence threshold:
memory_recall(entity="Jack", min_confidence=0.5)
→ Returns only memories with effective_confidence >= 0.5

# Re-verify a memory (resets decay):
memory_verify(memory_id=123)
→ Memory's verified_at updated to now
```

---

## Conflict Detection

Minna can detect contradicting facts:

```
# Store conflicting information:
memory_store(entity="Jack", attribute="timezone", value="UTC-5")
memory_store(entity="Jack", attribute="timezone", value="UTC+1")

# Detect conflicts:
memory_detect_conflicts(entity="Jack")
→ Returns:
{
  "conflicts": [{
    "attribute": "timezone",
    "values": ["UTC-5", "UTC+1"],
    "memory_ids": [45, 67]
  }]
}
```

---

## Integration with SPINE

### Context Stack Enrichment

Before subagent execution, Minna can enrich context:

```python
# In context stack building:
if minna_available:
    # Get relevant ADRs
    adrs = minna.memory_search(f"adr:{project_id}")

    # Get code style preferences
    prefs = minna.get_preferences(category="code_style")

    # Get stakeholder preferences
    stakeholder = minna.memory_who("Jack")

    # Inject into context_stack.context.background
    context_stack.context.background += f"\n\nADRs: {adrs}\nPreferences: {prefs}"
```

### Oscillation Pattern Storage

When OscillationTracker detects stuck states:

```python
# Store oscillation pattern for future reference
minna.memory_store(
    entity=f"pattern:{project_id}:oscillation",
    attribute="anti_pattern",
    value=oscillation.error_signature,
    context=oscillation.error_output[:500]
)

# Future session can check:
prior_oscillations = minna.memory_search(f"oscillation {error_signature}")
if prior_oscillations:
    resolution = minna.memory_recall(
        entity=prior_oscillations[0].entity,
        attribute="resolution"
    )
```

### Session Handover

At session end:

```python
minna.memory_summarize_session(
    session_id=f"session-{date.today()}",
    summary=summary_text,
    topics=["feature-x", "bug-fix-y"],
    entity_ids=[project_entity_id, stakeholder_id]
)
```

---

## Configuration

### 1. Add to .mcp.json

```json
{
  "mcpServers": {
    "minna-memory": {
      "command": "bash",
      "args": ["-c", "cd /path/to/mem-system-lite-mcp && .venv/bin/python -m mem_system_lite.server"],
      "env": {
        "MINNA_DB_PATH": "/path/to/project/.spine/minna.db"
      }
    }
  }
}
```

### 2. Create .spine/config.json

```json
{
  "project_id": "my-project",
  "project_label": "PROJ-MY-PROJECT",

  "minna": {
    "enabled": true,
    "database": ".spine/minna.db",
    "auto_summarize": true,

    "entity_types_available": [
      "person", "project", "concept", "topic", "preference"
    ],

    "naming_conventions": {
      "adrs": "adr:{project}:{id}",
      "errors": "error:{project}:{hash}",
      "tasks": "task:{project}:{id}",
      "tools": "tool:{project}:{name}",
      "patterns": "pattern:{project}:{name}"
    },

    "memory_attributes": [
      "decision", "rationale", "alternatives", "trade_offs",
      "task_outcome", "error_pattern", "resolution",
      "tool_success", "tool_failure"
    ],

    "context_enrichment": {
      "enabled": true,
      "include_decisions": true,
      "include_patterns": true,
      "include_error_resolutions": true,
      "max_memories_per_category": 10
    }
  }
}
```

---

## Graceful Degradation

Every Minna call has a sensible default:

```python
class MinnaIntegration:
    def is_available(self):
        try:
            return self._health_check()
        except:
            return False

    def search_memories(self, query, **kwargs):
        if not self.is_available():
            return []  # Graceful no-op
        return self._client.memory_search(query, **kwargs)

    def store_memory(self, entity, attribute, value, **kwargs):
        if not self.is_available():
            return None  # Graceful no-op
        return self._client.memory_store(entity, attribute, value, **kwargs)
```

| Scenario | Behavior |
|----------|----------|
| Minna MCP server not running | Returns empty results, SPINE continues |
| Database file doesn't exist | Created automatically on first use |
| Network/connection error | Graceful fallback, warning logged |
| Tool call fails | Returns None/empty, workflow continues |

---

## Benefits for SPINE-Like Systems

### Why Persistent Memory Matters

| Benefit | Impact |
|---------|--------|
| **Session Continuity** | No more "where were we?" at session start |
| **Reduced Re-Discovery** | Don't re-learn stakeholder preferences every time |
| **Pattern Recognition** | Detect recurring errors across sessions |
| **Decision Traceability** | Know WHY past decisions were made |
| **Preference Consistency** | Code style, communication style persisted |

### For Multi-Agent Orchestrators

| Benefit | How Minna Helps |
|---------|-----------------|
| **Subagent Context** | Enrich prompts with relevant memories before execution |
| **Error Recovery** | Look up past resolutions before attempting fixes |
| **Tool Selection** | Learn which tools work best for which tasks |
| **Conflict Resolution** | Detect contradicting information early |

### Compared to Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **In-context memory** | Fast, no setup | Lost on compaction/session end |
| **File-based (ai-memory/)** | Simple, git-tracked | Unstructured, hard to search |
| **Vector store** | Semantic search | Complex setup, costly |
| **Minna** | Structured, searchable, portable | Requires MCP server setup |

---

## Fringe Use Cases

Beyond the standard patterns, Minna can be stretched for creative applications:

### 1. Multi-Project Knowledge Transfer

**Scenario:** Learning from one project applies to another.

```
# Project A discovers a useful pattern
memory_store(entity="pattern:projectA:retry-backoff", attribute="pattern",
             value="Exponential backoff with jitter for API calls")

# Project B (different Minna DB) could import this
# Manual: Export from A, import to B
# Or: Use a shared "patterns" database path
```

**Fringe aspect:** Minna is per-project by design, but nothing stops you from pointing multiple projects at a shared `patterns.db` for cross-project learning.

### 2. Prompt Engineering Memory

**Scenario:** Remember which prompt phrasings work best.

```
memory_add_entity(name="prompt:code-review", entity_type="concept")
memory_store(entity="prompt:code-review", attribute="effective",
             value="Focus on security vulnerabilities, not style nitpicks")
memory_store(entity="prompt:code-review", attribute="ineffective",
             value="Generic 'review this code' produces low-quality output")
```

**Fringe aspect:** Using Minna to evolve prompts over time rather than just storing facts.

### 3. Debug Session Continuity

**Scenario:** Multi-day debugging where context is critical.

```
# Day 1: Initial investigation
memory_store(entity="debug:issue-123", attribute="hypothesis",
             value="Memory leak in connection pool - connections not released")
memory_store(entity="debug:issue-123", attribute="ruled_out",
             value="Not garbage collection - heap dumps normal")

# Day 2: Continue where you left off
profile = memory_who("debug:issue-123")
# Returns full investigation history
```

**Fringe aspect:** Treating a debug session as an entity that accumulates knowledge.

### 4. Code Review Persona Memory

**Scenario:** Remember reviewer preferences per codebase.

```
memory_store(entity="reviewer:security-bot", attribute="focus",
             value="SQL injection, XSS, auth bypass - ignore style")
memory_store(entity="reviewer:perf-bot", attribute="focus",
             value="N+1 queries, missing indexes, unbounded loops")

# Before review, load persona preferences
persona = memory_who("reviewer:security-bot")
```

**Fringe aspect:** Personas that evolve based on feedback rather than static prompts.

### 5. Codebase Archaeology

**Scenario:** Document tribal knowledge as you discover it.

```
# While exploring legacy code
memory_store(entity="legacy:payment-module", attribute="warning",
             value="Do NOT modify calculateTax() - breaks 3 downstream systems")
memory_store(entity="legacy:payment-module", attribute="history",
             value="Originally written for EU only, US bolted on in 2019")

# Future sessions get this context automatically
```

**Fringe aspect:** Building institutional knowledge that persists beyond any single developer.

### 6. A/B Testing Decisions

**Scenario:** Track which approaches were tried and their outcomes.

```
memory_store(entity="experiment:auth-flow", attribute="approach_a",
             value="OAuth2 with PKCE - worked but complex setup")
memory_store(entity="experiment:auth-flow", attribute="approach_b",
             value="Magic links - simpler but email deliverability issues")
memory_store(entity="experiment:auth-flow", attribute="chosen",
             value="Hybrid: Magic links for signup, OAuth for returning users")
```

**Fringe aspect:** Decision journaling that captures the exploration process, not just the outcome.

### 7. Inter-Agent Communication

**Scenario:** Agents leave messages for future agents.

```
# Architect agent leaves note
memory_store(entity="handoff:task-123", attribute="for_implementer",
             value="I chose microservices over monolith - see adr:arch-001 for rationale")

# Implementer agent picks up
notes = memory_recall(entity="handoff:task-123", attribute="for_implementer")
```

**Fringe aspect:** Asynchronous agent-to-agent communication via persistent memory.

---

## Edge Cases and Limitations

### 1. Entity Type Constraints

**Only 5 types allowed:** person, project, concept, topic, preference

**Workaround:** Use `concept` with naming prefixes (see naming conventions above)

### 2. Database Per-Machine

`.spine/minna.db` is gitignored (binary file, per-machine state).

**Implication:** Each developer/machine has their own memory store.

**If shared memory needed:** Consider a shared database path or export/import workflows.

### 3. No Automatic Cleanup

Memories persist indefinitely. Use confidence decay + filtering to surface relevant memories.

### 4. Single-User Assumption

Minna assumes single-user access to the database. No built-in locking for concurrent access.

### 5. Search Limitations

FTS5 search is text-based, not semantic. Use specific keywords for best results.

---

## Quick Reference

### Most Common Operations

```
# Store a decision
memory_add_entity(name="adr:project:decision-id", entity_type="concept")
memory_store(entity="adr:project:decision-id", attribute="decision", value="...")
memory_store(entity="adr:project:decision-id", attribute="rationale", value="...")

# Find past decisions
memory_search(query="authentication decision")

# Store error resolution
memory_store(entity="error:project:hash", attribute="resolution", value="...")

# Get stakeholder profile
memory_who("Jack")

# Set preference
memory_set_preference(category="code_style", key="line_length", value="100")

# Session summary
memory_summarize_session(session_id="...", summary="...")
```

---

## Related Documentation

| Resource | Description |
|----------|-------------|
| [SPINE Integration Strategy](https://github.com/fbratten/spine) | Strategy doc in `ai-memory/integrations/` |
| [spine-integrations](https://github.com/fbratten/spine-integrations) | Implementation package [PROJ-SOLAR-SPARK] |
| [mem-system-lite-mcp](https://github.com/fbratten/mem-system-lite-mcp) | Core Minna library |
| [MCP Orchestrator](mcp-orchestrator-integration.md) | Another optional integration |

---

## Summary

Minna Memory transforms SPINE from a stateless orchestrator into a **learning system** that:

- **Remembers** decisions, errors, and patterns across sessions
- **Learns** which approaches work best over time
- **Maintains** stakeholder context without re-discovery
- **Detects** contradictions and recurring issues

All with **graceful degradation**—SPINE works identically with or without Minna.

---

[← Back to Docs](README.md) | [MCP Orchestrator →](mcp-orchestrator-integration.md)
