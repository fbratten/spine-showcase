# Agent OS 2026 (v0.3.29-v0.3.30)

Agent OS 2026 introduces a structured cognition layer to SPINE: an OODA-based execution loop, episodic memory, embedding providers, task DAGs, and agent process management. These components compose existing SPINE subsystems into a coherent autonomous agent runtime.

---

## OODA Loop Composition

The core of Agent OS is a 5-phase OODA cycle that wires existing SPINE components into a structured observe-orient-decide-act-reflect loop.

```
+-------------------------------------------------------------------+
|                          OODALoop                                  |
|                                                                    |
|   +----------+   +----------+   +----------+   +----------+       |
|   | OBSERVE  |-->|  ORIENT  |-->|  DECIDE  |-->|   ACT    |       |
|   +----------+   +----------+   +----------+   +----------+       |
|        ^                                             |             |
|        |              +----------+                   |             |
|        +--------------| REFLECT  |<------------------+             |
|                       +----------+                                 |
|                                                                    |
|   LoopContext: phase tracking, iteration count, cycle mgmt         |
|   WorldState: unified environment view (facade over stores)        |
|   Outcome: canonical result schema from any action                 |
+-------------------------------------------------------------------+
```

### Phase Mapping

Each OODA phase maps to an existing SPINE component:

| Phase | Responsibility | SPINE Component |
|-------|---------------|-----------------|
| **Observe** | Gather current state from environment, memory, and task queue | WorldState, MemoryFacade, TaskQueue |
| **Orient** | Analyze observations, identify patterns, assess priorities | ContextStack, EpisodicMemory (recall) |
| **Decide** | Select next action based on orientation | TaskTypeRouter, TieredEnforcement |
| **Act** | Execute the chosen action via appropriate executor | Executor framework (7 executors) |
| **Reflect** | Evaluate outcome, update memory, detect oscillation | OscillationTracker, EpisodicMemory (store) |

### Core Classes

```python
from spine.agent_os.ooda import OODALoop, OODAConfig, OODACycle, OODAResult
from spine.agent_os.ooda import LoopContext, OODAPhase

# Configure the loop
config = OODAConfig(
    max_iterations=50,
    reflect_every_n=5,        # Full reflection every 5 cycles
    timeout_seconds=300,
    oscillation_threshold=3,
)

# Create and run
loop = OODALoop(config=config, executors=executor_registry)
result: OODAResult = await loop.run(initial_task)

# Access cycle history
for cycle in result.cycles:
    print(f"Phase: {cycle.phase}, Duration: {cycle.duration_ms}ms")
```

**OODAPhase** is an enum: `OBSERVE`, `ORIENT`, `DECIDE`, `ACT`, `REFLECT`.

### LoopContext

Tracks state across iterations within a single OODA run:

```python
from spine.agent_os.ooda import LoopContext

ctx = LoopContext()
ctx.phase           # Current OODAPhase
ctx.iteration       # Current iteration count
ctx.cycle_history   # List of completed OODACycle objects
ctx.elapsed_ms      # Total elapsed time
```

### WorldState and WorldSnapshot

`WorldState` provides a unified facade over all environment data sources. A `WorldSnapshot` is an immutable point-in-time capture used during the Observe phase.

```python
from spine.agent_os.world import WorldState, WorldSnapshot

world = WorldState(
    memory=memory_facade,
    task_queue=task_queue,
    health=health_monitor,
)

# Capture current state (immutable snapshot)
snapshot: WorldSnapshot = world.snapshot()
snapshot.pending_tasks    # Tasks awaiting execution
snapshot.recent_outcomes  # Last N action results
snapshot.memory_context   # Relevant memories for current goal
```

### Outcome

A canonical result schema returned by any action in the Act phase:

```python
from spine.agent_os.outcome import Outcome

outcome = Outcome(
    success=True,
    action="execute_task",
    result={"changes": ["file_a.py", "file_b.py"]},
    tokens_used=1250,
    duration_ms=3400,
    metadata={"executor": "subagent", "tier": 2},
)
```

---

## Episodic Memory

The 5th memory tier. Goal-based recall of past execution episodes, backed by SQLite with full-text search (FTS5).

