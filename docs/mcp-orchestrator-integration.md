# MCP Orchestrator Integration (v0.3.21+)

SPINE provides an **optional integration** with the MCP Orchestrator Blueprint project for intelligent tool routing with Claude-first dispatch.

---

## Overview

The `MCPOrchestratorExecutor` allows SPINE to delegate task execution to an external MCP Orchestrator service, which provides:

- **Intelligent tool selection** based on task capabilities
- **Claude-first provider bias** with automatic fallback
- **Learning from outcomes** (score boosts based on history)
- **Full observability** (structured logging, metrics)

**Key principle:** SPINE continues to work normally if MCP Orchestrator is unavailable. The integration uses **graceful degradation**—falling back to `SubagentExecutor` automatically.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         SPINE                                │
│                                                              │
│  AgenticLoop                                                 │
│  ├── TaskQueue                                               │
│  ├── Evaluators (Build/Test/LLM)                            │
│  └── Executor Selection:                                     │
│      │                                                       │
│      ├─▶ MCPOrchestratorExecutor (if available)             │
│      │       │                                               │
│      │       │ health_check() ──▶ Success? ──▶ Use it       │
│      │       │                                               │
│      │       └──────────────────▶ Failed?  ──┐              │
│      │                                        │              │
│      └─▶ SubagentExecutor (fallback) ◀───────┘              │
│                                                              │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           │ HTTP (only if MCP Orchestrator running)
                           │ http://localhost:8080
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              MCP Orchestrator Blueprint                      │
│                                                              │
│  Core Orchestrator                                           │
│  ├── Decision Engine (tool selection)                        │
│  ├── Invocation Engine (tool calling)                        │
│  └── Claude-first bias (1.5x weight)                        │
│                                                              │
│  Supporting Modules:                                         │
│  ├── Config Engine                                           │
│  ├── Logging & Observability                                │
│  ├── Dashboard (API endpoints)                              │
│  └── Learning Layer (score boosts)                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Separation of Concerns

| Responsibility | SPINE | MCP Orchestrator |
|----------------|-------|------------------|
| **WHEN** to execute | AgenticLoop decides | - |
| **HOW** to execute | - | Intelligent routing |
| Workflow orchestration | Yes | - |
| Task queue management | Yes | - |
| Oscillation detection | Yes | - |
| Build/Test verification | Yes | - |
| Tool selection | - | Yes |
| Provider fallback | - | Yes |
| Learning from outcomes | - | Yes |

**Summary:** SPINE decides WHEN, MCP Orchestrator decides HOW.

---

## Usage

### Basic Usage

```python
from spine.orchestrator.executors.mcp_orchestrator import (
    MCPOrchestratorExecutor,
    MCPOrchestratorConfig,
    create_mcp_executor,
)

# Simple creation with defaults
executor = create_mcp_executor(
    base_url="http://localhost:8080",
    capabilities=["code_generation", "python"],
)

# Check availability
if executor.is_available():
    result = executor.execute(task, project_path, role="implementer")
else:
    print("MCP Orchestrator not available, fallback will be used")
```

### With Configuration

```python
config = MCPOrchestratorConfig(
    base_url="http://localhost:8080",
    timeout_seconds=60,
    default_capabilities=["code_generation", "python"],
    fallback_enabled=True,  # Enable SubagentExecutor fallback
)

executor = MCPOrchestratorExecutor(config)
result = executor.execute(task, project_path, role="architect")
```

### CLI Usage (when available)

```bash
# Use MCP Orchestrator executor
python -m spine.orchestrator run --project /path \
    --executor mcp-orchestrator \
    --executor-url http://localhost:8080

# With explicit fallback disable (fail if unavailable)
python -m spine.orchestrator run --project /path \
    --executor mcp-orchestrator \
    --no-fallback
```

---

## Graceful Degradation

The integration is designed to **never break SPINE** if MCP Orchestrator is unavailable:

```python
# Pseudocode of fallback logic
executor = MCPOrchestratorExecutor(config)

if executor.is_available():
    # MCP Orchestrator is running - use it
    result = executor.execute(task, project_path, role)
else:
    # Automatic fallback to SubagentExecutor
    logger.warning("MCP Orchestrator unavailable, using SubagentExecutor")
    result = fallback_executor.execute(task, project_path, role)
```

