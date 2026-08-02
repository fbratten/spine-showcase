# SPINE recruiter-proof evidence manifest

Verified: 2026-08-02

## Purpose

This directory is the current, bounded recruiter-facing evidence entry for SPINE.
It supplements the larger public showcase without rewriting its historical feature documentation or interactive demos.

## Source and publication pins

- private source repository: `fbratten/spine`
- inspected source pin: `8fa62c1ebfc0c7b65680ef2d89b9953a519e4138`
- public showcase repository: `fbratten/spine-showcase`
- pre-change public showcase pin: `bb2c265252d0caaa634b12ab0254273b26640da7`
- current proof route: `recruiter-proof/index.html`

## Supported claims

- SPINE is a RunContext-governed orchestration runtime and multi-agent backbone.
- The adopted compiled path is `SkillCompiler -> PlanArtifact -> PlanExecutor` through the shared `execute_compiled_plan(...)` entrypoint.
- The adopted transports are the CLI `cmd_execute` path and `POST /api/orchestrator/execute`.
- AgenticLoop, OODA, scenario execution, review, fan-out and pipeline surfaces remain peer/reference surfaces unless separately adopted.
- The source documents a seven-tier memory architecture and a broader MCP/executor ecosystem.
- The public showcase currently contains seven interactive demos.

## Verification classes

- source inspection: current private `main`
- public documentation: current public `main`
- runtime and test receipts: source-reported only for this refresh
- new execution: not performed

## Test boundary

The source README at the inspected pin reports `3650+` tests. This static publication change did not re-run that suite and does not convert the claim into independent acceptance evidence.

## Non-claims

This proof does not claim:

- that every implemented SPINE surface is on the adopted compiled execution path;
- that every optional memory backend is active in one environment;
- that the public showcase is byte-current with every later private-source change;
- that the proof page itself executes SPINE;
- that the source test suite was re-run during publication.

## Public-data policy

The page contains architecture and capability summaries only. It publishes no private prompts, memory records, paths, credentials, agent messages, operational logs or customer-like data.
