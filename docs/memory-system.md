# Memory System (v0.4.0)

SPINE provides a 7-tier memory architecture unified by `MemoryFacade`. Each tier serves a different temporal scope and access pattern, from fast key-value lookups to graph-based relationship traversal across projects.

---

## Architecture

```
+-------------------------------------------------------------------------------------+
|                              MemoryFacade                                            |
|               Unified search across all 7 tiers                                      |
+--------+--------+--------+--------+--------+-----------+-----------+----------------+
| Tier 1 | Tier 2 | Tier 3 | Tier 4 | Tier 5 |  Tier 6   |  Tier 7   |                |
|KVStore |Scratch |Ephemer | Vector |Episodic|DeepMemory | Graph     | Federated      |
|        |  pad   |  al    | Store  |Memory  |  Store    | Memory    | Memory         |
|        |        |        |        |        |           |           |                |
|key=val |task    |session |semantic|goal-   |PostgreSQL | graph     | cross-project  |
|by ns   |notes   |w/ decay|+keyword|based   |+ pgvector | traversal | Minna queries  |
|        |        |        |        |recall  |           | + analytics|               |
|        |        |        |        |        |           |           |                |
|SQLite/ |in-mem  |in-mem  |LanceDB |SQLite  |PostgreSQL | PostgreSQL| MCP session    |
|File    |        |        |+keyword|+ FTS5  |+ pgvector | + NetworkX| pool fan-out   |
+--------+--------+--------+--------+--------+-----------+-----------+----------------+
                          |                        |
              +-----------+-----------+    +-------+--------+
              |     VerdictRouter     |    |  MemoryHooks   |
              | Routes accept/reject/ |    | OODA orient +  |
              | revise to correct tier|    | reflect wiring |
              +-----------------------+    +----------------+
```

---

## Tier 1: KVStore

Namespace-scoped key-value storage for structured data.

```python
from spine.memory.kv_store import KVStore

kv = KVStore(backend="sqlite", path="memory/kv.db")

# Write with namespace isolation
kv.set("config", "max_retries", "3")
kv.set("config", "timeout_ms", "5000")
kv.set("state", "last_task_id", "TASK-042")

# Read
value = kv.get("config", "max_retries")  # "3"

# List keys in namespace
keys = kv.keys("config")  # ["max_retries", "timeout_ms"]

# Delete
kv.delete("state", "last_task_id")
```

**Backends:** `SQLitePersistence` (default), `FilePersistence` (one file per namespace).

**Best for:** Configuration, counters, state flags, small structured data.

---

## Tier 2: Scratchpad

Short-term task notes that live for the duration of a task or subtask.

```python
from spine.memory.scratchpad import Scratchpad

pad = Scratchpad()

# Jot notes during task execution
pad.write("plan", "Need to refactor auth module first")
pad.write("blockers", "Waiting on API key for external service")

# Read back
note = pad.read("plan")

# Clear when task is done
pad.clear()
```

**Storage:** In-memory only. Not persisted across sessions.

**Best for:** Working memory during multi-step task execution, temporary notes, intermediate results.

---

## Tier 3: EphemeralMemory

Session-scoped memory with time-based decay. Entries lose relevance over time and are automatically pruned.

```python
from spine.memory.ephemeral import EphemeralMemory

eph = EphemeralMemory(decay_rate=0.1, prune_threshold=0.2)

# Store with automatic timestamping
eph.store("User prefers verbose output", tags=["preference"])
eph.store("Auth endpoint returns 401 for expired tokens", tags=["api", "auth"])

# Retrieve (scores decay over time)
results = eph.recall(tags=["auth"])
# [{"text": "Auth endpoint...", "score": 0.85, "age_seconds": 120}]

# Prune low-relevance entries
eph.prune()
```

**Storage:** In-memory with decay function. Cleared at session end.

**Best for:** Recent observations, conversational context, facts that matter now but not tomorrow.

---

## Tier 4: VectorStore

