# Executor Framework (v0.3.19+)

SPINE's orchestrator uses a **pluggable executor architecture** that separates task execution logic from the agentic loop. This allows swapping execution strategies without changing the core orchestration flow.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      AgenticLoop                            │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ TaskQueue   │───▶│  Executor   │───▶│  Evaluator  │     │
│  └─────────────┘    └──────┬──────┘    └─────────────┘     │
│                            │                                │
│        ┌───────────────────┼───────────────────┐           │
│        ▼                   ▼                   ▼           │
│  ┌────────────┐     ┌────────────┐     ┌────────────┐     │
│  │ Subagent   │     │ClaudeCode  │     │    MCP     │     │
│  │ Executor   │     │ Executor   │     │Orchestrator│     │
│  └────────────┘     └────────────┘     └────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

---

## Available Executors

### SubagentExecutor

Uses `.claude/agents/` persona definitions to execute tasks via the LLM client.

```python
from spine.orchestrator.executors import SubagentExecutor, SubagentConfig

config = SubagentConfig(
    agent_dir=project_path / ".claude" / "agents",
    model_override="claude-opus-4-5-20251101",
    use_context_stacks=True,
)
executor = SubagentExecutor(config)
result = executor.execute(task, project_path, role="implementer")
```

**Features:**
- Reads agent personas from `.claude/agents/*.md`
- Supports context stacks from YAML scenarios
- Role-based persona selection (architect, implementer, reviewer)
- Model override capability

**CLI Usage:**
```bash
python -m spine.orchestrator run --project /path \
    --executor subagent \
    --executor-model claude-opus-4-5-20251101
```

---

### ClaudeCodeExecutor

Spawns Claude Code CLI as a subprocess for task execution.

```python
from spine.orchestrator.executors import ClaudeCodeExecutor, ClaudeCodeConfig

config = ClaudeCodeConfig(
    model="opus",
    max_budget_usd=5.0,
    skip_permissions=False,
    use_context_stacks=True,
)
executor = ClaudeCodeExecutor(config)
result = executor.execute(task, project_path, role="researcher")
```

**Features:**
- Runs Claude Code CLI in subprocess
- Budget control via `--max-turns` (estimated from USD)
- Permission bypass for sandboxed environments
- Context stack integration for prompt building

**CLI Usage:**
```bash
# Basic usage
python -m spine.orchestrator run --project /path \
    --executor claude-code \
    --executor-budget 10.0

# With permission bypass (sandboxed only)
python -m spine.orchestrator run --project /path \
    --executor claude-code \
    --executor-skip-permissions
```

---

### MCPOrchestratorExecutor (v0.3.21+)

Delegates task execution to the MCP Orchestrator Blueprint for intelligent tool routing.

```python
from spine.orchestrator.executors.mcp_orchestrator import (
    MCPOrchestratorExecutor,
    MCPOrchestratorConfig,
    create_mcp_executor,
)

# Simple creation
executor = create_mcp_executor(
    base_url="http://localhost:8080",
    capabilities=["code_generation", "python"],
    fallback_enabled=True,
)
result = executor.execute(task, project_path, role="implementer")
```

**Features:**
- Intelligent tool selection based on capabilities
- Claude-first provider bias with automatic fallback
- **Graceful degradation** - falls back to SubagentExecutor if unavailable
- Learning from outcomes (score boosts)

**Graceful Fallback:**
```python
# If MCP Orchestrator is unavailable, automatic fallback occurs
if not executor.is_available():
    # SubagentExecutor is used automatically
    result.metadata["executor"] = "mcp_orchestrator_fallback"
```

**CLI Usage:**
```bash
# Use MCP Orchestrator executor
python -m spine.orchestrator run --project /path \
    --executor mcp-orchestrator \
    --executor-url http://localhost:8080

# Disable fallback (fail if unavailable)
python -m spine.orchestrator run --project /path \
    --executor mcp-orchestrator \
    --no-fallback
```

**→ [Full MCP Orchestrator Integration Guide](mcp-orchestrator-integration.md)**

---

## Executor Interface

All executors implement the base `Executor` interface:

```python
from spine.orchestrator.executors import Executor, ExecutorResult

class Executor:
    def execute(
        self,
        task: Task,
        project_path: Path,
        role: str | None = None,
    ) -> ExecutorResult:
        """Execute a task and return the result."""
        ...
```

**ExecutorResult:**
```python
@dataclass
class ExecutorResult:
    success: bool
    changes: list[str]        # List of changes made
    output: str               # Raw executor output
    error: str | None = None  # Error message if failed
    tokens_used: int = 0      # Token consumption
    duration_ms: int = 0      # Execution time
```

---

## Context Stack Integration

Both executors support context stacks from YAML scenario files (v0.3.20):

```bash
# Use a specific scenario
python -m spine.orchestrator run --project /path \
    --executor subagent \
    --scenario scenarios/research.yaml

# Role-specific scenarios
python -m spine.orchestrator run --project /path \
    --executor claude-code \
    --role-scenario "architect:scenarios/architecture.yaml" \
    --role-scenario "reviewer:scenarios/code-review.yaml"

# Disable context stacks (use legacy prompts)
python -m spine.orchestrator run --project /path \
    --executor subagent \
    --no-context-stacks
```

See [Context Stack Integration](context-stacks.md) for details on scenario files.

---

## Choosing an Executor

| Executor | Best For | Trade-offs |
|----------|----------|------------|
| **SubagentExecutor** | Programmatic control, persona-based tasks | Requires agent definitions |
| **ClaudeCodeExecutor** | Full CLI capabilities, file operations | Higher overhead, subprocess management |
| **MCPOrchestratorExecutor** | Intelligent tool routing, multi-provider | Requires external service, adds latency |

---

## Custom Executors

Create custom executors by implementing the base interface:

```python
from spine.orchestrator.executors import Executor, ExecutorResult
from spine.orchestrator.task_queue import Task
from pathlib import Path

class MyExecutor(Executor):
    def execute(
        self,
        task: Task,
        project_path: Path,
        role: str | None = None,
    ) -> ExecutorResult:
        # Your execution logic here
        return ExecutorResult(
            success=True,
            changes=["[OK] did_something"],
            output="Task completed",
        )
```

---

[← Back to Docs](README.md) | [Context Stacks →](context-stacks.md)
