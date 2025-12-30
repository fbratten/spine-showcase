# How SPINE Implements the Multi-Agent Playbook

> A detailed mapping between Anthropic's architectural blueprint for production-ready multi-agent systems and SPINE's implementation.

**Reference Document:** [Agent Architecture Blueprint (PDF)](../KB/Agent_Architecture_Blueprint_deckexample_v2_clean.pdf)

---

## Overview

The **Multi-Agent Playbook** is an architectural blueprint that outlines production-ready patterns for multi-agent AI systems. It addresses the fundamental challenge: *How do you manage delegation, state, execution, and failure without creating chaos?*

SPINE (v0.3.17) implements many of these patterns as a practical orchestration framework. This document maps each blueprint concept to its SPINE implementation.

---

## The Core Pattern: General Contractor Model

### Blueprint Definition

The blueprint establishes a **closed-loop system** led by an Orchestrator:

> "You do not prompt your sub-agents. You prompt your primary agent, and the primary agent prompts your sub-agents. They respond back to the primary agent, which synthesizes the information for you."

This is the **General Contractor Model**:
- The Orchestrator holds the "master blueprint"
- Sub-Agents are "specialized trades" (electricians, plumbers)
- Sub-Agents report to the contractor, not the homeowner
- Direct user communication from sub-agents is forbidden

### SPINE Implementation

| Blueprint Concept | SPINE Component | Location |
|-------------------|-----------------|----------|
| Orchestrator | `AgenticLoop` | `spine/orchestrator/loop.py` |
| Sub-Agent spawning | `fan_out()` | `spine/patterns/fan_out.py` |
| Closed-loop reporting | `ToolEnvelope` result wrapping | `spine/core/envelope.py` |
| Master Plan | Context Stack scenarios | `scenarios/*.yaml` |

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

---

## The Minimum Viable Stack

### Blueprint Definition

Four essential components:

| Component | Role |
|-----------|------|
| **Claude Opus 4.5** | Both Orchestrator and all Sub-Agents (uniform high-intelligence) |
| **Claude Code (CLI)** | The "agent harness" execution environment |
| **Built-in Task Tool** | Native tool for spawning sub-agents (`parallel=true`) |
| **Markdown File** | Immutable "master plan" with task-based lists |

### SPINE Implementation

| Blueprint Component | SPINE Equivalent | Notes |
|---------------------|------------------|-------|
| Claude Opus 4.5 | `InstrumentedLLMClient` | Multi-provider support (Anthropic, OpenAI, Gemini) |
| Claude Code | `run_scenario.py` + CLI | Entry point for orchestrated workflows |
| Task Tool | `fan_out()`, `pipeline()` | Custom parallel/sequential execution |
| Markdown File | Context Stacks + `NEXT.md` | 6-layer hierarchical context + task tracking |

SPINE extends the blueprint by supporting multiple LLM providers through a unified interface:

```python
from spine.client import InstrumentedLLMClient

client = InstrumentedLLMClient(
    provider="anthropic",  # or "openai", "gemini"
    model="claude-opus-4-5-20251101"
)
```

---

## The Five Pillars

The blueprint identifies five architectural pillars that support robust multi-agent systems. Here's how SPINE implements each:

### Pillar I: Communication

**Blueprint Principle:** Agents communicate through closed loops and verifiable artifacts.

> "Sub-agents must report exclusively to the Orchestrator, transmitting either high-level summaries or unaltered, verifiable artifacts. Direct user communication is forbidden."

**The Practice:**
- **Information to Condense:** Summaries, status metrics, operational signals
- **Information to Preserve:** Files/assets, test logs, error signals

**SPINE Implementation:**

| Practice | SPINE Component | Details |
|----------|-----------------|---------|
| Summary synthesis | `ConflictResolver` | Extracts consensus from multi-agent outputs |
| Artifact preservation | `ToolEnvelope.result` | Full response wrapped with metadata |
| Error signals | `LoopVerdict.REJECT` | Error state triggers routing |
| Test logs | `spine/logging/json_logger.py` | All calls logged with traces |

```python
# Every LLM call produces a verifiable artifact
envelope = ToolEnvelope.create(
    tool="anthropic:claude-sonnet-4-5",
    arguments={"prompt": "..."}
)

# Result includes full metadata for verification
result = envelope.complete(
    output="...",
    status=ResultStatus.OK,
    metrics=MCPMetrics(latency_ms=1200, tokens_in=500, tokens_out=800)
)
```

