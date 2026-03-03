# Dynamic Routing by Task Type (v0.3.26)

SPINE can automatically classify tasks and route them to the most appropriate executor based on task type.

---

## Overview

Instead of always using a single executor, Dynamic Routing analyzes each task and selects the best executor for it. Research tasks go to a research-optimized executor, code tasks go to a code-focused executor, etc.

```
┌─────────────────────────────────────────────────────────┐
│                     AgenticLoop                          │
│                                                          │
│  ┌──────────┐    ┌──────────────────┐    ┌───────────┐  │
│  │ TaskQueue │───▶│ TaskTypeRouter   │───▶│ Evaluator │  │
│  └──────────┘    └────────┬─────────┘    └───────────┘  │
│                           │                              │
│        ┌──────────────────┼──────────────────┐          │
│        ▼                  ▼                  ▼          │
│   ┌─────────┐      ┌──────────┐      ┌──────────┐     │
│   │Subagent │      │ClaudeCode│      │SmallLLM  │     │
│   │Executor │      │Executor  │      │Executor  │     │
│   └─────────┘      └──────────┘      └──────────┘     │
└─────────────────────────────────────────────────────────┘
```

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
