# Pattern Guide

SPINE has two main orchestration patterns:

| Pattern | How it runs | When to use |
|---------|-------------|-------------|
| **Fan-Out** | Parallel | Independent tasks that can run together |
| **Pipeline** | Sequential | Steps that depend on each other |

---

## Fan-Out (Parallel)

Run multiple tasks at once, then combine the results. Good when tasks don't depend on each other.

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

### When to use it

| Scenario | Example |
|----------|---------|
| Multi-source research | Query multiple docs at once |
| Parallel analysis | Security, style, and logic review |
| Independent subtasks | Generate tests for multiple functions |
| Competitive evaluation | Compare approaches |

### Pseudo example

```
tasks = [
    { message: "Analyze source A", agent: "analyst-a" },
    { message: "Analyze source B", agent: "analyst-b" },
    { message: "Analyze source C", agent: "analyst-c" }
]

result = fan_out(parent_envelope, tasks, max_workers: 5)
```

### Cost

```
Fan-out with 3 agents:
├── Base: 1x (orchestrator)
├── Agents: 3x (parallel)
├── Aggregation: 0.5x
└── Total: ~4.5x single-agent cost

Trade-off: more expensive, but faster wall-clock time
```

---

## Pipeline (Sequential)

Process data through stages where each stage feeds the next. Good for transformations that build on each other.

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌──────────┐
│  Stage  │ ──▶ │  Stage  │ ──▶ │  Stage  │ ──▶ │  Stage   │
│    1    │     │    2    │     │    3    │     │    4     │
└─────────┘     └─────────┘     └─────────┘     └──────────┘
  Analyze        Extract        Transform       Synthesize
```

### When to use it

| Scenario | Example |
|----------|---------|
| Document processing | Parse → Extract → Summarize |
| Data transformation | Fetch → Clean → Analyze → Report |
| Build processes | Lint → Test → Build → Deploy |
| Staged analysis | Explore → Plan → Implement → Verify |

### Pseudo example

```
steps = [
    { name: "analyze", prompt: "You are an analyst." },
    { name: "extract", prompt: "Extract key findings." },
    { name: "synthesize", transform: combine_results }
]

result = pipeline(parent_envelope, steps)
```

---

## ToolEnvelope

Both patterns wrap operations in a ToolEnvelope for traceability.

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
│  timing:                                │
│    started_at, finished_at, duration_ms │
│  usage:                                 │
│    input_tokens, output_tokens          │
└─────────────────────────────────────────┘
```

### Trace examples

**Fan-out:**
```
root_id: "research-task-001"
├── span_id: "orchestrator"
│   ├── span_id: "agent-security" (parallel)
│   ├── span_id: "agent-style" (parallel)
│   └── span_id: "agent-logic" (parallel)
└── span_id: "aggregator"
```

**Pipeline:**
```
root_id: "document-process-001"
├── span_id: "stage-1-analyze"
├── span_id: "stage-2-extract"
├── span_id: "stage-3-transform"
└── span_id: "stage-4-synthesize"
```

---

## Combining Patterns

### Fan-out inside a pipeline

Run parallel tasks as one stage:

```
┌─────────┐     ┌─────────────────────────┐     ┌─────────┐
│ Prepare │ ──▶ │      Fan-Out Stage      │ ──▶ │Synthesize│
└─────────┘     │  ┌───┐ ┌───┐ ┌───┐     │     └─────────┘
                │  │ A │ │ B │ │ C │     │
                │  └───┘ └───┘ └───┘     │
                └─────────────────────────┘
```

### Pipelines inside fan-out

Run independent pipelines in parallel:

```
                ┌───────────────────┐
        ┌──────▶│ Pipeline 1: Docs  │──────┐
        │       └───────────────────┘      │
┌───────┴───┐   ┌───────────────────┐  ┌───┴───────┐
│  Dispatch │──▶│ Pipeline 2: Code  │──▶│ Aggregate │
└───────┬───┘   └───────────────────┘  └───┬───────┘
        │       ┌───────────────────┐      │
        └──────▶│ Pipeline 3: Tests │──────┘
                └───────────────────┘
```

---

## Which Pattern?

| Factor | Fan-Out | Pipeline |
|--------|---------|----------|
| Task independence | High | Low |
| Order matters | No | Yes |
| Speed priority | Wall-clock time | Throughput |
| Error isolation | Good | Cascading risk |
| Debugging | Parallel traces | Linear flow |

```
Fan-Out when:
├── Tasks are independent
├── Need fastest wall-clock time
├── Results can be merged
└── Partial results OK

Pipeline when:
├── Steps depend on each other
├── Order matters
├── Data transforms sequentially
└── Need checkpoints
```

---

## Logging

Everything logs to `./logs/YYYY-MM-DD/*.json`:
- Full envelope with trace hierarchy
- Token usage and cost estimates
- Timing
- Experiment IDs

---

## Tips

### Fan-out
1. Set reasonable max_workers
2. Handle partial failures
3. Don't go overboard on parallelism - cost adds up
4. Use meaningful component names for debugging

### Pipeline
1. Make stages idempotent for safe retries
2. Validate between stages
3. Log intermediate results
4. Use transform functions to shape data

### Both
1. Start simple
2. Watch costs
3. Always use ToolEnvelope
4. Test failure modes

---

## Related Docs

- [Architecture Overview](architecture.md) - system design and components
- [Tiered Enforcement Protocol](tiered-enforcement.md) - when to use each capability level

[← Back to Main](../README.md)