**Payoff:** The Orchestrator confirms task completion based on artifact presence and can route based on clear error signals.

---

### Pillar II: Execution

**Blueprint Principle:** Execution is a choice—parallel for speed, sequential for logic.

> "The orchestration pattern must adapt to the task's dependencies. Independent tasks should be parallelized for throughput; interdependent tasks must be sequential to ensure logical integrity."

**The Practice:**

| Mode | When | Use Cases |
|------|------|-----------|
| **Parallel** | Independent tasks | 5 agents testing 5 user stories; simultaneous data aggregation |
| **Sequential** | Strict dependencies | Plan → Build → Host → Test workflow |
| **Hybrid** | Advanced workflows | Sequential master plan with parallel bursts |

**SPINE Implementation:**

```python
# Parallel Execution - fan_out()
from spine.patterns import fan_out, FanOutTask

tasks = [
    FanOutTask(user_message="Review security", component="security-reviewer"),
    FanOutTask(user_message="Review style", component="style-reviewer"),
    FanOutTask(user_message="Review logic", component="logic-reviewer"),
]
result = fan_out(parent_envelope, client, tasks, max_workers=5)

# Sequential Execution - pipeline()
from spine.patterns import pipeline, PipelineStep

steps = [
    PipelineStep(name="plan", system_prompt="You are an architect."),
    PipelineStep(name="build", transform=lambda prev: extract_spec(prev)),
    PipelineStep(name="test", system_prompt="You are a tester."),
]
result = pipeline(parent_envelope, client, steps)
```

**Hybrid Approach in SPINE:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Sequential Master Plan                    │
│                                                              │
│  ┌──────┐    ┌──────┐    ┌──────┐    ┌─────────────────┐   │
│  │ Plan │───▶│Build │───▶│ Host │───▶│      Test       │   │
│  └──────┘    └──────┘    └──────┘    │  ┌───┐ ┌───┐   │   │
│                                       │  │ 1 │ │ 2 │   │   │
│                                       │  ├───┤ ├───┤   │   │
│                                       │  │ 3 │ │ 4 │   │   │
│                                       │  └───┘ └───┘   │   │
│                                       │ (parallel burst)│   │
│                                       └─────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

### Pillar III: Empowerment & Isolation

**Blueprint Principle:** Empower agents with tools, but isolate them in sandboxes.

> "Instead of restricting an agent's tools to ensure safety, provide the 'right tooling' to maximize capability and restrict the environment to contain risk."

**The Practice:**
- **Functional Empowerment:** Comprehensive tool suites, not minimal scripts
- **Environmental Isolation:** Each agent gets its own sandbox (E2B)
- **The Private Workshop:** Total freedom inside, no effect outside

**SPINE Implementation:**

| Concept | SPINE Component | Details |
|---------|-----------------|---------|
| Right Tooling | MCP server integration | browser-mcp, research-agent-mcp, etc. |
| Tool suites | `InstrumentedLLMClient` | Full provider capabilities |
| Isolation | `TraceScope` boundaries | Logical isolation via trace hierarchy |
| Token optimization | `TokenOptimizedClient` | 57-87% savings for 1-6 tools |

```python
# Comprehensive tooling via MCP integration
from spine.integration import TokenOptimizedClient

if should_use_code_execution(tool_count=3):
    client = TokenOptimizedClient()
    result = client.execute_workflow(
        workflow_code="""
        const session = await clarifyQuestion({topic: "AI agents"});
        return await searchSources({research_id: session.id});
        """,
        required_tools=["clarify_question", "search_sources"]
    )
```

**Current Gap:** SPINE doesn't yet implement per-agent E2B sandboxes. Sub-agents share the execution environment. This is identified as a future enhancement opportunity.

---

### Pillar IV: State & Handoffs

**Blueprint Principle:** State lives in the environment, not the agent's memory.

> "To achieve persistence and enable handoffs between agent sessions, you must preserve the work artifacts and the execution environment, not the agent's conversational history."

**The Practice - Four Artifacts:**

| Artifact | Description |
|----------|-------------|
| **Environment Artifact** | The hosted sandbox (E2B) where the app runs |
| **Instructional Artifact** | The "Master Plan" markdown file |
| **Result Artifact** | Summary logs, named assets, error logs |
| **Codebase Artifact** | The source code itself ("brownfield state") |

**The Factory Shift Change:** A new worker doesn't read the outgoing worker's mind. They look at the machinery, check the SOP, and read the logbook.

**SPINE Implementation:**

