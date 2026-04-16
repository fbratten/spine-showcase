# SPINE - Multi-Agent Orchestration System

> A context engineering and multi-agent backbone framework for complex software development workflows.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Status](https://img.shields.io/badge/status-active-green)]()
[![Version](https://img.shields.io/badge/version-0.5.0-blue)]()
[![Live Site](https://img.shields.io/badge/site-live-blue)](https://fbratten.github.io/spine-showcase/)
[![Demos](https://img.shields.io/badge/demos-7%20interactive-purple)](https://fbratten.github.io/spine-showcase/demos/)

---

## Overview

**SPINE** is a RunContext-governed orchestration runtime and multi-agent backbone. Its compiled execution path (`SkillCompiler → PlanArtifact → PlanExecutor`) is operational through two production transports: CLI (`cmd_execute`) and HTTP (`POST /api/orchestrator/execute`). Broader legacy runtime surfaces (`cmd_run`/AgenticLoop, `OODALoop.run()`, `run_scenario.py`, API sibling routes, `spine.review` CLI) remain intentionally separate unless a concrete trigger justifies adoption.

> *Two production transports on the compiled execution path: CLI and HTTP. Other runtime surfaces intentionally separate.*

### Key Capabilities

**Production compiled execution path (adopted):**

| Capability | Description |
|------------|-------------|
| 🎯 **Authority Inversion** | RunContext as sole runtime truth — 4 inverted modules, SkillCompiler cognitive compiler, PlanValidator (v0.5.0) |
| ⚙️ **Compiled Execution** | `SkillCompiler → PlanArtifact → PlanExecutor` — via `execute_compiled_plan(ctx, executor, project_path)` |
| 🖥️ **CLI Transport** | `python -m spine.orchestrator execute --task "..."` → `cmd_execute` (S53) |
| 🌐 **HTTP Transport** | `POST /api/orchestrator/execute` → `post_execute` (S54) |
| 📊 **Full Traceability** | ToolEnvelope instrumentation with hierarchical trace correlation |
| 🤖 **Multi-Provider Support** | Anthropic, OpenAI, Google Gemini, Grok |
| 📋 **Tiered Enforcement** | Balanced capability usage based on task complexity |
| ⚙️ **Pluggable Executors** | 7 executor types including SmallLLMExecutor for 3B-8B models |
| 🔗 **MCP Session Pool** | Persistent MCP connections with background event loop |
| 🧬 **7-Tier Memory** | KV, Scratchpad, Ephemeral, Vector, Episodic, DeepMemory (pgvector), GraphMemory — unified by MemoryFacade |
| 📐 **Embedding Providers** | 7 providers (Local, OpenAI, Voyage, ONNX, Gemini, Keyword, Placeholder) |

**Peer / reference runtime surfaces (intentionally separate from the compiled execution transports):**

| Surface | Status | Notes |
|---------|--------|-------|
| 🔁 **AgenticLoop** (`cmd_run`) | Peer pattern | Task-queue-driven "run until done"; deferred adoption onto compiled path pending typed I/O or new runtime feature |
| 🧭 **OODA Loop** (`OODALoop.run()`) | Peer RunContext-authoritative cognition loop | Not part of the compiled execution transport path adopted by CLI/HTTP |
| 📝 **spine.review CLI** | Peer pattern | Multi-persona parallel code review; peer orchestration, not compiled-plan execution |
| 🧠 **Context Stacks** (`run_scenario.py`) | Reference runtime | One-shot instrumented LLM with context-stack system prompt; structurally distinct from compiled PlanArtifact execution |
| 🔄 **Fan-out / Pipeline patterns** | Reference | Parallelism/fan-out patterns remain deferred on the compiled path |
| 📈 **Observability** | Reference | Static HTML reports, REST sibling routes, health checks — management/query, not execution |
| 🧠 **Persistent Memory** | Reference | Optional Minna Memory integration for cross-session memory |
| 🔄 **Agent OS 2026 (internal phase term)** | Reference / layer name | Internal label for the OODA composition + episodic memory + agent process + task DAG layer (v0.3.30). Not SPINE's public identity — SPINE is an orchestration runtime and multi-agent backbone. |

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

### Module Structure (v0.4.0)

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
├── agent_os/       # Agent OS 2026 (v0.3.29-v0.4.0)
│   ├── ooda.py                  # OODALoop, OODAConfig, OODACycle, LoopContext
│   ├── world.py                 # WorldState, WorldSnapshot
│   ├── outcome.py               # Outcome canonical result schema
│   └── process.py               # AgentProcess, ProcessManager
├── memory/         # 7-tier memory system (v0.4.0)
│   ├── kv_store.py              # Tier 1: namespace-scoped key-value
│   ├── scratchpad.py            # Tier 2: short-term task notes
│   ├── ephemeral.py             # Tier 3: session-scoped with decay
│   ├── vector_store.py          # Tier 4: hybrid semantic + keyword search
│   ├── episodic.py              # Tier 5: goal-based episode recall (v0.3.29)
│   ├── deep_store.py            # Tier 6: PostgreSQL + pgvector deep memory (v0.4.0)
│   ├── deep_config.py           # DeepStoreConfig (connection, decay, scoping)
│   ├── graph_memory.py          # Tier 7: graph traversal + analytics (v0.4.0)
│   ├── hooks.py                 # MemoryHooks — OODA orient/reflect integration (v0.4.0)
│   ├── federated.py             # FederatedMemory — cross-project Minna queries (v0.4.0)
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

### CLI and HTTP — Production Transports

The two adopted transports for compiled plan execution:

```bash
# CLI: Compile → Policy → Execute → Verify through RunContext (S53)
python -m spine.orchestrator execute --task "..." [--executor subagent] [--dry-run] [--plan-only]

# HTTP (S54): identical semantics via REST
POST /api/orchestrator/execute
Content-Type: application/json
{
  "task": "...",
  "executor": "subagent",
  "dry_run": false,
  "plan_only": false
}
```

Both transports call `execute_compiled_plan(ctx, executor, project_path)` under the hood and leave all execution truth on `RunContext`. `--dry-run` is execution-adjacent (still runs the official path; per-executor side-effect suppression). `--plan-only` is a compile/policy preview that stops before `begin_execution()`.

### Other Runtime Surfaces — Reference / Peer Patterns

The following exist in the source repository as peer patterns or reference runtimes. They are intentionally separate from the compiled execution transports and are documented here for completeness, not as adopted surfaces.

```bash
# AgenticLoop (peer pattern — task-queue-driven "run until done")
python -m spine.orchestrator run --project /path --executor subagent

# Scenario runner (reference — one-shot instrumented LLM with context stacks)
python run_scenario.py scenarios/research.yaml

# Code review CLI (peer pattern — multi-persona parallel review)
python -m spine.review . --parallel

# Utility CLIs (observability, not execution)
python -m spine.reports generate --title "Sprint Report" --days 7
python -m spine.health --verbose
python -m spine.api --port 8000
```

See the [source repository](https://github.com/fbratten/spine) for the full, up-to-date CLI surface.

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
| [Agent OS 2026](docs/agent-os.md) | OODA loop, deep memory hooks, agent processes, task DAGs (v0.3.29-v0.4.0) |
| [Memory System](docs/memory-system.md) | 7-tier memory architecture with MemoryFacade (v0.3.29-v0.4.0) |
| [Deep Memory](docs/deep-memory.md) | PostgreSQL+pgvector deep store, graph memory, federation, OODA hooks (NEW v0.4.0) |
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
| **0.5.0** | Authority Inversion - RunContext as sole runtime truth. 4 modules inverted (FivePointProtocol, VerdictRouter, OODALoop, ContentPipelineExecutor). SkillCompiler cognitive compiler with tool_checker enforcement. PlanValidator (6 checks). Policy tags. 84 new tests. |
| **0.4.0** | Phase 3 Deep Memory - DeepMemoryStore (pgvector Tier 6), GraphMemory (Tier 7), FederatedMemory, MemoryHooks + OODA integration, dashboard health check |
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