### Fallback Scenarios

| Scenario | Behavior |
|----------|----------|
| MCP Orchestrator not running | Automatic fallback to SubagentExecutor |
| Network timeout | Automatic fallback with warning logged |
| HTTP error (4xx/5xx) | Automatic fallback with error logged |
| httpx not installed | Import error caught, fallback used |

---

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MCP_ORCHESTRATOR_URL` | `http://localhost:8080` | Base URL for MCP Orchestrator |
| `MCP_ORCHESTRATOR_TIMEOUT` | `60` | Request timeout in seconds |
| `MCP_ORCHESTRATOR_FALLBACK` | `true` | Enable automatic fallback |
| `MCP_ORCHESTRATOR_API_KEY` | - | Optional API key for authentication |

### Config from Environment

```python
# Load config from environment variables
config = MCPOrchestratorConfig.from_env()
executor = MCPOrchestratorExecutor(config)
```

---

## API Endpoints Used

The executor communicates with MCP Orchestrator via these endpoints:

### Health Check
```http
GET /health/ready
```
Returns 200 if MCP Orchestrator is ready to accept requests.

### Execute Task
```http
POST /execute
Content-Type: application/json

{
  "task": "Generate a Python function to calculate fibonacci",
  "capabilities": ["code_generation", "python"],
  "context": {
    "project_path": "/path/to/project",
    "task_id": "task-001",
    "role": "implementer"
  },
  "timeout_ms": 60000
}
```

**Response:**
```json
{
  "request_id": "uuid-here",
  "status": "success",
  "result": "def fibonacci(n):\n    ...",
  "tool_used": "claude_code_generation",
  "provider": "anthropic",
  "latency_ms": 1234,
  "tokens": {
    "input": 150,
    "output": 200,
    "total": 350
  }
}
```

---

## Role to Capabilities Mapping

The executor maps SPINE roles to MCP Orchestrator capabilities:

| SPINE Role | Capabilities |
|------------|--------------|
| `architect` | system_design, architecture, planning |
| `implementer` | code_generation, python, implementation |
| `reviewer` | code_review, analysis, quality |
| `researcher` | research, analysis, synthesis |

---

## ExecutorResult Metadata

When using MCP Orchestrator, the result includes additional metadata:

```python
result = executor.execute(task, project_path, role)

# Metadata when MCP Orchestrator is used
result.metadata = {
    "executor": "mcp_orchestrator",
    "request_id": "uuid-here",
    "tool_used": "claude_code_generation",
    "provider": "anthropic",
    "latency_ms": 1234,
}

# Metadata when fallback is used
result.metadata = {
    "executor": "mcp_orchestrator_fallback",
    "fallback_reason": "mcp_orchestrator_unavailable",
}
```

---

## Prerequisites

### 1. MCP Orchestrator Must Be Running

```bash
cd "/path/to/Adaptive MCP Orchestrator Blueprint"
cd projects/p6-infrastructure
docker-compose up -d
```

Verify:
```bash
curl http://localhost:8080/health
# Should return: {"status": "healthy", ...}
```

### 2. httpx Dependency

The executor requires `httpx` for HTTP communication:

```bash
pip install httpx
```

This is already included in SPINE's `requirements.txt` as an optional dependency.

---

## Troubleshooting

### MCP Orchestrator Not Available

**Symptom:** Warning logged, fallback to SubagentExecutor

**Check:**
```bash
# Is it running?
curl http://localhost:8080/health

# Check Docker
docker ps | grep mcp

# Start if needed
docker-compose up -d
```

### Connection Timeout

**Symptom:** `MCP Orchestrator timeout after 60s`

**Solutions:**
1. Increase timeout in config: `timeout_seconds=120`
2. Check network connectivity
3. Check MCP Orchestrator logs for slow operations

### Import Error (httpx not installed)

**Symptom:** `ImportError: httpx not found`

**Solution:**
```bash
pip install httpx
```

---

## How to Disable

If you want to disable MCP Orchestrator integration:

```python
# Option 1: Disable fallback (fail if unavailable)
config = MCPOrchestratorConfig(fallback_enabled=False)

# Option 2: Use SubagentExecutor directly
from spine.orchestrator.executors import SubagentExecutor
executor = SubagentExecutor(config)
```

---

[← Back to Docs](README.md) | [Executors →](executors.md)
