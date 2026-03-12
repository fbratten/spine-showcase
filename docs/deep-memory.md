# Deep Memory (v0.4.0)

Phase 3 Deep Memory extends SPINE's memory system from 5 tiers to 7, adding PostgreSQL-backed persistent semantic memory, graph traversal, cross-project federation, and OODA lifecycle hooks.

---

## Architecture

```
+----------------------------------------------------------------------+
|                          Agent OS + OODA Loop                         |
|                                                                       |
|   OBSERVE --> ORIENT ---------> DECIDE --> ACT --> REFLECT            |
|                  |                                    |                |
|                  v                                    v                |
|         +-----------------+               +-----------------+         |
|         | MemoryHooks     |               | MemoryHooks     |         |
|         | .orient_hook()  |               | .reflect_hook() |         |
|         +---------+-------+               +--------+--------+         |
|                   |                                |                  |
+----------------------------------------------------------------------+
                    |                                |
        +-----------v--------------------------------v-----------+
        |                Deep Memory Plane                       |
        |                                                        |
        |   +-----------------+    +-----------------+           |
        |   | DeepMemoryStore |    |   GraphMemory   |           |
        |   |   (Tier 6)      |    |   (Tier 7)      |           |
        |   |                 |    |                  |           |
        |   | Entities        |    | Shortest path    |           |
        |   | Memories+embed  |    | Neighborhood     |           |
        |   | Relationships   |    | Centrality       |           |
        |   | Decisions       |    | Clustering       |           |
        |   +---------+-------+    +------+-----------+           |
        |             |                   |                       |
        |             +-----+    +--------+                       |
        |                   |    |                                |
        |            +------v----v------+                         |
        |            |   PostgreSQL     |                         |
        |            |   + pgvector     |                         |
        |            +------------------+                         |
        |                                                        |
        |   +------------------------------------------------+   |
        |   |            FederatedMemory                      |   |
        |   |  Cross-project Minna MCP server federation      |   |
        |   |  (read-only, parallel fan-out, config-gated)    |   |
        |   +------------------------------------------------+   |
        +--------------------------------------------------------+
```

---

## Components

### DeepMemoryStore (Tier 6)

PostgreSQL + pgvector backed persistent memory. Stores typed entities, memories with vector embeddings, directed relationships, and OODA decision logs.

**Key capabilities:**

| Capability | Description |
|------------|-------------|
| Entity management | Typed nodes with aliases, fuzzy search via pg_trgm |
| Semantic search | pgvector cosine similarity over HNSW indexes |
| Confidence decay | Configurable half-life model (default 90 days) |
| Relationship graph | Directed entity connections with multi-hop traversal |
| Decision audit trail | OODA phase/cycle/outcome logging |
| Project scoping | `project_scope` field enables cross-project federation |
| Batch embedding sync | Deferred embedding generation for bulk imports |

```python
from spine.memory.deep_store import DeepMemoryStore
from spine.memory.deep_config import DeepStoreConfig

config = DeepStoreConfig(
    enabled=True,
    database_url="postgresql://spine:spine@localhost:5432/spine_memory",
    confidence_half_life_days=90.0,
    project_scope="my-project",
)
store = DeepMemoryStore(config, embedding_provider=provider)
store.init_schema()

# Entity + memory
store.store_memory("AuthModule", "component", "architecture",
                   "JWT-based with 1-hour token expiry", confidence=0.95)

# Semantic search
results = store.search("how do tokens expire?", limit=5)

# Recall with confidence decay
memories = store.recall("AuthModule", min_confidence=0.5)

# Relationships
store.link("AuthModule", "UserService", "CALLS")
related = store.get_related("AuthModule", hops=2)

# OODA decision logging
store.log_decision(
    goal="Fix auth flow", cycle_number=3, phase="act",
    decision={"action": "refactor"}, outcome={"success": True},
)

# Batch sync pending embeddings
count = store.sync_pending_embeddings(batch_size=50)
```

**Opt-in by design:** Disabled by default. All existing SPINE functionality (Tiers 1-5) works without PostgreSQL. When unavailable, DeepMemoryStore returns empty results without errors.

---

### GraphMemory (Tier 7)

