# Documentation

## Core Documentation

- [Blueprint Implementation](blueprint-implementation.md) - How SPINE implements the Multi-Agent Playbook
- [Architecture Overview](architecture.md) - System design and components
- [Tiered Enforcement Protocol](tiered-enforcement.md) - When to use each capability level
- [Pattern Guide](patterns.md) - Fan-out and Pipeline usage

## New in v0.3.29-v0.3.30

- [Agent OS 2026](agent-os.md) - OODA loop composition, episodic memory, agent processes, task DAGs
- [Memory System](memory-system.md) - 5-tier memory architecture unified by MemoryFacade

## New in v0.3.28

- [MCP Session Pool & Self-Description](mcp-session-pool.md) - Persistent MCP sessions + 4-layer self-description generator

## New in v0.3.27

- [SmallLLMExecutor](small-llm-executor.md) - Orchestrate 3B-8B quantized models via MCP self-description layers

## New in v0.3.26

- [Dynamic Routing](dynamic-routing.md) - Automatic task classification and executor selection by type

## New in v0.3.22

- [Minna Memory Integration](minna-memory-integration.md) - Persistent cross-session memory with graceful fallback

## New in v0.3.21

- [MCP Orchestrator Integration](mcp-orchestrator-integration.md) - Optional intelligent tool routing with graceful fallback

## New in v0.3.20

- [Executor Framework](executors.md) - SubagentExecutor, ClaudeCodeExecutor, and MCPOrchestratorExecutor
- [Context Stack Integration](context-stacks.md) - YAML scenario files for prompt building
- [Agent Harness Automation](claude-code-automation.md) - Disable prompts, auto-reload after compaction (Claude Code)

## Reference Materials

The **Multi-Agent Playbook** PDF is available in the `KB/` folder:
- [`KB/Multi-Agent-Playbook-Blueprint.pdf`](../KB/Multi-Agent-Playbook-Blueprint.pdf)

This architectural blueprint on production-ready multi-agent systems (from Anthropic's research) serves as one of the foundations for SPINE's design patterns.

## Quick Links

| Topic | Document |
|-------|----------|
| **New to SPINE?** | Start with [Blueprint Implementation](blueprint-implementation.md) |
| **System Design** | [Architecture Overview](architecture.md) |
| **Capability Usage** | [Tiered Enforcement Protocol](tiered-enforcement.md) |
| **Orchestration Patterns** | [Pattern Guide](patterns.md) |
| **Executors** | [Executor Framework](executors.md) |
| **Context Stacks** | [Context Stack Integration](context-stacks.md) |
| **MCP Orchestrator** | [MCP Orchestrator Integration](mcp-orchestrator-integration.md) |
| **Agent OS 2026 (NEW)** | [Agent OS 2026](agent-os.md) |
| **Memory System (NEW)** | [Memory System](memory-system.md) |
| **Dynamic Routing** | [Dynamic Routing](dynamic-routing.md) |
| **SmallLLMExecutor** | [SmallLLMExecutor](small-llm-executor.md) |
| **MCP Session Pool** | [MCP Session Pool](mcp-session-pool.md) |
| **Minna Memory** | [Minna Memory Integration](minna-memory-integration.md) |
| **Automation** | [Agent Harness Automation](claude-code-automation.md) |

[← Back to Main](../README.md)
