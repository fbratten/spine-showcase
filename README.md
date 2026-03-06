# SPINE - Multi-Agent Orchestration System

> A context engineering and multi-agent backbone framework for complex software development workflows.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Status](https://img.shields.io/badge/status-active-green)]()
[![Version](https://img.shields.io/badge/version-0.3.30-blue)]()
[![Live Site](https://img.shields.io/badge/site-live-blue)](https://fbratten.github.io/spine-showcase/)
[![Demos](https://img.shields.io/badge/demos-7%20interactive-purple)](https://fbratten.github.io/spine-showcase/demos/)

---

## Overview

**SPINE** (Software Pipeline for INtelligent Engineering) provides standardized instrumentation, multi-provider LLM access, and orchestration patterns that connect agentic projects for long-running, complex development workflows.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| 🔄 **Multi-Agent Orchestration** | Fan-out (parallel) and Pipeline (sequential) patterns |
| 📊 **Full Traceability** | ToolEnvelope instrumentation with hierarchical trace correlation |
| 🤖 **Multi-Provider Support** | Anthropic, OpenAI, Google Gemini, Grok |
| 📋 **Tiered Enforcement** | Balanced capability usage based on task complexity |
| 🧠 **Context Stacks** | Reproducible, structured context management via YAML scenarios |
| 🔁 **Agentic Loop** | Autonomous "run until done" with oscillation detection |
| 📝 **AI Code Review** | Multi-persona parallel review with consensus ranking |
| 📈 **Observability** | Static HTML reports, REST API, health checks |
| ⚙️ **Pluggable Executors** | 7 executor types including SmallLLMExecutor for 3B-8B models |
| 🔀 **Dynamic Routing** | Automatic task classification and executor selection by type |
| 🤖 **Small LLM Support** | Orchestrate 3B-8B quantized models via MCP self-description layers |
| 🔗 **MCP Session Pool** | Persistent MCP connections with background event loop |
| 🧠 **Persistent Memory** | Optional Minna Memory integration for cross-session memory |
| 🔄 **Agent OS 2026** | OODA loop composition, episodic memory, agent processes, task DAGs |
| 🧬 **5-Tier Memory** | KVStore, Scratchpad, Ephemeral, Vector, Episodic — unified by MemoryFacade |
| 📐 **Embedding Providers** | 7 providers (Local, OpenAI, Voyage, ONNX, Gemini, Keyword, Placeholder) |

---

## 🏗️ Architectural Foundation: The Multi-Agent Playbook

SPINE implements patterns from the **Multi-Agent Playbook**—an architectural blueprint for production-ready multi-agent systems that addresses the core challenge: *How do you manage delegation, state, execution, and failure without creating chaos?*

### The General Contractor Model

SPINE follows a **closed-loop orchestrator pattern** where:

```
User
  │
  ▼
┌─────────────────────────────────────────────┐
│              SPINE Orchestrator              │
│  AgenticLoop + ToolEnvelope instrumentation │
└──────────────────┬──────────────────────────┘
                   │ fan_out() or pipeline()
       ┌───────────┼───────────┐
       ▼           ▼           ▼
   ┌───────┐   ┌───────┐   ┌───────┐
   │Worker │   │Worker │   │Worker │
   │Agent 1│   │Agent 2│   │Agent 3│
   └───┬───┘   └───┬───┘   └───┬───┘
       │           │           │
       └───────────┼───────────┘
                   │ Results via ToolEnvelope
                   ▼
┌─────────────────────────────────────────────┐
│         Synthesized Response to User         │
└─────────────────────────────────────────────┘
```

- **You prompt the Orchestrator**, not sub-agents directly
- Sub-agents report exclusively to the Orchestrator
- The Orchestrator synthesizes and delivers results
- Direct user communication from sub-agents is forbidden

### The Five Pillars

SPINE implements all five architectural pillars from the blueprint:

| Pillar | Blueprint Principle | SPINE Implementation |
|--------|---------------------|----------------------|
| **I. Communication** | Closed loops, verifiable artifacts | `ToolEnvelope` result wrapping, structured logs |
| **II. Execution** | Parallel for speed, sequential for logic | `fan_out()` and `pipeline()` patterns |
| **III. Empowerment** | Right tooling in isolated environments | MCP integration, `TraceScope` boundaries |
| **IV. State** | State in environment, not agent memory | `NEXT.md` integration, Context Stacks |
| **V. Resilience** | Blast radius containment, error routing | `OscillationTracker`, `LoopVerdict` system |

### Context Management: Signal vs. Noise

The Orchestrator holds **executive signal** (low context), while sub-agents absorb **execution noise** (high context):

```
Orchestrator Context (Signal)          Sub-Agent Context (Noise)
├── Master Plan                        ├── Full document content
├── Operational metrics                ├── Raw API responses
├── Synthesized outputs                ├── Detailed logs
└── Error signals                      └── Environment state
```

**→ [Read the full Blueprint Implementation Guide](docs/blueprint-implementation.md)**

**→ [View the Multi-Agent Playbook (PDF)](KB/Multi-Agent-Playbook-Blueprint.pdf)**

---

## Architecture

SPINE operates across **three distinct capability layers**:

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 1: Host Agent                       │
│  Built-in subagent types via host environment               │
│  (Explore, Plan, code-architect, visual-tester, etc.)       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Layer 2: MCP Servers                      │
│  External tools via Model Context Protocol                   │
│  (browser-mcp, next-conductor, research-agent-mcp)          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Layer 3: SPINE Python                      │
│  Custom orchestration framework                              │
│  (fan_out, pipeline, ToolEnvelope, AgenticLoop)             │
└─────────────────────────────────────────────────────────────┘
```

### Context Stack Structure

SPINE uses a hierarchical context stack for consistent LLM interactions:

```json
{
  "global": { "operator": "...", "brand": "..." },
  "character": { "speaker": "...", "audience": "..." },
  "command": { "task": "...", "success": "..." },
  "constraints": { "tone": "...", "format": "...", "do": [], "dont": [] },
  "context": { "background": "...", "references": [] },
  "input": { "user_request": "..." }
}
```

### Module Structure (v0.3.30)

```
spine/
├── core/           # ToolEnvelope, TraceScope
├── client/         # InstrumentedLLMClient, provider configs, retry/timeout
├── patterns/       # fan_out(), pipeline(), hermeneutic_loop(), safe_access()
├── orchestrator/   # AgenticLoop, OscillationTracker, TaskQueue
│   ├── context_stack.py         # Context stack loader/builder
│   ├── context_discovery.py     # Layered context discovery L1-L4
│   ├── task_router.py           # Dynamic Routing — TaskTypeRouter (v0.3.26)
│   ├── routing_callbacks.py     # Routing callbacks factory (v0.3.26)
│   ├── mcp_self_description.py  # 4-layer MCP self-description generator (v0.3.28)
│   ├── capability_registry.py   # Project capability scanning + S41 map
│   ├── gap_tracker.py           # Structured gap detection and clustering
│   └── executors/               # 7 pluggable executors
│       ├── base.py              # Executor interface + PlaceholderExecutor
│       ├── subagent.py          # SubagentExecutor + context stacks
│       ├── claude_code.py       # ClaudeCodeExecutor (CLI subprocess)
│       ├── mcp_orchestrator.py  # MCPOrchestratorExecutor
│       ├── content_pipeline.py  # ContentPipelineExecutor (video/content)
│       ├── small_llm_executor.py    # SmallLLMExecutor — 3B-8B models (v0.3.27)
│       └── mcp_session_pool.py      # MCPSessionPool — persistent sessions (v0.3.28)
├── agent_os/       # Agent OS 2026 (v0.3.29-v0.3.30)
│   ├── ooda.py                  # OODALoop, OODAConfig, OODACycle, LoopContext
│   ├── world.py                 # WorldState, WorldSnapshot
│   ├── outcome.py               # Outcome canonical result schema
│   └── process.py               # AgentProcess, ProcessManager
├── memory/         # 5-tier memory system
│   ├── kv_store.py              # Tier 1: namespace-scoped key-value
│   ├── scratchpad.py            # Tier 2: short-term task notes
│   ├── ephemeral.py             # Tier 3: session-scoped with decay
│   ├── vector_store.py          # Tier 4: hybrid semantic + keyword search
│   ├── episodic.py              # Tier 5: goal-based episode recall (v0.3.29)
│   ├── facade.py                # MemoryFacade — unified cross-tier search
│   ├── verdict_router.py        # Routes accept/reject/revise to tiers
│   ├── persistence.py           # SQLitePersistence, FilePersistence
│   └── embeddings/              # 7 embedding providers
│       ├── base.py              # EmbeddingProvider ABC
│       ├── local.py             # SentenceTransformers
│       ├── openai.py            # OpenAI embeddings API
│       ├── voyage.py            # Voyage AI (code-optimized)
│       ├── onnx.py              # ONNX Runtime
│       ├── gemini.py            # Google Gemini
│       ├── keyword.py           # TF-IDF fallback
│       └── placeholder.py       # Testing/development
├── grammar/        # EBNF-Rig Veda knowledge annotation
├── review/         # AI-powered code review
├── integration/    # Token-optimized MCP execution
├── enforcement/    # Tiered + Five-Point Protocol enforcement
├── health/         # Component health monitoring
├── api/            # FastAPI REST API + /api/reviews
├── reports/        # Static HTML report generator
└── logging/        # Structured JSON logging
```

---

## Tiered Enforcement Protocol

SPINE balances capability usage against overhead costs through a **three-tier system**:

| Tier | Task Type | Enforcement | Examples |
|------|-----------|-------------|----------|
| **Tier 1** | Simple | None required | Typo fixes, single-file edits |
| **Tier 2** | Medium | Recommended | Multi-file changes, new features |
| **Tier 3** | Complex | Mandatory | Architecture decisions, research, UI-heavy |

### Why Tiered Enforcement?

| Factor | Consideration |
|--------|---------------|
| **Token Cost** | Parallel subagents = 2-6x cost increase |
| **Latency** | Subagent spawn adds 10-30 seconds |
| **Over-engineering** | Simple tasks don't need orchestration |
| **Context Fragmentation** | Subagents don't share full conversation context |

**→ [Try the Interactive Tier Classifier](https://fbratten.github.io/spine-showcase/demos/tier-classifier/)**

---

## Core Patterns

### Fan-Out (Parallel Execution)

Execute multiple tasks simultaneously with automatic result aggregation:

```
                    ┌─────────────┐
                    │   Parent    │
                    │  Envelope   │
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌────────────┐  ┌────────────┐  ┌────────────┐
    │  Analyst A │  │  Analyst B │  │  Analyst C │
    └────────────┘  └────────────┘  └────────────┘
           │               │               │
           └───────────────┼───────────────┘
                           ▼
                    ┌─────────────┐
                    │  Aggregate  │
                    │   Results   │
                    └─────────────┘
```

**Use Cases:** Research tasks, parallel code analysis, multi-source data gathering

### Pipeline (Sequential Processing)

Chain processing steps with automatic result transformation:

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Analyze │ ──▶ │ Extract │ ──▶ │Transform│ ──▶ │Synthesize│
└─────────┘     └─────────┘     └─────────┘     └─────────┘
```

**Use Cases:** Document processing, staged analysis, build pipelines

### Agentic Loop (Autonomous Execution)

Run tasks until completion with built-in resilience:

```
┌──────────────────────────────────────────────────────────┐
│                     AgenticLoop                          │
├──────────────────────────────────────────────────────────┤
│  ┌─────────┐    ┌──────────┐    ┌───────────┐           │
│  │  Task   │───▶│ Execute  │───▶│ Evaluate  │           │
│  │  Queue  │    │          │    │           │           │
│  └─────────┘    └──────────┘    └─────┬─────┘           │
│                                       │                  │
│       ┌───────────────────────────────┼──────────┐      │
│       │                               │          │      │
│       ▼                               ▼          ▼      │
│   ┌────────┐                    ┌────────┐  ┌────────┐  │
│   │ ACCEPT │                    │ REVISE │  │ REJECT │  │
│   │  Done  │                    │ Retry  │  │  Skip  │  │
│   └────────┘                    └────────┘  └────────┘  │
│                                                          │
│  OscillationTracker: Detects stuck states               │
│  (A-B-A-B patterns, repeated errors)                    │
└──────────────────────────────────────────────────────────┘
```

### ToolEnvelope (Instrumentation)

Every LLM call is wrapped for full traceability:

```
┌─────────────────────────────────────────┐
│              ToolEnvelope               │
├─────────────────────────────────────────┤
│  id: "call-abc123"                      │
│  tool: "anthropic:claude-sonnet-4-5"    │
│  trace:                                 │
│    root_id: "task-xyz"                  │
│    parent_id: "orchestrator-001"        │
│    span_id: "subagent-research"         │
│  metadata:                              │
│    tags: ["research", "phase-1"]        │
│    experiment_id: "exp-2025-001"        │
│  metrics:                               │
│    tokens_in, tokens_out, latency_ms    │
└─────────────────────────────────────────┘
```

---

## Interactive Demos

**[View all demos →](https://fbratten.github.io/spine-showcase/demos/)**

| Demo | Description |
|------|-------------|
| **[Tier Classifier](https://fbratten.github.io/spine-showcase/demos/tier-classifier/)** | Determine the appropriate enforcement tier for any task |
| **[Provider Picker](https://fbratten.github.io/spine-showcase/demos/provider-picker/)** | Choose the right LLM provider based on your task type |
| **[Cost Calculator](https://fbratten.github.io/spine-showcase/demos/cost-calculator/)** | Estimate API costs by model and token usage |
| **[Fan-Out Simulator](https://fbratten.github.io/spine-showcase/demos/fan-out-simulator/)** | Visualize parallel task execution with configurable workers |
| **[Pipeline Builder](https://fbratten.github.io/spine-showcase/demos/pipeline-builder/)** | Build and simulate sequential processing chains |

---

## Use Cases

### Autonomous Software Development

SPINE enables coordinated multi-agent workflows for:

- **Code Review:** Parallel reviewers for security, style, and logic with consensus ranking
- **Research Tasks:** Multi-source investigation with conflict detection and synthesis
- **UI Development:** Visual verification with browser automation
- **Architecture Design:** Structured design reviews with documentation generation

### Project Integration

SPINE has been successfully integrated with:

| Project | Integration Type |
|---------|------------------|
| Golden Thread System | Full MVP development with tiered enforcement |
| spine-dashboard | Real-time monitoring via SPINE API |
| Adaptivearts.ai | Research and content generation workflows |

---

## Technical Highlights

### Multi-Provider Support

| Provider | Models | Status |
|----------|--------|--------|
| Anthropic | Claude Opus 4.5, Sonnet 4.5, Haiku 4.5 | ✅ Active |
| Google | Gemini 3 Pro, Gemini 3 Flash | ✅ Active |
| OpenAI | GPT-5.1, GPT-5 mini | ✅ Active |
| xAI | Grok 4.1 | ✅ Active |

### Observability Stack

| Component | Purpose |
|-----------|---------|
| `spine/logging/` | Structured JSON logs with trace hierarchy |
| `spine/api/` | FastAPI REST API with OpenAPI docs |
| `spine/reports/` | Self-contained HTML reports with Chart.js |
| `spine/health/` | Component health monitoring |

### CLI Tools

```bash
# Run orchestrator with SubagentExecutor (uses .claude/agents/ personas)
python -m spine.orchestrator run --project /path --executor subagent

# Run with Dynamic Routing (auto-selects executor by task type) [v0.3.26]
python -m spine.orchestrator run --project /path --executor router \
    --route CODE:subagent --route RESEARCH:claude-code

# Run with SmallLLMExecutor (3B-8B models via MCP) [v0.3.27]
python -m spine.orchestrator run --project /path --executor small-llm

# Classify task type without executing [v0.3.26]
python -m spine.orchestrator classify --project /path --task-id TASK-001

# Generate MCP self-description for a server [v0.3.28]
python -m spine.orchestrator describe --project /path --server my-mcp

# Run with context stacks from scenario files
python -m spine.orchestrator run --project /path --executor subagent --scenario scenarios/research.yaml

# Run with LLM evaluation
python -m spine.orchestrator run --project /path --llm-eval

# Generate reports
python -m spine.reports generate --title "Sprint Report" --days 7

# Health checks
python -m spine.health --verbose

# Code review
python -m spine.review . --parallel

# Start API server
python -m spine.api --port 8000
```

---

## Documentation

| Document | Description |
|----------|-------------|
| [Blueprint Implementation](docs/blueprint-implementation.md) | How SPINE implements the Multi-Agent Playbook |
| [Architecture Overview](docs/architecture.md) | System design and components |
| [Pattern Guide](docs/patterns.md) | Fan-out and Pipeline usage |
| [Tiered Protocol](docs/tiered-enforcement.md) | Full enforcement protocol |
| [Executor Framework](docs/executors.md) | 7 executor types including SmallLLMExecutor |
| [Dynamic Routing](docs/dynamic-routing.md) | Task classification and executor selection (NEW v0.3.26) |
| [SmallLLMExecutor](docs/small-llm-executor.md) | 3B-8B model orchestration via MCP self-description (NEW v0.3.27) |
| [MCP Session Pool](docs/mcp-session-pool.md) | Persistent MCP sessions + self-description generator (v0.3.28) |
| [Agent OS 2026](docs/agent-os.md) | OODA loop, episodic memory, agent processes, task DAGs (NEW v0.3.29-v0.3.30) |
| [Memory System](docs/memory-system.md) | 5-tier memory architecture with MemoryFacade (NEW v0.3.29) |
| [Context Stack Integration](docs/context-stacks.md) | YAML scenario files for prompt building |
| [MCP Orchestrator Integration](docs/mcp-orchestrator-integration.md) | Optional intelligent tool routing |
| [Minna Memory Integration](docs/minna-memory-integration.md) | Persistent cross-session memory |
| [Agent Harness Automation](docs/claude-code-automation.md) | Disable prompts, auto-reload context (Claude Code) |

### Reference Materials

| Resource | Description |
|----------|-------------|
| [Multi-Agent Playbook (PDF)](KB/Multi-Agent-Playbook-Blueprint.pdf) | Architectural blueprint for production-ready multi-agent systems |

---

## Version History

| Version | Highlights |
|---------|------------|
| **0.3.30** | Agent Processes (ProcessManager), Task DAG (dependency resolution, cycle detection) |
| **0.3.29** | Agent OS 2026 — OODA loop, EpisodicMemory, WorldState, Outcome, 7 embedding providers, MemoryFacade |
| **0.3.28** | MCPSessionPool (persistent MCP sessions) + MCP Self-Description Generator (4-layer L0-L3) |
| **0.3.27** | SmallLLMExecutor — orchestrate 3B-8B quantized LLMs via MCP self-description layers |
| **0.3.26** | Dynamic Routing — TaskTypeRouter, classify_task_type, routing callbacks + Pattern C + retry/timeout |
| **0.3.25** | Memory-First Learning Loop — 5 behaviors, gap tracker, capability registry, session consolidation |
| **0.3.24** | Content pipeline, ephemeral session memory, context discovery L1-L4, runtime tier enforcement |
| **0.3.22** | Minna Memory Integration - persistent cross-session memory with graceful fallback |
| **0.3.21** | MCP Orchestrator Integration - optional intelligent tool routing with graceful fallback |
| **0.3.20** | Context Stack Integration - executors use `scenarios/*.yaml` for prompt building |
| **0.3.19** | Executor Framework - `SubagentExecutor`, `ClaudeCodeExecutor` with pluggable design |
| **0.3.18** | Dashboard integration - `/api/reviews` endpoints for review history |
| **0.3.17** | Inline diff annotations, cost tracking per review |
| **0.3.16** | NEXT.md integration for AgenticLoop |
| **0.3.15** | `create_spine_llm_evaluator()` factory |
| **0.3.14** | Static HTML report generator |
| **0.3.13** | FastAPI REST API surface |
| **0.3.12** | Health check system, common utilities |
| **0.3.11** | Tier enforcement gate (commit-msg hook) |
| **0.3.10** | Token-optimized MCP execution (57-87% savings) |
| **0.3.9** | ConflictResolver for multi-agent synthesis |
| **0.3.6-8** | AI-powered code review module |

---

## About

SPINE is developed as part of the **AdaptiveArts.ai** research initiative, focusing on intelligent software development workflows and multi-agent coordination.

### The Meta-Goal

> *"The goal is not to build the application. It is to build the system that builds the application."*

SPINE embodies this philosophy—it's a backbone framework that enables building applications through orchestrated multi-agent workflows.

### Contact

- **GitHub:** [github.com/fbratten](https://github.com/fbratten)
- **Portfolio:** [View all projects](https://github.com/fbratten)

---

## License

This project is licensed under the [MIT License](LICENSE).