```python
from spine.memory.episodic import EpisodicMemory, Episode, EpisodeEvent

episodic = EpisodicMemory(db_path="memory/episodes.db")

# Record an episode
episode = Episode(
    goal="Implement authentication module",
    events=[
        EpisodeEvent(action="analyze", result="3 files need changes", phase="orient"),
        EpisodeEvent(action="implement", result="auth.py created", phase="act"),
        EpisodeEvent(action="test", result="all tests pass", phase="reflect"),
    ],
    outcome=Outcome(success=True, action="implement_auth", result={}),
    tags=["authentication", "security"],
)
episodic.store(episode)

# Goal-based recall
similar = episodic.recall(goal="Add OAuth support", limit=5)
# Returns episodes with similar goals, ranked by relevance

# Full-text search across all episode content
results = episodic.search("authentication failure handling")
```

Episodic Memory integrates with the OODA Reflect phase: after each action cycle, the loop stores an episode capturing what was attempted, what happened, and what was learned. During Orient, past episodes with similar goals are recalled to inform the current decision.

---

## Memory System Overview

Agent OS completes SPINE's memory architecture with 5 tiers unified by `MemoryFacade`:

| Tier | Class | Scope | Backend |
|------|-------|-------|---------|
| 1 | `KVStore` | Namespace-scoped key-value | SQLite or file |
| 2 | `Scratchpad` | Short-term task notes | In-memory |
| 3 | `EphemeralMemory` | Session-scoped with decay | In-memory |
| 4 | `VectorStore` | Hybrid semantic + keyword search | LanceDB + keyword |
| 5 | `EpisodicMemory` | Goal-based episode recall | SQLite + FTS5 |

```python
from spine.memory.facade import MemoryFacade

facade = MemoryFacade(
    kv=kv_store,
    scratchpad=scratchpad,
    ephemeral=ephemeral_memory,
    vector=vector_store,
    episodic=episodic_memory,
)

# Unified search across all tiers
results = facade.search("authentication patterns", top_k=10)
```

**[Full Memory System Guide](memory-system.md)**

---

## Embedding Providers

7 embedding providers support the VectorStore and EpisodicMemory tiers:

| Provider | Class | Use Case |
|----------|-------|----------|
| Local (SentenceTransformers) | `LocalEmbeddingProvider` | Offline, privacy-sensitive workloads |
| OpenAI | `OpenAIEmbeddingProvider` | High-quality embeddings via API |
| Voyage AI | `VoyageEmbeddingProvider` | Code-optimized embeddings |
| ONNX Runtime | `ONNXEmbeddingProvider` | Fast local inference, no PyTorch needed |
| Gemini | `GeminiEmbeddingProvider` | Google ecosystem integration |
| Keyword (fallback) | `KeywordEmbeddingProvider` | Zero-dependency TF-IDF fallback |
| Placeholder | `PlaceholderEmbeddingProvider` | Testing and development |

All providers implement the `EmbeddingProvider` abstract base class:

```python
from spine.memory.embeddings import EmbeddingProvider

class EmbeddingProvider(ABC):
    @abstractmethod
    def embed(self, texts: list[str]) -> list[list[float]]:
        """Generate embeddings for a list of texts."""
        ...

    @abstractmethod
    def dimension(self) -> int:
        """Return the embedding dimension."""
        ...
```

Provider selection is automatic based on available dependencies, with `KeywordEmbeddingProvider` as the universal fallback.

---

## Agent Processes

`AgentProcess` and `ProcessManager` provide subprocess lifecycle management for long-running agent tasks.

```python
from spine.agent_os.process import AgentProcess, ProcessManager, ProcessStatus

manager = ProcessManager()

# Start an agent process
process: AgentProcess = manager.spawn(
    name="research-worker",
    executor=research_executor,
    task=research_task,
)

# Monitor status
process.status        # ProcessStatus enum: RUNNING, COMPLETED, FAILED, CANCELLED
process.pid           # Process identifier
process.elapsed_ms    # Time since start

# Manage lifecycle
manager.list_active()        # All running processes
manager.cancel(process.pid)  # Cancel a specific process
manager.wait_all(timeout=60) # Wait for all to complete
```