Hybrid semantic and keyword search over persistent document collections.

```python
from spine.memory.vector_store import VectorStore

vs = VectorStore(
    path="memory/vectors",
    embedding_provider=embedding_provider,  # Any of the 7 providers
)

# Index documents
vs.add("doc-001", text="Authentication uses JWT tokens with 1-hour expiry", tags=["auth"])
vs.add("doc-002", text="Rate limiting is set to 100 requests per minute", tags=["api"])

# Semantic search
results = vs.search("How long are auth tokens valid?", top_k=5)

# Keyword search (fallback when no embedding provider)
results = vs.keyword_search("JWT expiry")

# Hybrid search (combines semantic + keyword scores)
results = vs.hybrid_search("token expiration policy", top_k=5)
```

**Backends:** LanceDB for vector storage, keyword index for fallback/hybrid mode.

**Embedding providers:** 7 providers available (Local, OpenAI, Voyage, ONNX, Gemini, Keyword, Placeholder). See [Agent OS](agent-os.md#embedding-providers) for details.

**Best for:** Knowledge bases, documentation search, semantic retrieval over large corpora.

---

## Tier 5: EpisodicMemory

Goal-based recall of past execution episodes. Stores structured records of what was attempted, what happened, and what was learned.

```python
from spine.memory.episodic import EpisodicMemory, Episode, EpisodeEvent

episodic = EpisodicMemory(db_path="memory/episodes.db")

# Store a complete episode
episode = Episode(
    goal="Fix failing CI pipeline",
    events=[
        EpisodeEvent(action="diagnose", result="Test timeout in auth module", phase="orient"),
        EpisodeEvent(action="fix", result="Increased timeout, added retry", phase="act"),
        EpisodeEvent(action="verify", result="CI green after fix", phase="reflect"),
    ],
    outcome=Outcome(success=True, action="fix_ci", result={"files_changed": 2}),
    tags=["ci", "testing", "auth"],
)
episodic.store(episode)

# Goal-based recall (finds episodes with similar goals)
similar = episodic.recall(goal="Debug intermittent test failures", limit=5)

# Full-text search across all episode content
results = episodic.search("timeout auth")
```

**Backend:** SQLite with FTS5 for full-text search.

**Best for:** Learning from past actions, avoiding repeated mistakes, recalling solutions to similar problems.

---

## Tier 6: DeepMemoryStore (v0.4.0)

PostgreSQL + pgvector backed persistent memory for long-term semantic recall. Opt-in: requires PostgreSQL with the pgvector extension. Falls back gracefully (returns empty results) when unavailable.

```python
from spine.memory.deep_store import DeepMemoryStore
from spine.memory.deep_config import DeepStoreConfig

config = DeepStoreConfig(
    enabled=True,
    database_url="postgresql://spine:spine@localhost:5432/spine_memory",
    embedding_provider="voyage",       # Reuses SPINE's embedding providers
    confidence_half_life_days=90.0,    # Confidence decays over time
    project_scope="my-project",        # Scoping for cross-project federation
)
store = DeepMemoryStore(config, embedding_provider=provider)
store.init_schema()

# Store a memory about an entity
store.store_memory("AuthModule", "component", "architecture",
                   "JWT-based with 1-hour token expiry")

# Semantic vector search (pgvector cosine similarity)
results = store.search("token expiration policy", limit=5)

# Entity-scoped recall with confidence decay
memories = store.recall("AuthModule", attribute="architecture")

# Relationship graph
store.link("AuthModule", "UserService", "CALLS")
related = store.get_related("AuthModule", hops=2)

# OODA decision logging for provenance
store.log_decision(
    goal="Fix auth flow", cycle_number=3, phase="act",
    decision={"action": "refactor_jwt"}, outcome={"success": True},
)
```

**Key features:**
- Entity management with typed nodes, aliases, and fuzzy search (pg_trgm)
- Semantic vector search via pgvector HNSW indexes
- Confidence decay using a configurable half-life model
- Relationship graph (directed entity connections with multi-hop traversal)
- OODA decision audit trail
- Project scoping for cross-project federation
- Batch embedding sync for deferred embedding generation

**Best for:** Long-term knowledge persistence, cross-session semantic recall, decision provenance.

---

## Tier 7: GraphMemory (v0.4.0)

Graph traversal and analytics layer over DeepMemoryStore's relationship tables. Provides advanced graph operations without requiring a dedicated graph database.

```python
from spine.memory.graph_memory import GraphMemory

graph = GraphMemory(deep_store=store)

# Shortest path between entities (BFS via recursive CTE)
path = graph.shortest_path("SPINE", "MemoryFacade")
# GraphPath(nodes=["SPINE", "Orchestrator", "MemoryFacade"], total_weight=2)

# N-hop neighborhood extraction
hood = graph.neighborhood("AuthModule", hops=2)
# GraphNeighborhood with all nodes and edges within 2 hops

# Centrality analysis
central = graph.central_entities(metric="degree", top_k=5)
# [{"name": "SPINE", "degree": 42, "metric": "degree"}, ...]

# Betweenness/closeness centrality (requires NetworkX)
central = graph.central_entities(metric="betweenness", top_k=5)

# Connected components and entity clustering
components = graph.connected_components()
clusters = graph.entity_clusters(min_cluster_size=3)

# Subgraph extraction
sub = graph.subgraph(["AuthModule", "UserService", "TokenManager"])

# Graph-level statistics
stats = graph.stats()
# GraphStats(node_count=150, edge_count=320, density=0.014, ...)
```

**Key features:**
- Shortest path via PostgreSQL recursive CTEs (no graph DB needed)
- Neighborhood subgraph extraction with configurable hop depth
- Degree centrality via SQL; betweenness/closeness via NetworkX (optional)
- Connected component detection and density-based clustering
- Operates on the same PostgreSQL tables as DeepMemoryStore

**Best for:** Understanding entity relationships, discovering knowledge clusters, tracing dependency chains.

---

## FederatedMemory (v0.4.0)

Read-only cross-project memory federation. Queries remote Minna MCP memory servers in parallel and returns results with full provenance metadata.

```python
from spine.memory.federated import FederatedMemory, FederatedConfig

config = FederatedConfig(
    servers={
        "project_a_minna": {
            "command": "uv",
            "args_module": "minna_memory.server",
            "cwd": "/path/to/project-a",
        },
        "project_b_minna": {
            "command": "uv",
            "args_module": "minna_memory.server",
            "cwd": "/path/to/project-b",
        },
    },
    max_results_per_server=5,
    timeout_seconds=10,
)

fed = FederatedMemory(config=config)
fed.open()

# Search across all configured servers (parallel fan-out)
result = fed.search("authentication patterns", limit=10)
for hit in result.top(5):
    print(f"[{hit.server}] {hit.entity}.{hit.attribute} = {hit.value}")

# Entity-specific recall across projects
result = fed.recall("AuthModule", attribute="architecture")

fed.close()
```

**Key features:**
- Config-whitelisted server connections (no unconstrained discovery)
- Parallel fan-out via ThreadPoolExecutor
- Budget controls: max servers, max results per server, timeout
- Full provenance metadata (server, project, tool, confidence)
- Graceful degradation: server failures are logged, not propagated
- Local-first: SPINE's internal memory plane is always primary

**Best for:** Cross-project knowledge sharing, discovering related patterns in sibling projects.

---

## MemoryHooks: OODA Integration (v0.4.0)

`MemoryHooks` connects the OODA loop phases to the deep memory subsystem, enriching orientation with recalled knowledge and persisting decisions for provenance.

```python
from spine.memory.hooks import MemoryHooks

hooks = MemoryHooks(deep_store=store, graph_memory=graph, episodic=episodic)

# Orient hook: enriches OODA orientation with deep memory context
enrichment = hooks.orient_hook(goal="analyze auth module", cycle=1)
# OrientEnrichment with deep_memories, graph_context, related_decisions

# Reflect hook: persists OODA decision to deep store
hooks.reflect_hook(
    goal="analyze auth module", cycle=1,
    decision={"action": "scan_dependencies"},
    outcome={"success": True, "files_found": 12},
)

# Episode sync: syncs completed episodes to deep store entities
hooks.episode_sync_hook(episode_id="ep-abc123")
```

**Hook integration points:**
- **Orient phase:** Recalls semantically similar memories, graph neighborhoods of relevant entities, and past OODA decisions for the current goal
- **Reflect phase:** Logs decisions with full context (goal, cycle, phase, outcome) for provenance tracking
- **Episode sync:** Creates deep store entities from completed episodic memory episodes

All hooks degrade gracefully -- if the deep store or graph memory is unavailable, hooks return empty results without raising errors.

---

## MemoryFacade

Unified interface that searches across all 7 tiers and merges results:

```python
from spine.memory.facade import MemoryFacade

facade = MemoryFacade(
    kv=kv_store,
    scratchpad=scratchpad,
    ephemeral=ephemeral_memory,
    vector=vector_store,
    episodic=episodic_memory,
    deep_store=deep_store,          # Tier 6 (v0.4.0)
    graph_memory=graph_memory,      # Tier 7 (v0.4.0)
)

# Search all 7 tiers at once
results = facade.search("authentication", top_k=10)
# Returns ranked results from whichever tiers have relevant data

# Tier-specific access still available
facade.kv.get("config", "timeout_ms")
facade.episodic.recall(goal="Fix auth bug")
```

The facade handles score normalization across tiers using source weights so that results from different backends are comparable. Deep store results use semantic similarity scores, graph results use entity-match relevance, and both are weighted alongside the existing tiers.

---

## VerdictRouter

Routes `LoopVerdict` decisions (from the AgenticLoop evaluator) to the appropriate memory tier:

| Verdict | Action | Target Tier |
|---------|--------|-------------|
| `ACCEPT` | Store successful outcome as episode | EpisodicMemory |
| `REVISE` | Update scratchpad with revision notes | Scratchpad |
| `REJECT` | Log rejection reason, update ephemeral | EphemeralMemory |

```python
from spine.memory.verdict_router import VerdictRouter

router = VerdictRouter(facade=memory_facade)

# After AgenticLoop evaluation
router.route(verdict=loop_verdict, task=current_task, result=executor_result)
```

---

## Persistence Backends

Two persistence backends are available for tiers that support durable storage:

| Backend | Class | Storage | Best For |
|---------|-------|---------|----------|
| SQLite | `SQLitePersistence` | Single `.db` file | Production, atomic writes |
| File | `FilePersistence` | Directory of JSON files | Debugging, human inspection |

```python
from spine.memory.persistence import SQLitePersistence, FilePersistence

# SQLite (default)
sqlite = SQLitePersistence(path="memory/store.db")

# File-based
file = FilePersistence(path="memory/store/")
```

---

## Memory Tier Selection Guide

| Need | Tier | Why |
|------|------|-----|
| Store a config value | KVStore | Fast lookup by key |
| Track intermediate results | Scratchpad | Temporary, task-scoped |
| Remember what just happened | EphemeralMemory | Decays naturally |
| Search documentation/knowledge | VectorStore | Semantic retrieval |
| Learn from past executions | EpisodicMemory | Goal-based recall |
| Long-term semantic knowledge | DeepMemoryStore | Persistent, confidence-decayed, pgvector search |
| Trace entity relationships | GraphMemory | Shortest path, centrality, clustering |
| Query across projects | FederatedMemory | Cross-project Minna federation |
| Search everything at once | MemoryFacade | Cross-tier unified search |

---

[Back to Docs](README.md) | [Agent OS](agent-os.md) | [Deep Memory Guide](deep-memory.md)
