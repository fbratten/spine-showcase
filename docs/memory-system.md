# Memory System (v0.3.29)

SPINE provides a 5-tier memory architecture unified by `MemoryFacade`. Each tier serves a different temporal scope and access pattern, from fast key-value lookups to goal-based episodic recall.

---

## Architecture

```
+-------------------------------------------------------------------+
|                       MemoryFacade                                 |
|            Unified search across all tiers                         |
+----------+----------+----------+----------+-----------------------+
|  Tier 1  |  Tier 2  |  Tier 3  |  Tier 4  |       Tier 5         |
| KVStore  |Scratchpad|Ephemeral |VectorStor|   EpisodicMemory     |
|          |          | Memory   |    e     |                       |
| key=val  |task notes| session  | semantic |    goal-based         |
| by ns    |short-term| w/ decay | + keyword|    episode recall     |
|          |          |          |          |                       |
| SQLite/  |in-memory |in-memory | LanceDB +|   SQLite + FTS5      |
| File     |          |          | keyword  |                       |
+----------+----------+----------+----------+-----------------------+
                          |
              +-----------+-----------+
              |     VerdictRouter     |
              | Routes accept/reject/ |
              | revise to correct tier|
              +-----------------------+
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

## MemoryFacade

Unified interface that searches across all 5 tiers and merges results:

```python
from spine.memory.facade import MemoryFacade

facade = MemoryFacade(
    kv=kv_store,
    scratchpad=scratchpad,
    ephemeral=ephemeral_memory,
    vector=vector_store,
    episodic=episodic_memory,
)

# Search all tiers at once
results = facade.search("authentication", top_k=10)
# Returns ranked results from whichever tiers have relevant data

# Tier-specific access still available
facade.kv.get("config", "timeout_ms")
facade.episodic.recall(goal="Fix auth bug")
```

The facade handles score normalization across tiers so that results from different backends are comparable.

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
| Search everything at once | MemoryFacade | Cross-tier unified search |

---

[Back to Docs](README.md) | [Agent OS](agent-os.md)