Graph traversal and analytics layer over DeepMemoryStore's relationship tables. Uses PostgreSQL recursive CTEs for traversal and optionally NetworkX for in-memory analytics.

**Key capabilities:**

| Capability | Description |
|------------|-------------|
| Shortest path | BFS via recursive CTE with relationship type filtering |
| Neighborhood extraction | N-hop subgraph around any entity |
| Centrality analysis | Degree (SQL), betweenness/closeness (NetworkX) |
| Connected components | Identify isolated knowledge clusters |
| Subgraph extraction | Extract edges between a specific set of entities |
| Entity clustering | Density-based grouping via component analysis |

```python
from spine.memory.graph_memory import GraphMemory

graph = GraphMemory(deep_store=store)

# Path finding
path = graph.shortest_path("SPINE", "MemoryFacade")

# Neighborhood
hood = graph.neighborhood("AuthModule", hops=2, rel_types=["CALLS", "IMPORTS"])

# Centrality
top = graph.central_entities(metric="degree", top_k=10)

# Clustering
clusters = graph.entity_clusters(min_cluster_size=3)

# Statistics
stats = graph.stats()
print(graph.summary())
```

**No graph database required.** GraphMemory operates on the same PostgreSQL tables as DeepMemoryStore. NetworkX is an optional dependency for advanced analytics (betweenness, closeness centrality, connected components).

---

### FederatedMemory

Read-only cross-project memory federation via Minna MCP servers. Queries whitelisted remote servers in parallel and returns results with full provenance.

**Design principles:**
- **Config-gated:** Only servers listed in `FederatedConfig.servers` are queried
- **Read-only:** No cross-project writes (local memory plane is always authoritative)
- **Budget-controlled:** Max servers, max results per server, per-server timeout
- **Graceful degradation:** Individual server failures are logged, not propagated

```python
from spine.memory.federated import FederatedMemory, FederatedConfig

config = FederatedConfig(
    servers={
        "project_a_minna": {
            "command": "uv",
            "args_module": "minna_memory.server",
            "cwd": "/path/to/project-a",
        },
    },
    max_results_per_server=5,
    timeout_seconds=10,
    max_servers=10,
)

fed = FederatedMemory(config=config)
fed.open()

# Parallel search across all servers
result = fed.search("authentication patterns", limit=10)
print(result.summary())
# "Federated: 3 servers queried, 0 failed, 12 hits"

for hit in result.top(5):
    print(f"[{hit.server}/{hit.project}] {hit.entity}.{hit.attribute}")

# Entity-specific recall
result = fed.recall("AuthModule", attribute="architecture")

fed.close()
```

FederatedMemory uses `MCPSessionPool` for persistent MCP connections and `ThreadPoolExecutor` for parallel fan-out.

---

### MemoryHooks

Callback hooks that connect OODA loop phases to the deep memory subsystem. Three hooks are available:

| Hook | Phase | What It Does |
|------|-------|-------------|
| `orient_hook()` | Orient | Recalls deep memories, graph neighborhoods, and past OODA decisions relevant to the current goal |
| `reflect_hook()` | Reflect | Persists the decision and outcome to DeepMemoryStore for provenance tracking |
| `episode_sync_hook()` | Post-episode | Promotes completed episodic memory episodes to deep store entities |

```python
from spine.memory.hooks import MemoryHooks, OrientEnrichment

hooks = MemoryHooks(deep_store=store, graph_memory=graph, episodic=episodic)

# Orient: what does deep memory know about this goal?
enrichment: OrientEnrichment = hooks.orient_hook(
    goal="analyze auth module", cycle=1, top_k=5,
)

# enrichment.deep_memories      -> semantically similar stored memories
# enrichment.graph_context      -> entity neighborhoods for matching entities
# enrichment.related_decisions  -> past OODA decisions matching this goal
# enrichment.has_context        -> True if any source returned results
# enrichment.source_count       -> number of sources that contributed

# Reflect: persist what was decided and what happened
decision_id = hooks.reflect_hook(
    goal="analyze auth module", cycle=1,
    decision={"action": "scan_dependencies"},
    outcome={"success": True, "files_found": 12},
)
```

All hooks degrade gracefully. If deep_store or graph_memory is `None` or unavailable, hooks return empty/default results.