| Blueprint Artifact | SPINE Equivalent | Location |
|--------------------|------------------|----------|
| Master Plan | Context Stack scenarios | `scenarios/*.yaml` |
| Result Artifacts | Structured logs | `logs/YYYY-MM-DD/*.json` |
| Session state | NEXT.md integration | `spine/orchestrator/next_integration.py` |
| Codebase state | CLAUDE.md | Auto-generated by smart-inventory |

```python
# Automatic session notes on loop completion
from spine.orchestrator import create_next_callbacks

callbacks = create_next_callbacks(project_path)

loop = AgenticLoop(
    project_path=project_path,
    task_queue=queue,
    on_loop_end=callbacks.on_loop_end,  # Appends session notes to NEXT.md
)
```

**Payoff:** Perfect recall and seamless multi-session handoffs. An agent can resume work on a "brownfield codebase" instantly without needing to "remember" what it did previously.

---

### Pillar V: Resilience

**Blueprint Principle:** Resilience is architected through isolation and error routing.

> "A single sub-agent failure must not derail the entire orchestration. The system must contain the 'blast radius' of a failure and have built-in logic for recovery."

**The Practice:**
- **Isolation:** Catastrophic failures contained to single, disposable environments
- **Error Routing:** Test failures route back to Build step (self-correcting loop)

**SPINE Implementation:**

| Concept | SPINE Component | Details |
|---------|-----------------|---------|
| Blast radius containment | `OscillationTracker` | Detects stuck states before runaway |
| Error routing | `LoopVerdict` | ACCEPT/REVISE/REJECT decision logic |
| Self-correcting loop | `AgenticLoop` retry logic | REVISE triggers retry with feedback |
| Failure detection | Pattern matching | A-B-A-B oscillation, repeated errors |

```python
from spine.orchestrator import (
    AgenticLoop, LoopVerdict, OscillationTracker
)

# Oscillation detection prevents infinite loops
tracker = OscillationTracker()
result = tracker.record_error(error_output, files_modified)
if result.oscillating:
    print(f"Oscillation detected: {result.reason}")
    # System escalates or stops

# Verdict system enables self-correction
# REVISE: Retry with feedback (up to max_revisions)
# REJECT: Skip task, escalate
# ACCEPT: Task complete, proceed
```

```
┌──────┐    ┌───────┐    ┌──────┐    ┌──────┐
│ Plan │───▶│ Build │───▶│ Host │───▶│ Test │
└──────┘    └───┬───┘    └──────┘    └──┬───┘
                │                        │
                │    ┌──────────────┐    │
                └────│ REVISE Loop  │◀───┘
                     │ (on failure) │
                     └──────────────┘
```

---

## Context Management: Signal vs. Noise

### Blueprint Definition

> "The Orchestrator holds the signal, Sub-Agents absorb the noise."

The primary strategy for preventing context window overflow is **distributed computing**. The token load is split across many distinct agents, with the Orchestrator acting as a compression filter.

| Agent Type | Context Type | Examples |
|------------|--------------|----------|
| **Orchestrator** | Executive Signal (Low Context) | Master Plan, operational metrics, synthesized outputs, error signals |
| **Sub-Agents** | Execution Noise (High Context) | Raw PDF text, full DOM, detailed logs, environment variables |

This is the **CEO and Department Heads** model: A CEO doesn't read every employee email. They rely on department heads to manage the noise and report back only critical signals.

### SPINE Implementation

SPINE implements this through its **Context Stack** structure and **Tiered Enforcement Protocol**:

```
Context Stack (Orchestrator Signal)
├── global      → operator, brand identity
├── character   → speaker persona, target audience
├── command     → task specification, success criteria
├── constraints → tone, format, do/don't rules
├── context     → background info (compressed)
└── input       → user request

Sub-Agent Context (Execution Noise)
├── Full document content
├── Raw API responses
├── Detailed logs
└── Environment state
```

**Tiered Enforcement** ensures appropriate context distribution:

| Tier | Context Strategy |
|------|------------------|
| Tier 1 | Direct handling (no distribution needed) |
| Tier 2 | Orchestrator + Sub-Agents with summary reporting |
| Tier 3 | Full distributed computing with artifact handoffs |

---

## Performance Metrics

### Blueprint Definition

Four key metrics for agentic systems:

| Metric | Description |
|--------|-------------|
| **Review Velocity** | How quickly can a human confirm the work is correct? |
| **Tool Calls to Completion** | Smarter models are cheaper (Opus in 5 calls beats Sonnet in 10) |
| **Artifact Validation** | Did the required artifact get created? (check filesystem, not chat) |
| **Resource Observability** | "Five Agent Summary" tracks tool uses and token usage |

