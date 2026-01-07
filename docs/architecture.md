# SPINE Architecture Overview

SPINE is a context engineering and multi-agent backbone framework built for long-running, complex software development workflows. It handles instrumentation, multi-provider LLM access, and orchestration patterns that tie agentic projects together.

The goal: coordinated multi-agent development with full traceability, cost tracking, and reproducible context stacks.

---

## Three Capability Layers

SPINE works across three layers:

| Layer | What | Why |
|-------|------|-----|
| **1. Claude Native** | Task tool subagents (Explore, Plan, code-architect, visual-tester) | Parallel agents without MCP overhead |
| **2. MCP Servers** | browser-mcp, next-conductor, research-agent-mcp, smart-inventory | External tool integration |
| **3. SPINE Python** | fan_out(), pipeline(), ToolEnvelope, TraceScope | Custom instrumented orchestration |

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 1: Claude Native                   │
│  Built-in Task tool with subagent_types                     │
│  (Explore, Plan, code-architect, visual-tester, etc.)       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Layer 2: MCP Servers                     │
│  External tools via Model Context Protocol                  │
│  (browser-mcp, next-conductor, research-agent-mcp)          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Layer 3: SPINE Python                     │
│  Custom orchestration framework                             │
│  (fan_out, pipeline, ToolEnvelope, run_scenario.py)         │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer 1: Claude Native Subagents

These are available via `Task(subagent_type="...")`:

| Subagent | Model | What it does |
|----------|-------|--------------|
| `Explore` | Fast | Codebase exploration, file discovery |
| `Plan` | Sonnet | Architecture planning, implementation design |
| `research-coordinator` | Opus | Multi-source research with synthesis |
| `code-architect` | Sonnet | System design, architectural decisions |
| `visual-tester` | Haiku | UI verification via browser automation |
| `context-engineer` | Sonnet | Context stack design and optimization |
| `general-purpose` | Default | Complex multi-step tasks |

Works out of the box with Claude Code. Subagents can access conversation context and run in parallel.

---

## Layer 2: MCP Servers

| Server | Tools | What for |
|--------|-------|----------|
| `browser-mcp` | navigate, screenshot, click, type | Visual UI testing |
| `next-conductor` | read_next, update_next, init_next | Task tracking |
| `research-agent-mcp` | clarify, search, evaluate, synthesize | Research workflows |
| `research-notes-mcp` | parse, cluster, contradictions | Note processing |
| `research-log-mcp` | create, log, cite | Citation management |
| `smart-inventory` | analyze_project | CLAUDE.md generation |

```
Claude Code ←→ MCP Protocol ←→ External Tools
                   │
                   ├── File Systems
                   ├── Browsers (Playwright)
                   ├── Task Management
                   └── Research Workflows
```

---

## Layer 3: SPINE Python

| Component | What it does |
|-----------|--------------|
| `run_scenario.py` | Execute reproducible context stack scenarios |
| `fan_out()` | Parallel subagent execution with aggregation |
| `pipeline()` | Sequential multi-step processing |
| `ToolEnvelope` | Instrumented LLM calls with trace correlation |
| `TraceScope` | Hierarchical context management |
| `LogAggregator` | Query and analyze execution logs |

### Directory Layout

```
spine/
├── _protocols/               # Usage protocols - read first
│   └── tiered-spine-usage.md
├── _templates/               # Templates for other projects
├── .claude/agents/           # Subagent definitions
├── KB/                       # Knowledge base
├── spine/                    # Core Python package
│   ├── core/                 # ToolEnvelope, TraceScope
│   ├── logging/              # Structured JSON logging
│   ├── client/               # Provider configs
│   └── patterns/             # fan_out, pipeline
├── tools/                    # Applications built on SPINE
├── scripts/                  # Scripts and tests
├── scenarios/                # Scenario configs (.yaml)
├── logs/                     # Structured logs
└── run_scenario.py           # Main entry point
```

---

## Context Stacks

SPINE uses hierarchical context stacks for consistent LLM interactions.

```
Context Stack
├── global      → operator, brand identity
├── character   → speaker persona, target audience
├── command     → task specification, success criteria
├── constraints → tone, format, do/don't rules
├── context     → background info, references
└── input       → user request
```

| Layer | What it holds |
|-------|---------------|
| `global` | System-level config - operator, brand identity |
| `character` | Agent persona and target audience |
| `command` | Task specification and success criteria |
| `constraints` | Tone, format, do/don't rules |
| `context` | Background info and references |
| `input` | The actual user request |

---

## Core Components

### ToolEnvelope

Every LLM call gets wrapped in a ToolEnvelope with:
- **id** - unique correlation ID
- **tool** - provider and model (e.g., "anthropic:claude-opus-4-5")
- **trace** - hierarchical linking (root_id → parent_id → span_id)
- **metadata** - tags, sandbox profile, retry policy

```
envelope = create_envelope(tool: "claude-opus", prompt: "...")
child = envelope.create_child(tool: "claude-sonnet")  // auto-linked
```

### TraceScope

Automatic trace propagation across agent hierarchies:

```
with TraceScope("orchestrator"):
    // calls inherit this trace

    with TraceScope("subagent-research"):
        // linked as child of orchestrator
```

---

## Multi-Provider Support

SPINE abstracts away provider differences:

| Provider | Models | Capabilities |
|----------|--------|--------------|
| Anthropic | Claude Opus 4.5, Sonnet 4.5, Haiku 4.5 | Full tool use, vision |
| OpenAI | GPT-5.1, GPT-5 mini | Tool use, vision |
| Google | Gemini 3 Pro, Gemini 3 Flash | Tool use, vision |
| xAI | Grok | Tool use |

---

## Observability

Logs go to `./logs/YYYY-MM-DD/*.json` with:
- Full envelope with trace hierarchy
- Token usage and cost estimates
- Timing (started_at, finished_at, duration_ms)
- Experiment tracking IDs

### Trace Hierarchy

```
root_id: "session-abc123"
├── parent_id: "orchestrator-001"
│   ├── span_id: "subagent-research-1"
│   ├── span_id: "subagent-research-2"
│   └── span_id: "subagent-research-3"
└── parent_id: "orchestrator-002"
    └── span_id: "synthesis-001"
```

---

## Related Docs

- [Tiered Enforcement Protocol](tiered-enforcement.md) - when to use each capability level
- [Pattern Guide](patterns.md) - fan-out and pipeline usage

[← Back to Main](../README.md)
