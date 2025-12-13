# SPINE Architecture

## Overview

SPINE (Multi-Agent Orchestration System) is built on a three-layer architecture that provides comprehensive multi-agent coordination capabilities. Each layer serves a distinct purpose in the orchestration framework.

## Three Capability Layers

### Layer 1: Claude Native

The foundational layer leverages Claude's native capabilities for intelligent task decomposition and execution:

- **Task Tool** - Primary orchestration interface with specialized modes:
  - `Explore` - Discovery and reconnaissance of codebases and requirements
  - `Plan` - Strategic planning and workflow design
  - `code-architect` - System design and architectural decisions
- **Visual Testing Subagents** - Automated UI validation and screenshot-based verification

This layer provides the intelligent decision-making core that drives the orchestration process.

### Layer 2: MCP Servers

The middle layer consists of specialized Model Context Protocol (MCP) servers that extend capabilities:

- **browser-mcp** - Web automation and browser interaction capabilities
  - Page navigation and interaction
  - Form filling and testing
  - Visual regression testing
- **next-conductor** - Next.js framework orchestration
  - Project scaffolding
  - Component generation
  - Build and deployment coordination
- **research-agent-mcp** - Information gathering and analysis
  - Web research and data collection
  - Documentation synthesis
  - Knowledge base queries

These servers provide domain-specific functionality while maintaining standardized interfaces.

### Layer 3: SPINE Python

The execution layer implements core orchestration patterns and instrumentation:

- **fan_out** - Parallel execution pattern for independent tasks
  - Concurrent agent execution
  - Result aggregation
  - Error isolation
- **pipeline** - Sequential execution pattern for dependent tasks
  - Step-by-step workflow execution
  - State passing between stages
  - Conditional branching
- **ToolEnvelope** - Instrumentation wrapper for full traceability
  - Execution timing and metrics
  - Input/output capture
  - Error tracking
- **TraceScope** - Hierarchical context management
  - Parent-child trace relationships
  - Context propagation
  - Distributed tracing support

## Context Stack Structure

SPINE maintains reproducible context stacks for debugging and optimization:

```json
{
  "trace_id": "abc123",
  "parent_trace_id": "xyz789",
  "operation": "fan_out",
  "timestamp": "2025-12-13T13:00:00Z",
  "inputs": {
    "tasks": ["task1", "task2", "task3"]
  },
  "outputs": {
    "results": ["result1", "result2", "result3"]
  },
  "metadata": {
    "duration_ms": 1250,
    "provider": "anthropic",
    "model": "claude-3-5-sonnet"
  }
}
```

## Core Patterns

### Fan-Out Pattern
Used for parallel execution when tasks are independent:
- Distributes work across multiple agents simultaneously
- Reduces overall execution time
- Ideal for testing, analysis, and data gathering tasks

### Pipeline Pattern
Used for sequential execution when tasks have dependencies:
- Ensures proper ordering of operations
- Passes context between stages
- Ideal for build, deploy, and multi-step workflows

## Integration

The three layers work together seamlessly:
1. **Claude Native** makes intelligent decisions about task decomposition
2. **MCP Servers** provide specialized capabilities as needed
3. **SPINE Python** executes patterns with full instrumentation

This architecture enables complex software development workflows while maintaining complete traceability and reproducibility.
