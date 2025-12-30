# SPINE - Multi-Agent Orchestration System

> A context engineering and multi-agent backbone framework for complex software development workflows.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Status](https://img.shields.io/badge/status-active-green)]()
[![Version](https://img.shields.io/badge/version-0.3.17-blue)]()
[![Live Site](https://img.shields.io/badge/site-live-blue)](https://fbratten.github.io/spine-showcase/)
[![Demos](https://img.shields.io/badge/demos-6%20interactive-purple)](https://fbratten.github.io/spine-showcase/demos/)

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
| 🧠 **Context Stacks** | Reproducible, structured context management |
| 🔁 **Agentic Loop** | Autonomous "run until done" with oscillation detection |
| 📝 **AI Code Review** | Multi-persona parallel review with consensus ranking |
| 📈 **Observability** | Static HTML reports, REST API, health checks |

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
│                    Layer 1: Claude Native                    │
│  Built-in Task tool with subagent_types                     │
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

### Module Structure (v0.3.17)

```
spine/
├── core/           # ToolEnvelope, TraceScope
├── client/         # InstrumentedLLMClient, provider configs
├── patterns/       # fan_out(), pipeline()
├── orchestrator/   # AgenticLoop, OscillationTracker, TaskQueue
├── memory/         # kv_store, vector_store, scratchpad
├── review/         # AI-powered code review
├── integration/    # Token-optimized MCP execution
├── enforcement/    # Tiered enforcement gate
├── health/         # Component health monitoring
├── api/            # FastAPI REST API
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
| OpenAI | GPT-4o, GPT-4 | ✅ Active |
| Google | Gemini Pro, Gemini Ultra | ✅ Active |
| xAI | Grok | ✅ Active |

### Observability Stack

| Component | Purpose |
|-----------|---------|
| `spine/logging/` | Structured JSON logs with trace hierarchy |
| `spine/api/` | FastAPI REST API with OpenAPI docs |
| `spine/reports/` | Self-contained HTML reports with Chart.js |
| `spine/health/` | Component health monitoring |

### CLI Tools

```bash
# Run orchestrator
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

### Reference Materials

| Resource | Description |
|----------|-------------|
| [Multi-Agent Playbook (PDF)](KB/Multi-Agent-Playbook-Blueprint.pdf) | Architectural blueprint for production-ready multi-agent systems |

---

## Version History

| Version | Highlights |
|---------|------------|
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