**ProcessStatus** values: `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED`.

---

## Task DAG

The task system supports dependency graphs with topological ordering, cycle detection, and priority-based scheduling.

```python
from spine.orchestrator.task_queue import TaskQueue, FileTaskQueue, MemoryTaskQueue
from spine.orchestrator.task_queue import Task

# In-memory task queue with DAG support
queue = MemoryTaskQueue()

# Add tasks with dependencies
task_a = Task(id="analyze", description="Analyze codebase", priority=1)
task_b = Task(id="implement", description="Implement changes", priority=2, depends_on=["analyze"])
task_c = Task(id="test", description="Run tests", priority=3, depends_on=["implement"])
task_d = Task(id="docs", description="Update docs", priority=2, depends_on=["analyze"])

queue.add(task_a)
queue.add(task_b)
queue.add(task_c)
queue.add(task_d)

# Get next ready task (all dependencies satisfied)
ready = queue.get_next_ready()  # Returns "analyze" (no deps)

# After completing "analyze":
queue.mark_complete("analyze")
ready = queue.get_next_ready()  # Returns "implement" or "docs" (both unblocked)
```

**Features:**
- **Dependency resolution:** `get_next_ready()` returns only tasks whose dependencies are all satisfied
- **Cycle detection:** Adding a dependency that would create a cycle raises an error
- **Priority sorting:** Among ready tasks, lower priority number executes first
- **Two backends:** `FileTaskQueue` (persistent, file-backed) and `MemoryTaskQueue` (in-memory)

Both implement the `TaskQueue` abstract base class:

```python
from spine.orchestrator.task_queue import TaskQueue

class TaskQueue(ABC):
    @abstractmethod
    def add(self, task: Task) -> None: ...

    @abstractmethod
    def get_next_ready(self) -> Task | None: ...

    @abstractmethod
    def mark_complete(self, task_id: str) -> None: ...
```

---

## Architecture: How OODA Wraps Existing Components

```
+----------------------------------------------------------------------+
|                         Agent OS Runtime                              |
|                                                                       |
|  +----------------------------------------------------------------+  |
|  |                        OODALoop                                 |  |
|  |                                                                 |  |
|  |  OBSERVE --> ORIENT --> DECIDE --> ACT --> REFLECT              |  |
|  |     |          |          |         |         |                 |  |
|  |     v          v          v         v         v                 |  |
|  |  WorldState  Context   TaskType   Executor  Episodic           |  |
|  |  (facade)    Stack     Router     Framework Memory              |  |
|  +-----+----------+----------+---------+---------+----------------+  |
|        |          |          |         |         |                    |
|  +-----v----------v----------v---------v---------v----------------+  |
|  |                    Existing SPINE Layer                          |  |
|  |                                                                  |  |
|  |  MemoryFacade    ContextStack    TieredEnforcement               |  |
|  |  (5 tiers)       (YAML scenarios) (3-tier protocol)             |  |
|  |                                                                  |  |
|  |  ToolEnvelope    TraceScope      InstrumentedLLMClient          |  |
|  |  (tracing)       (hierarchy)     (4 providers)                  |  |
|  |                                                                  |  |
|  |  TaskQueue       Executors (7)   MCPSessionPool                 |  |
|  |  (DAG-aware)     + TaskTypeRouter (persistent sessions)         |  |
|  |                                                                  |  |
|  |  EmbeddingProviders (7)          ProcessManager                 |  |
|  +------------------------------------------------------------------+|
+----------------------------------------------------------------------+
```

Agent OS does not replace existing SPINE components. It composes them into a higher-level execution model where the OODA loop provides the cognitive rhythm and existing subsystems handle the actual work.

---

## Version History

| Version | What Changed |
|---------|-------------|
| **0.3.30** | Agent Processes (ProcessManager), Task DAG (dependency resolution, cycle detection) |
| **0.3.29** | OODA Loop, LoopContext, WorldState, Outcome, EpisodicMemory, EmbeddingProviders (7) |

---

[Back to Docs](README.md) | [Memory System](memory-system.md)
