# SmallLLMExecutor (v0.3.27)

Orchestrate 3B-8B quantized language models via MCP self-description layers for cost-optimized, edge-capable task execution.

---

## Overview

The SmallLLMExecutor is SPINE's 6th executor type, designed for tasks where a full-size flagship model is overkill. It wraps small, fast models (CodeLlama 7B, Qwen2.5-Coder 3B, Phi-3.5, DeepSeek-Coder) and provides them with structured MCP context to compensate for their limited capabilities.

```
┌───────────────────────────────────────────────────────────┐
│                    SmallLLMExecutor                         │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │              MCP Self-Description Layers            │   │
│  │  L0: Instructions (server identity, tool guide)     │   │
│  │  L1: Schema (tool parameter reference)              │   │
│  │  L2: Resources (fetched from MCP servers)           │   │
│  │  L3: Prompts (workflow steps from MCP servers)      │   │
│  └──────────────────────┬─────────────────────────────┘   │
│                         │                                  │
│                         ▼                                  │
│  ┌──────────────────────────────────────────────────┐     │
│  │            Small LLM (3B-8B params)               │     │
│  │  Ollama (local) or Anthropic Haiku (API)          │     │
│  └──────────────────────────────────────────────────┘     │
│                         │                                  │
│                         ▼                                  │
│  ┌──────────────────────────────────────────────────┐     │
│  │         TOOL_CALL: format → MCP Execution         │     │
│  │         via MCPSessionPool (persistent)           │     │
│  └──────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────┘
```

---

## MCP Self-Description Layers

The key insight: small models achieve significantly better tool usage accuracy when given structured context at multiple layers, rather than relying on raw schemas alone. SPINE provides this context at four layers:

| Layer | Content | Token Budget |
|-------|---------|-------------|
| **L0** | Server identity, tool selection guide, workflow | ~1024 |
| **L1** | Tool parameter reference (compact schema) | ~512 |
| **L2** | Resources fetched from MCP servers | ~1024 |
| **L3** | Workflow prompts from MCP servers | ~1024 |

Total context budget: ~4096 tokens (configurable per layer).

---

## Configuration

```python
from spine.orchestrator.executors.small_llm_executor import SmallLLMExecutor, SmallLLMConfig

config = SmallLLMConfig(
    model_name="qwen2.5-coder:3b",
    provider="ollama",              # "ollama" | "anthropic"
    base_url="http://localhost:11434",
    max_context_tokens=4096,
    mcp_servers=["research-agent-mcp", "evaluation-mcp"],
    temperature=0.1,
)
executor = SmallLLMExecutor(config)
result = executor.execute(task, project_path)
```

### Supported Providers

| Provider | Models | Caching |
|----------|--------|---------|
| **Ollama** (local) | CodeLlama 7B, Qwen2.5-Coder 3B, Phi-3.5, DeepSeek-Coder | KV cache |
| **Anthropic** (API) | Haiku 4.5 | Prompt caching |

---

## CLI Usage

```bash
# Run with SmallLLMExecutor
python -m spine.orchestrator run --project /path --executor small-llm

# Combined with Dynamic Routing
python -m spine.orchestrator run --project /path \
    --executor router \
    --route ANALYSIS:small-llm \
    --route CODE:subagent
```

## Scenario Template

```yaml
# _templates/scenarios/small-llm-mcp-task.yaml
global:
  operator: "SPINE SmallLLMExecutor"
command:
  task: "${TASK_DESCRIPTION}"
context:
  background: "${MCP_INSTRUCTIONS}"   # L0
  references:
    - "${TOOL_SCHEMA}"                 # L1
    - "${CONTEXT_RESOURCES}"           # L2
constraints:
  format: "${WORKFLOW_PROMPT}"         # L3
  token_budget:
    l0_instructions: 1024
    l1_schema: 512
    l2_resources: 1024
    l3_prompts: 1024
```

---

## MCPSessionPool Integration (v0.3.28)

SmallLLMExecutor uses `MCPSessionPool` for persistent MCP connections instead of spawning a new subprocess per tool call:

- **Before (v0.3.27):** ~110-220ms overhead per MCP tool call (subprocess spawn)
- **After (v0.3.28):** Near-zero overhead (persistent connection via background event loop)

See [MCP Session Pool](mcp-session-pool.md) for details.

---

## Key Design Decisions

- **Implements Executor interface** — Transparent to AgenticLoop, composes with Dynamic Routing
- **Token budget management** — Each L0-L3 layer has a configurable budget, content truncated to fit
- **TOOL_CALL: format** — Simple text format (`TOOL_CALL: tool_name(param=value)`) instead of JSON tool_use, optimized for small model output parsing
- **Graceful degradation** — Falls back to direct execution if MCP context unavailable

---

[← Back to Docs](README.md) | [MCP Session Pool →](mcp-session-pool.md)
