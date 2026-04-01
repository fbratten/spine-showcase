# RunContext: Authority Inversion (v0.5.0)

SPINE v0.5.0 introduces **authority inversion** - a fundamental architectural shift where RunContext becomes the sole source of runtime truth. All core modules now operate ON RunContext rather than alongside it, eliminating dual-truth bugs and simplifying session handoff.

---

## The Problem: Dual Truth

Before v0.5.0, modules like FivePointProtocol returned a ProtocolResult AND could optionally write to RunContext. Two places carried state. When they diverged, which was correct?

```
BEFORE (v0.4.0):
  FivePointProtocol.audit()
    -> returns ProtocolResult (truth #1)
    -> optionally writes to RunContext (truth #2)
    -> downstream code reads from... which one?
```

## The Solution: Authority Inversion

Three rules govern all inverted modules:

1. **If `run_context` provided** - RUNTIME mode: write all state to RunContext, return `None`
2. **If `run_context` is None** - STANDALONE mode: return authoritative Result object (backward compatible)
3. **Never dual-write** - never build BOTH RunContext state AND Result object

```
AFTER (v0.5.0):
  FivePointProtocol.audit(output_metadata, run_context=ctx)
    -> writes findings to ctx
    -> returns None
    -> downstream reads from ctx (sole truth)
```

---

## RunContext State Machine

RunContext carries all state through a strict lifecycle:

```
CREATED -> CLARIFIED -> SCOPED -> PLANNED -> EXECUTING -> VERIFYING -> COMPLETED
                                                 |
                                            FAILED / HALTED / NEEDS_HUMAN
```

Each transition corresponds to a lifecycle method:

| Method | Transition | Artifact Produced |
|--------|-----------|-------------------|
| `ctx.clarify()` | CREATED -> CLARIFIED | ClarificationArtifact |
| `ctx.set_scope()` | CLARIFIED -> SCOPED | ScopeArtifact |
| `SkillCompiler.compile(ctx)` | SCOPED -> PLANNED | PlanArtifact |
| `ctx.begin_execution()` | PLANNED -> EXECUTING | - |
| `ctx.verify()` | EXECUTING -> COMPLETED/FAILED | VerificationArtifact |

---

## Four Inverted Modules

### FivePointProtocol

The Five-Point Protocol (Clarify, Scope, Plan, Execute, Verify) now writes all phases directly to RunContext.

```python
# Runtime mode - RunContext is sole truth
ctx = RunContext.create(task="Audit deployment", source="user")
protocol = FivePointProtocol(constraints)
result = protocol.run_full(task, output_metadata, run_context=ctx)
# result is None - all truth in ctx
assert result is None
assert ctx.verification is not None

# Standalone mode - backward compatible
result = protocol.run_full(task, output_metadata)
# result is authoritative ProtocolResult
assert result.passed is True
```

### VerdictRouter

Routes task results to memory stores (Tier A/B/C) based on verdicts. In runtime mode, writes routing events to RunContext.

```python
# Runtime mode
router.route(task_id="t1", verdict=ACCEPT, content="...", run_context=ctx)
# Returns None - routing truth in ctx.execution_log

# Read from RunContext
events = [e for e in ctx.execution_log if e.event_type == "verdict_routing"]
tier = events[0].data["tier"]  # "TIER_A"
```

### OODALoop

The OODA cycle (Observe-Orient-Decide-Act-Reflect) writes all phase data to RunContext. Key change: **LoopContext is NOT used as truth in runtime mode** - RunContext owns the cycle state.

```python
# Runtime mode
await ooda.run(goal="Analyze system", run_context=ctx)
# Returns None - cycle data in ctx.execution_log

cycles = [e for e in ctx.execution_log if e.event_type == "ooda_cycle"]
```

### ContentPipelineExecutor

The 5-stage content pipeline (Analyze, Bridge, Validate, Refine, Execute, Review) writes stage results in real-time to RunContext via `_write_stage_to_run_context()`.

---

