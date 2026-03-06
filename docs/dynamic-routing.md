# Dynamic Routing by Task Type (v0.3.26)

SPINE can automatically classify tasks and route them to the most appropriate executor based on task type.

---

## Overview

Instead of always using a single executor, Dynamic Routing analyzes each task and selects the best executor for it. Research tasks go to a research-optimized executor, code tasks go to a code-focused executor, etc.

<div style="font-family: 'Inter', system-ui, sans-serif; background: #0f172a; border-radius: 12px; padding: 24px; max-width: 620px; color: #e2e8f0; border: 1px solid #1e293b;">
  <div style="text-align: center; font-weight: 700; font-size: 1.1em; color: #94a3b8; margin-bottom: 16px; letter-spacing: 0.05em;">AgenticLoop</div>
  <div style="display: flex; align-items: center; justify-content: center; gap: 10px; margin-bottom: 16px;">
    <div style="padding: 10px 18px; background: #2563eb; border-radius: 8px; font-weight: 600; box-shadow: 0 0 12px rgba(37,99,235,0.4);">TaskQueue</div>
    <div style="font-size: 1.3em; color: #64748b;">&#10140;</div>
    <div style="padding: 10px 18px; background: #7c3aed; border-radius: 8px; font-weight: 600; box-shadow: 0 0 12px rgba(124,58,237,0.4);">TaskTypeRouter</div>
    <div style="font-size: 1.3em; color: #64748b;">&#10140;</div>
    <div style="padding: 10px 18px; background: #0d9488; border-radius: 8px; font-weight: 600; box-shadow: 0 0 12px rgba(13,148,136,0.4);">Evaluator</div>
  </div>
  <div style="text-align: center; font-size: 1.2em; color: #64748b; margin-bottom: 12px;">&#9661; &nbsp; &#9661; &nbsp; &#9661;</div>
  <div style="display: flex; justify-content: center; gap: 14px;">
    <div style="flex: 1; text-align: center; padding: 12px 10px; background: #1e293b; border-radius: 8px; border: 1px solid #334155;">Subagent<br>Executor</div>
    <div style="flex: 1; text-align: center; padding: 12px 10px; background: #1e293b; border-radius: 8px; border: 1px solid #334155;">ClaudeCode<br>Executor</div>
    <div style="flex: 1; text-align: center; padding: 12px 10px; background: #1e293b; border-radius: 8px; border: 1px solid #334155;">SmallLLM<br>Executor</div>
  </div>
</div>

---

## Task Types

SPINE classifies tasks into six types:

| TaskType | Description | Example Tasks |
|----------|-------------|---------------|
| `CODE` | Code writing, bug fixes, refactoring | "Fix the authentication bug", "Add unit tests" |
| `RESEARCH` | Information gathering, analysis | "Investigate performance bottleneck" |
| `CONTENT` | Document writing, content generation | "Write API documentation" |
| `REVIEW` | Code review, quality checks | "Review the PR for security issues" |
| `ANALYSIS` | Data analysis, metrics, reporting | "Analyze test coverage trends" |
| `GENERAL` | Fallback for unclassified tasks | "Help me with this" |

## Classification Strategy

`classify_task_type()` uses a priority chain:

1. **Metadata override** — `task.metadata["task_type"]` if present
2. **Tag matching** — Task tags mapped to types (e.g., `"code"` tag → `CODE`)
3. **Keyword heuristics** — Task description scanned for type-specific keywords
4. **Fallback** — Returns `GENERAL`

---

## TaskTypeRouter

The `TaskTypeRouter` implements the `Executor` interface, making it transparent to the `AgenticLoop`. No changes to the loop are needed.

```python
from spine.orchestrator.task_router import TaskTypeRouter, TaskType, RoutingRule
from spine.orchestrator.executors import SubagentExecutor, ClaudeCodeExecutor

# Create routing rules
rules = [
    RoutingRule(TaskType.CODE, SubagentExecutor(code_config)),
    RoutingRule(TaskType.RESEARCH, ClaudeCodeExecutor(research_config)),
    RoutingRule(TaskType.CONTENT, SubagentExecutor(content_config)),
]

# Create router (with fallback executor for unmatched types)
router = TaskTypeRouter(
    rules=rules,
    fallback=SubagentExecutor(default_config),
)

# Router.execute() auto-classifies and delegates
result = router.execute(task, project_path, role="implementer")
```

## CLI Usage

```bash
# Use dynamic routing
python -m spine.orchestrator run --project /path \
    --executor router \
    --route CODE:subagent \
    --route RESEARCH:claude-code \
    --route CONTENT:subagent

# Classify a task without executing
python -m spine.orchestrator classify --project /path --task-id TASK-001
```

## Routing Callbacks

```python
from spine.orchestrator.routing_callbacks import create_routing_callbacks

callbacks = create_routing_callbacks()
# Returns: on_task_start (logging) + on_pre_execute (metadata injection)
```

---

## Key Design Decisions

- **Router implements Executor** — Zero changes to AgenticLoop or any existing code
- **Heuristic classification** — No LLM call needed for routing (fast, free)
- **Composable** — Router can wrap any executor, including SmallLLMExecutor
- **Config-driven** — Routing rules in `.spine/config.json` for zero-flag usage

---

[← Back to Docs](README.md) | [Executor Framework →](executors.md)