### SPINE Implementation

| Blueprint Metric | SPINE Component | Details |
|------------------|-----------------|---------|
| Review Velocity | `spine/review/` module | AI-powered code review with verdicts |
| Tool Calls | `spine/logging/json_logger.py` | Full call tracking with timing |
| Artifact Validation | File-based task completion | Check `ai-coordination/tasks/completed/` |
| Observability | `spine/reports/generator.py` | HTML reports with Chart.js visualizations |

```bash
# Generate observability report
python -m spine.reports generate --title "Sprint Report" --days 7

# Output: spine_report_TIMESTAMP.html with:
# - Token usage breakdown
# - Cost estimates per provider
# - Trace tree visualization
# - Tool call frequency charts
```

---

## The Meta-Goal

### Blueprint Definition

> "The goal is not to build the application. It is to build the system that builds the application."

This represents a fundamental shift in perspective:
- **Yesterday:** Prompt engineers talking to a single model
- **Today:** System architects designing resilient, scalable, observable agentic frameworks

The agent becomes the **new compositional unit of engineering**.

### SPINE as Meta-System

SPINE embodies this philosophy. It's not a tool for building one application—it's a **backbone framework** that enables building *many* applications through orchestrated multi-agent workflows.

**Projects built using SPINE:**
- Golden Thread System
- spine-dashboard
- monitoring-and-management-solution
- AI-Human-Project-Admin-Dashboard

**SPINE's value proposition:**

```
┌─────────────────────────────────────────────────────────────┐
│                        SPINE                                 │
│  (The system that builds the application)                   │
├─────────────────────────────────────────────────────────────┤
│  • ToolEnvelope instrumentation                             │
│  • TraceScope hierarchical tracing                          │
│  • fan_out() / pipeline() orchestration                     │
│  • AgenticLoop autonomous execution                         │
│  • Multi-provider LLM support                               │
│  • Structured logging & observability                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────┐
        │         Applications Built              │
        │  • Golden Thread System                 │
        │  • spine-dashboard                      │
        │  • monitoring-solution                  │
        │  • (your next project)                  │
        └─────────────────────────────────────────┘
```

---

## Gaps and Future Opportunities

The blueprint identifies patterns that SPINE could further develop:

| Blueprint Pattern | SPINE Status | Enhancement Opportunity |
|-------------------|--------------|-------------------------|
| E2B Sandbox per Agent | ❌ Not implemented | True blast radius containment via isolated containers |
| Five Agent Summary | 🟡 Partial | Standardized per-agent health metrics format |
| Built-in Task Tool | 🟡 Conceptual | Expose SPINE as MCP tool (`spine.dispatch`) |
| Live Link (Hosted App) | ❌ Not implemented | Auto-deploy for parallel testing |
| Uniform Model Stack | ✅ Implemented | `InstrumentedLLMClient` with multi-provider support |

---

## Quick Reference: Blueprint → SPINE Mapping

| Blueprint Concept | SPINE Implementation |
|-------------------|----------------------|
| Orchestrator | `AgenticLoop` |
| Sub-Agent spawning | `fan_out()`, `pipeline()` |
| Closed-loop reporting | `ToolEnvelope` |
| Master Plan | Context Stack (`scenarios/*.yaml`) |
| Verifiable artifacts | Structured logs (`logs/YYYY-MM-DD/`) |
| Error routing | `LoopVerdict` (ACCEPT/REVISE/REJECT) |
| Oscillation detection | `OscillationTracker` |
| State handoffs | `NEXT.md` integration |
| Context compression | Context Stack layers |
| Observability | `spine/reports/` + `spine/api/` |
| Multi-provider | `InstrumentedLLMClient` |

---

## Related Documentation

- [Architecture Overview](architecture.md) — SPINE system design
- [Pattern Guide](patterns.md) — Fan-out and Pipeline usage
- [Tiered Enforcement Protocol](tiered-enforcement.md) — When to use each capability level

---

## References

- **Blueprint PDF:** `KB/Agent_Architecture_Blueprint_deckexample_v2_clean.pdf`
- **SPINE Repository:** [github.com/fbratten/spine](https://github.com/fbratten/spine) (private)
- **SPINE Showcase:** [fbratten.github.io/spine-showcase](https://fbratten.github.io/spine-showcase/)

---

*Document created: 2025-12-30*  
*SPINE Version: 0.3.17*
