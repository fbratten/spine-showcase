# MCP Session Pool & Self-Description Generator (v0.3.28)

Persistent MCP sessions via background event loop + 4-layer self-description generation for MCP servers.

---

## MCPSessionPool

### The Problem

In v0.3.27, SmallLLMExecutor spawned a new MCP subprocess per tool call (~110-220ms overhead each). For workflows with 5-10 tool calls, this added 1-2 seconds of pure overhead.

### The Solution

`MCPSessionPool` maintains persistent MCP sessions in a background thread with its own asyncio event loop, exposing a sync API to the rest of SPINE.

```
┌────────────────────────────────────────────────────────────┐
│                    MCPSessionPool                           │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Background Thread (asyncio event loop)      │  │
│  │                                                       │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐           │  │
│  │  │ Session  │  │ Session  │  │ Session  │           │  │
│  │  │ Server A │  │ Server B │  │ Server C │           │  │
│  │  └──────────┘  └──────────┘  └──────────┘           │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                  │
│                    Sync API Surface                         │
│  get_session() | call_tool() | fetch_resources()           │
│  fetch_prompt() | close_all() | reconnect()                │
└────────────────────────────────────────────────────────────┘
```

### Usage

```python
from spine.orchestrator.executors.mcp_session_pool import MCPSessionPool

pool = MCPSessionPool()

# Open session (lazy, cached)
entry = pool.get_session("research-agent-mcp")

# Call tools via pooled session
result = pool.call_tool("research-agent-mcp", "search_sources", {
    "query": "context engineering patterns"
})

# Fetch L2 resources
resources = pool.fetch_resources("research-agent-mcp", [
    "research://schema"
])

# Fetch L3 prompts
prompt = pool.fetch_prompt("research-agent-mcp", "analyze", {
    "topic": "multi-agent coordination"
})

# Cleanup
pool.close_all()
```

### Key Features

| Feature | Description |
|---------|-------------|
| **Persistent sessions** | Open once, reuse across multiple tool calls |
| **Background event loop** | `threading.Thread` + `asyncio.new_event_loop()` — isolated from main thread |
| **Sync API** | All methods are synchronous — no async/await needed by callers |
| **Lazy initialization** | Sessions opened on first use, cached for reuse |
| **Reconnection** | `reconnect(server_name)` for recovery from dead sessions |
| **Reverse teardown** | `close_all()` closes sessions in reverse order |

---

## MCP Self-Description Generator

Generates 4-layer MCP self-descriptions for SPINE-managed servers, following the pattern from mcp-builder-mcp.

### The 4 Layers

| Layer | Role | Content |
|-------|------|---------|
| **L0** | `instructions=` | Server identity, tool selection guide, workflow, triggers |
| **L1** | `@mcp.resource()` | Tool parameter reference as `://schema` resource |
| **L2** | `@mcp.prompt()` | Workflow steps as prompt definitions |
| **L3** | Static docs | README.md, SKILL.md, MCP_INFO.md content |

### Two Modes

**Standalone** (default): Pure Python composition, no external dependencies.

```python
from spine.orchestrator.mcp_self_description import (
    MCPSelfDescriptionGenerator,
    DescriptionConfig,
    ToolDescriptor,
    WorkflowStep,
)

gen = MCPSelfDescriptionGenerator()
result = gen.generate(DescriptionConfig(
    server_name="my-analyzer",
    server_description="Analyzes code quality",
    tools=[
        ToolDescriptor(name="analyze", description="Analyze project",
                       parameters={"path": "string"}),
    ],
    workflows=[
        WorkflowStep(order=1, tool_name="analyze", description="Run analysis"),
    ],
))

print(result.instructions)   # L0
print(result.resources)      # L1
print(result.prompts)        # L2
print(result.docs)           # L3
```

**Delegated**: Bridges to mcp-builder-mcp for full templated output with generated server code, tests, and documentation. Falls back to standalone if mcp-builder-mcp is unavailable.

```python
config = DescriptionConfig(
    server_name="my-analyzer",
    mode="delegated",  # Uses mcp-builder-mcp
    ...
)
result = gen.generate(config)
# result.metadata["mode"] == "delegated"
# result.metadata["pattern_id"] == "pattern-abc123"
```

### CLI

```bash
# Generate self-description for a single server
python -m spine.orchestrator describe --project /path --server my-mcp

# Generate for all MCP servers in .mcp.json
python -m spine.orchestrator describe --project /path

# JSON output
python -m spine.orchestrator describe --project /path --server my-mcp --output json
```

### CapabilityRegistry Integration

```python
gen = MCPSelfDescriptionGenerator()

# Auto-generate from a CapabilityRegistry scan result
result = gen.generate_from_capability(capability, project_path)

# Callback factory for AgenticLoop
callbacks = create_description_callbacks(
    generator=gen,
    project_path=project_path,
)
```

---

## Testing

```bash
python scripts/tests/test_mcp_session_pool.py       # 32 tests
python scripts/tests/test_mcp_self_description.py    # 34 tests
python scripts/tests/test_small_llm.py               # 40 tests (uses pool)
```

---

[← Back to Docs](README.md) | [SmallLLMExecutor ←](small-llm-executor.md)
