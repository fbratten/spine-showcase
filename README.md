# SPINE - Multi-Agent Orchestration System

> A context engineering and multi-agent backbone framework for complex software development workflows.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Status](https://img.shields.io/badge/status-active-green)]()
[![Demo](https://img.shields.io/badge/demo-live-blue)](demos/tier-classifier/)

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
│  (fan_out, pipeline, ToolEnvelope, run_scenario.py)         │
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

**→ [Try the Interactive Tier Classifier](demos/tier-classifier/)**

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
└─────────────────────────────────────────┘
```

---

## Interactive Demos

| Demo | Description |
|------|-------------|
| **[Tier Classifier](demos/tier-classifier/)** | Determine the appropriate enforcement tier for any task |

*More demos coming soon*

---

## Use Cases

### Autonomous Software Development

SPINE enables coordinated multi-agent workflows for:

- **Code Review:** Parallel reviewers for security, style, and logic
- **Research Tasks:** Multi-source investigation with synthesis
- **UI Development:** Visual verification with browser automation
- **Architecture Design:** Structured design reviews with documentation

### Project Integration

SPINE has been successfully integrated with:

| Project | Integration Type |
|---------|------------------|
| Golden Thread System | Full MVP development with tiered enforcement |
| Adaptivearts.ai | Research and content generation workflows |

---

## Technical Highlights

### Multi-Provider Support

| Provider | Models | Status |
|----------|--------|--------|
| Anthropic | Claude Opus 4.5, Sonnet 4.5, Haiku 4.5 | ✅ Active |
| OpenAI | GPT-5 | ✅ Active |
| Google | Gemini Pro, Gemini Ultra | ✅ Active |
| xAI | Grok | ✅ Active |

### Structured Logging

All interactions are logged with:
- Full envelope with trace hierarchy
- Token usage and cost estimates
- Timing (started_at, finished_at, duration_ms)
- Experiment tracking IDs

---

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture Overview](docs/architecture.md) | System design and components |
| [Tiered Protocol](docs/tiered-enforcement.md) | Full enforcement protocol |
| [Pattern Guide](docs/patterns.md) | Fan-out and Pipeline usage |

---

## About

SPINE is developed as part of the **Adaptivearts.ai** research initiative, focusing on intelligent software development workflows and multi-agent coordination.

### Contact

- **GitHub:** [github.com/fbratten](https://github.com/fbratten)
- **Portfolio:** [View all projects](https://github.com/fbratten)

---

## License

This project is licensed under the [MIT License](LICENSE).