## Cognitive Compiler

v0.5.0 adds a compile-time validation pipeline that catches broken plans before execution.

### SkillCompiler

Compiles RunContext (in CLARIFIED or SCOPED state) plus a SkillRegistry into a PlanArtifact:

1. Build task description from ctx.clarification + ctx.scope
2. Match skills via keyword + embedding similarity
3. **Enforce tool availability** - tool_checker callback verifies MCP tools exist at compile time
4. Include meta-skill (5PP) if configured
5. Flatten into ordered PlanSteps with dependency tracking
6. Topological sort (Kahn's algorithm) for execution order
7. Detect parallel groups for concurrent execution
8. Validate via PlanValidator
9. Write plan to ctx (state -> PLANNED)

```python
compiler = SkillCompiler(
    registry=skill_registry,
    tool_checker=lambda tool: tool in available_mcp_tools,
)
plan = compiler.compile(ctx)  # ctx must be CLARIFIED or SCOPED
# ctx.plan is now set, ctx.state == PLANNED
```

### PlanValidator

Validates every compiled plan against 6 structural checks:

| Check | What It Catches |
|-------|----------------|
| non_empty | Empty plans with no steps |
| no_orphans | Steps depending on non-existent steps |
| dag_acyclic | Circular dependency chains |
| required_skills_ordered | Dependencies executing after dependents |
| policy_consistency | Halt gates placed after irreversible steps |
| optional_output_safety | Required phases consuming optional outputs |

### Policy Tags

Plan steps can carry policy tags (`destructive`, `irreversible`, `approval-gate`) that trigger PolicyDecisions at execution time:

```python
decision = ctx.check_policy()
if decision.verdict == PolicyVerdict.REQUIRE_REVIEW:
    # Human must approve before execution continues
```

### plan_summary()

Unified observability method on RunContext that returns a complete view:

- Skills included and excluded (with reasons and missing tools)
- Step count, tool count, parallel groups
- Policy decision and halt-on-failure steps
- Whether human approval is required
- Optional dependency warnings

---

## Serialization and Session Handoff

All RunContext state serializes to JSON, enabling session handoff:

```python
# Save at end of session
ctx.save("run_001.json")

# Resume in next session
ctx = RunContext.load("run_001.json")
# All artifacts, events, findings, verification preserved
```

---

## Design Principles

1. **Sole Source of Truth** - If RunContext disappears, system breaks. If Result object disappears, system still works.
2. **No Dual Write** - Never build BOTH RunContext state AND Result object when run_context is provided.
3. **Return Value Irrelevance** - In runtime mode, return value is None. Downstream reads from RunContext.
4. **Backward Compatibility** - Standalone mode (no run_context) unchanged. Legacy code still works.
5. **Serialization Preservation** - Round-trip JSON serialization preserves all authority.

---

## Test Coverage

84 acceptance tests verify the authority inversion:

| Test Suite | Tests | Focus |
|-----------|-------|-------|
| test_run_context_wiring.py | 36 | Authority inversion for all 4 modules |
| test_compiler_enforcement.py | 12 | SkillCompiler tool availability |
| test_plan_validation.py | 23 | PlanValidator 6-check validation |
| test_policy_tag_enforcement.py | 13 | Policy tag enforcement |

Key test patterns:
- `assertIsNone(result)` - verifies runtime mode returns None
- No dual-write guards - verifies only RunContext holds truth
- Serialization round-trip - verifies JSON preserves authority
- 9-phase truth audit - comprehensive system integrity check

---

## Version History

| Version | Change |
|---------|--------|
| v0.5.0 | Authority inversion (4 modules), SkillCompiler, PlanValidator, policy tags, plan_summary() |
| v0.4.0 | Deep memory (Tier 6-7), federated memory, OODA hooks |
| v0.3.30 | Agent OS initial: OODA loop, episodic memory, agent processes |

---

[Back to Documentation Index](README.md) | [Architecture Overview](architecture.md) | [Agent OS 2026](agent-os.md)