---

## Configuration

Deep memory is configured via `.spine/config.json`:

```json
{
  "deep_store": {
    "enabled": true,
    "database_url": "postgresql://spine:spine@localhost:5432/spine_memory",
    "embedding_provider": "voyage",
    "embedding_dimensions": 1024,
    "confidence_half_life_days": 90.0,
    "auto_sync_embeddings": true,
    "project_scope": "my-project"
  }
}
```

Load configuration programmatically:

```python
from spine.memory.deep_config import DeepStoreConfig
from pathlib import Path

config = DeepStoreConfig.load(Path("/path/to/project"))
```

| Setting | Default | Description |
|---------|---------|-------------|
| `enabled` | `false` | Master switch for deep memory |
| `database_url` | `postgresql://spine:spine@localhost:5432/spine_memory` | PostgreSQL connection string |
| `embedding_provider` | `local` | Reuses SPINE's embedding provider names |
| `embedding_dimensions` | `384` | Must match the selected provider |
| `confidence_half_life_days` | `90.0` | Days for confidence to decay by half |
| `auto_sync_embeddings` | `true` | Generate embeddings on store (vs batch) |
| `project_scope` | `""` | Project identifier for federation |

---

## Integration with OODA Loop

The OODA loop calls MemoryHooks at two points:

1. **Orient phase:** `orient_hook()` queries deep store and graph memory to provide additional context for decision-making. The returned `OrientEnrichment` includes:
   - Semantically similar memories from the deep store
   - Graph neighborhoods of entities matching the current goal
   - Past OODA decisions from previous cycles with similar goals

2. **Reflect phase:** `reflect_hook()` logs the decision and its outcome to the deep store's `spine_decisions` table, creating an auditable trail of agent reasoning.

3. **Episode sync:** After an episodic memory episode completes, `episode_sync_hook()` creates corresponding entities in the deep store, bridging ephemeral episode records with long-term persistent memory.

This integration means the OODA loop progressively builds a knowledge graph as it operates: each cycle potentially adds new memories, relationships, and decision records that inform future cycles.

---

## Integration with MemoryFacade

`MemoryFacade` includes deep store and graph memory as searchable sources:

```python
facade = MemoryFacade(
    kv=kv, scratchpad=pad, ephemeral=eph,
    vector=vs, episodic=episodic,
    deep_store=store,       # Tier 6
    graph_memory=graph,     # Tier 7
)

# Searches all 7 tiers, scores weighted by source type
results = facade.search("authentication", top_k=10)
```

Source weights for cross-tier ranking:

| Source | Weight | Rationale |
|--------|--------|-----------|
| vector | 1.0 | Highest-fidelity semantic match |
| episodic | 0.9 | Goal-based recall with proven outcomes |
| deep | 0.85 | Persistent semantic memory with decay |
| graph | 0.8 | Structural relationship context |
| kv | 0.8 | Exact key match |
| scratchpad | 0.7 | Working memory (temporary) |
| ephemeral | 0.6 | Session-scoped (decays fastest) |

---

## Memory Tier Summary

| Tier | Component | Backend | Scope | Added |
|------|-----------|---------|-------|-------|
| 1 | KVStore | SQLite/File | Namespace key-value | v0.3.24 |
| 2 | Scratchpad | In-memory | Short-term task notes | v0.3.24 |
| 3 | EphemeralMemory | In-memory | Session with decay | v0.3.24 |
| 4 | VectorStore | LanceDB + keyword | Semantic search | v0.3.24 |
| 5 | EpisodicMemory | SQLite + FTS5 | Goal-based episode recall | v0.3.29 |
| 6 | DeepMemoryStore | PostgreSQL + pgvector | Long-term persistent memory | v0.4.0 |
| 7 | GraphMemory | PostgreSQL + NetworkX | Graph traversal + analytics | v0.4.0 |
| -- | FederatedMemory | MCP session pool | Cross-project Minna queries | v0.4.0 |
| -- | MemoryHooks | Callback layer | OODA orient/reflect wiring | v0.4.0 |

---

[Back to Docs](README.md) | [Memory System](memory-system.md) | [Agent OS](agent-os.md)
