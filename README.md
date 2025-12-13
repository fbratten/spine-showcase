# SPINE - Multi-Agent Orchestration System

![License](https://img.shields.io/badge/license-MIT-blue)
![Status](https://img.shields.io/badge/status-active-green)
![Demo](https://img.shields.io/badge/demo-live-blue)

**This is a showcase repository. Source code is maintained privately.**

## Overview

SPINE is a context engineering and multi-agent backbone framework designed for complex software development workflows. It provides standardized instrumentation, multi-provider LLM access, and orchestration patterns that enable sophisticated multi-agent coordination.

## Key Capabilities

• **Multi-agent orchestration** - Fan-out parallel and pipeline sequential patterns for complex workflows  
• **Full traceability** - ToolEnvelope instrumentation for complete execution visibility  
• **Multi-provider LLM support** - Anthropic, OpenAI, Google Gemini, Grok integration  
• **Tiered enforcement protocol** - Balanced capability usage across complexity tiers  
• **Reproducible context stacks** - Hierarchical trace correlation for debugging and optimization  

## Architecture

SPINE operates in three capability layers:

```
┌─────────────────────────────────────────────────────┐
│  Layer 1: Claude Native                             │
│  ├─ Task tool (Explore, Plan, code-architect)       │
│  └─ Visual testing subagents                        │
├─────────────────────────────────────────────────────┤
│  Layer 2: MCP Servers                               │
│  ├─ browser-mcp (web automation)                    │
│  ├─ next-conductor (Next.js orchestration)          │
│  └─ research-agent-mcp (information gathering)      │
├─────────────────────────────────────────────────────┤
│  Layer 3: SPINE Python                              │
│  ├─ fan_out (parallel execution)                    │
│  ├─ pipeline (sequential execution)                 │
│  ├─ ToolEnvelope (instrumentation wrapper)          │
│  └─ TraceScope (hierarchical context management)    │
└─────────────────────────────────────────────────────┘
```

## Interactive Demos

Explore SPINE capabilities through interactive demonstrations:

👉 [Launch Demos](demos/tier-classifier/)

## Documentation

- [Architecture Overview](docs/architecture.md) - Detailed explanation of the three capability layers
- [Tiered Enforcement System](docs/tiered-enforcement.md) - How SPINE manages complexity across tiers

## Contact

For more information or inquiries, visit: [github.com/fbratten](https://github.com/fbratten)

---

*Source code is maintained privately. For inquiries, contact through GitHub.*
