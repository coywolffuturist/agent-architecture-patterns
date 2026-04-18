# BS3 × mem9 Build Plan
## Brain Simulator 3 Persistent Memory via mem9 + TiDB Cloud

*Author: Jared (OS-1) | April 2026 | Build Day Target: AGI House Hackathon*

---

## The Problem (Recap)

Brain Simulator 3's Universal Knowledge Store (UKS) is an in-memory graph of Things and Relationships. It has four compounding failure modes at scale:

1. **Unbounded RAM growth** — no eviction policy; graph accumulates forever
2. **No relevance scoring** — every lookup traverses the whole graph
3. **No durability** — state lost on restart; persistence is bolted on
4. **No semantic search** — lookup is by exact name only

These are the same four problems our Mitosis agents have with markdown-based memory. Same disease, different host.

**mem9 + TiDB Cloud is the cure** — and it's already built.

---

## What We're Building

A **BS3 Memory Adapter** that:
1. Intercepts UKS writes → persists Things/Relationships to mem9 (mnemo-server → TiDB Cloud)
2. Intercepts UKS reads on miss → semantic search via mem9, lazy-loads result into working set
3. Caps working set size in RAM — evicts lowest-scored entries
4. Runs a decay engine via TiFlash to retire stale knowledge

The result: BS3's UKS becomes a **relevance-gated working set** backed by unlimited cloud-persistent memory with hybrid vector + keyword search.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    BS3 Agent Runtime                     │
│                                                         │
│   UKS Working Set (RAM, capped at N Things)             │
│        ↕  write-through / read-on-miss                  │
│   BS3MemoryAdapter  ←────── ContextManager              │
│        ↕                    (relevance scoring)         │
│   mem9 REST API                                         │
│   POST /v1alpha2/mem9s/memories    (store)              │
│   GET  /v1alpha2/mem9s/memories/search  (recall)        │
│        ↕                                                │
│   mnemo-server (Go)                                     │
│        ↕                                                │
│   TiDB Cloud Serverless                                 │
│   ├── memories table (TiKV row store)                   │
│   ├── VECTOR index on embedding (cosine search)         │
│   ├── FULLTEXT index on content (keyword search)        │
│   └── TiFlash columnar replica (decay analytics)        │
│                                                         │
│   Decay Engine (cron, TiFlash-powered)                  │
│   "Things not accessed in 30d, low confidence → archive"│
└─────────────────────────────────────────────────────────┘
```

---

## mem9 Schema Mapping

mem9's `memories` table (TiDB Cloud):

```sql
CREATE TABLE memories (
  id           VARCHAR(36) PRIMARY KEY,
  content      MEDIUMTEXT NOT NULL,           -- serialized Thing/Relationship
  source       VARCHAR(100),                  -- "bs3-uks"
  tags         JSON,                          -- ["thing", "dog"] or ["relationship"]
  metadata     JSON,                          -- { type, relationships, layer, confidence }
  embedding    VECTOR(1024) GENERATED ALWAYS AS (
                 EMBED_TEXT("tidbcloud_free/amazon/titan-embed-text-v2", content)
               ) STORED,                      -- auto-embedded by TiDB Cloud, no OpenAI needed
  memory_type  VARCHAR(20) DEFAULT 'pinned',
  agent_id     VARCHAR(100),                  -- which BS3 agent owns this
  state        VARCHAR(20) DEFAULT 'active',
  version      INT DEFAULT 1,
  created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Thing → Memory mapping

```json
{
  "content": "Fido is-a dog, has legs:4 (inherited), name:Fido",
  "source": "bs3-uks",
  "tags": ["thing", "dog", "animal"],
  "metadata": {
    "type": "thing",
    "thing_name": "Fido",
    "relationships": [
      {"rel_type": "is-a", "target": "dog", "confidence": 1.0},
      {"rel_type": "has", "attribute": "legs", "value": 4, "inherited": true}
    ],
    "access_count": 0,
    "confidence": 1.0,
    "last_accessed": "2026-04-18T22:00:00Z"
  }
}
```

### Relationship → Memory mapping

```json
{
  "content": "Fido can play fetch IF weather is sunny",
  "source": "bs3-uks",
  "tags": ["relationship", "clause", "conditional"],
  "metadata": {
    "type": "clause",
    "source_thing": "Fido",
    "relation": "can",
    "target": "play fetch",
    "condition": {"rel_type": "is", "subject": "weather", "value": "sunny"},
    "confidence": 0.9
  }
}
```

---

## BS3 Memory Adapter (Python)

```python
import requests
import json
from dataclasses import dataclass, field
from typing import Optional
import time

MEM9_API_URL = "https://api.mem9.ai"   # or self-hosted mnemo-server
MEM9_API_KEY = "your-api-key-here"     # from: curl -sX POST https://api.mem9.ai/v1alpha1/mem9s
BS3_AGENT_ID = "bs3-uks"

HEADERS = {
    "X-API-Key": MEM9_API_KEY,
    "X-Mnemo-Agent-Id": BS3_AGENT_ID,
    "Content-Type": "application/json"
}


@dataclass
class ThingRef:
    name: str
    memory_id: str
    access_count: int = 0
    confidence: float = 1.0
    last_accessed: float = field(default_factory=time.time)


class BS3MemoryAdapter:
    """
    Wraps BS3's UKS with a persistent cloud memory backend via mem9.
    
    Drop this in as a proxy layer between BS3's Thing lookup/creation
    and the in-memory UKS. The UKS becomes a relevance-gated working set.
    """

    def __init__(
        self,
        uks,                          # the live BS3 UKS instance
        max_working_set: int = 5000,  # max Things held in RAM
        relevance_threshold: float = 0.3,
        decay_days: int = 30
    ):
        self.uks = uks
        self.max_working_set = max_working_set
        self.theta = relevance_threshold
        self.decay_days = decay_days
        self._index: dict[str, ThingRef] = {}  # name → ThingRef

    # ──────────────────────────────────────────────
    # WRITE PATH: UKS mutation → persist to mem9
    # ──────────────────────────────────────────────

    def add_thing(self, thing) -> str:
        """
        Add a Thing to the UKS and persist it to mem9.
        Returns mem9 memory ID.
        """
        # 1. Add to live UKS as normal
        self.uks.add_thing(thing)

        # 2. Serialize and store in mem9
        memory_id = self._store_thing(thing)

        # 3. Track in local index
        self._index[thing.name] = ThingRef(
            name=thing.name,
            memory_id=memory_id
        )

        # 4. Evict if working set is full
        if len(self._index) > self.max_working_set:
            self._evict_lowest_scored()

        return memory_id

    def add_relationship(self, source_thing, rel_type, target_thing, **kwargs):
        """Add a Relationship and persist it."""
        self.uks.add_relationship(source_thing, rel_type, target_thing, **kwargs)
        self._store_relationship(source_thing, rel_type, target_thing, **kwargs)

    # ──────────────────────────────────────────────
    # READ PATH: miss → mem9 semantic search → lazy load
    # ──────────────────────────────────────────────

    def get_thing(self, name: str, context: str = None) -> Optional[object]:
        """
        Retrieve a Thing. If not in working set, search mem9 and lazy-load.
        context: optional natural language context for relevance scoring.
        """
        # 1. Check live UKS first
        thing = self.uks.get_thing(name)
        if thing:
            if name in self._index:
                self._index[name].access_count += 1
                self._index[name].last_accessed = time.time()
            return thing

        # 2. Miss — search mem9
        results = self._search_mem9(name, context or name, top_k=5)
        if not results:
            return None

        # 3. Find exact match or best semantic match
        best = next(
            (r for r in results if r["metadata"].get("thing_name") == name),
            results[0] if results else None
        )
        if not best:
            return None

        # 4. Deserialize and load into UKS
        thing = self._deserialize_thing(best)
        self.uks.add_thing(thing)

        self._index[name] = ThingRef(
            name=name,
            memory_id=best["id"],
            access_count=best["metadata"].get("access_count", 0),
            confidence=best["metadata"].get("confidence", 1.0)
        )

        return thing

    def semantic_search(self, query: str, top_k: int = 10) -> list:
        """
        Search mem9 for semantically related Things.
        Returns deserialized Things, sorted by relevance.
        Not limited to working set — searches all persisted knowledge.
        """
        results = self._search_mem9(query, query, top_k=top_k)
        return [self._deserialize_thing(r) for r in results if r]

    # ──────────────────────────────────────────────
    # mem9 API calls
    # ──────────────────────────────────────────────

    def _store_thing(self, thing) -> str:
        """Serialize a BS3 Thing and POST to mem9."""
        relationships = []
        for rel in getattr(thing, 'relationships', []):
            relationships.append({
                "rel_type": str(rel.rel_type),
                "target": str(rel.target),
                "confidence": getattr(rel, 'confidence', 1.0),
                "inherited": getattr(rel, 'inherited', False)
            })

        content = self._thing_to_content(thing, relationships)

        payload = {
            "content": content,
            "source": "bs3-uks",
            "tags": self._extract_tags(thing),
            "metadata": {
                "type": "thing",
                "thing_name": thing.name,
                "relationships": relationships,
                "access_count": 0,
                "confidence": getattr(thing, 'confidence', 1.0)
            }
        }

        resp = requests.post(
            f"{MEM9_API_URL}/v1alpha2/mem9s/memories",
            headers=HEADERS,
            json=payload,
            timeout=10
        )
        resp.raise_for_status()
        return resp.json()["id"]

    def _store_relationship(self, source, rel_type, target, **kwargs):
        """Store a standalone Relationship."""
        content = f"{source.name} {rel_type} {target.name}"
        if condition := kwargs.get('condition'):
            content += f" IF {condition}"

        payload = {
            "content": content,
            "source": "bs3-uks",
            "tags": ["relationship", str(rel_type)],
            "metadata": {
                "type": "relationship",
                "source_thing": source.name,
                "rel_type": str(rel_type),
                "target_thing": target.name,
                "confidence": kwargs.get('confidence', 1.0),
                **{k: v for k, v in kwargs.items() if k not in ('confidence',)}
            }
        }

        requests.post(
            f"{MEM9_API_URL}/v1alpha2/mem9s/memories",
            headers=HEADERS,
            json=payload,
            timeout=10
        )

    def _search_mem9(self, query: str, context: str, top_k: int = 10) -> list:
        """Hybrid search: vector + keyword over all persisted memories."""
        resp = requests.get(
            f"{MEM9_API_URL}/v1alpha2/mem9s/memories/search",
            headers=HEADERS,
            params={"q": query, "limit": top_k},
            timeout=10
        )
        if resp.status_code != 200:
            return []
        return resp.json().get("memories", [])

    def _update_memory(self, memory_id: str, updates: dict):
        """Update metadata on an existing memory (e.g., access_count, confidence)."""
        requests.patch(
            f"{MEM9_API_URL}/v1alpha2/mem9s/memories/{memory_id}",
            headers=HEADERS,
            json=updates,
            timeout=5
        )

    # ──────────────────────────────────────────────
    # Working set management
    # ──────────────────────────────────────────────

    def _evict_lowest_scored(self):
        """Evict the lowest-relevance Thing from the working set."""
        if not self._index:
            return

        now = time.time()
        scored = {
            name: self._relevance_score(ref, now)
            for name, ref in self._index.items()
        }
        worst_name = min(scored, key=scored.get)
        ref = self._index.pop(worst_name)

        # Flush updated access count back to mem9
        self._update_memory(ref.memory_id, {
            "metadata": {
                "access_count": ref.access_count,
                "last_accessed": ref.last_accessed
            }
        })

        # Remove from live UKS
        self.uks.remove_thing(worst_name)

    def _relevance_score(self, ref: ThingRef, now: float) -> float:
        """Score = 0.3*recency + 0.3*frequency + 0.4*confidence."""
        days_old = (now - ref.last_accessed) / 86400
        recency   = 1.0 / (1.0 + days_old)
        frequency = min(1.0, ref.access_count / 100.0)
        return 0.3 * recency + 0.3 * frequency + 0.4 * ref.confidence

    # ──────────────────────────────────────────────
    # Helpers
    # ──────────────────────────────────────────────

    def _thing_to_content(self, thing, relationships: list) -> str:
        """Render a Thing as natural language for embedding."""
        lines = [f"{thing.name}"]
        for rel in relationships[:20]:  # cap at 20 to keep content focused
            inherited = " (inherited)" if rel.get("inherited") else ""
            lines.append(f"  {rel['rel_type']} → {rel['target']}{inherited}")
        return "\n".join(lines)

    def _extract_tags(self, thing) -> list[str]:
        """Extract meaningful tags from a Thing's relationships."""
        tags = ["thing"]
        for rel in getattr(thing, 'relationships', []):
            if str(rel.rel_type) == "is-a":
                tags.append(str(rel.target))
        return tags[:10]

    def _deserialize_thing(self, memory: dict) -> object:
        """Reconstruct a BS3 Thing from a mem9 memory record."""
        meta = memory.get("metadata", {})
        # This needs to be adapted to your BS3 Thing class constructor
        # Pseudocode:
        # thing = UKSThing(name=meta["thing_name"])
        # for rel in meta.get("relationships", []):
        #     thing.add_relationship(rel["rel_type"], rel["target"], confidence=rel["confidence"])
        # return thing
        raise NotImplementedError("Implement for your BS3 Thing class")
```

---

## Decay Engine

Runs as a cron job. Uses TiFlash (auto-provisioned by TiDB Cloud) for fast analytical queries without blocking the transactional path.

```python
import requests

def run_decay_engine(api_url: str, api_key: str, decay_factor: float = 0.95, archive_threshold: float = 0.1):
    """
    Decay engine — call periodically (daily or weekly).
    Reduces confidence on stale memories and archives below threshold.
    """
    headers = {"X-API-Key": api_key, "Content-Type": "application/json"}

    # 1. Get all active memories not accessed in 30 days (via search with stale filter)
    # In production this would be a direct TiDB SQL query via mnemo-server admin API
    # or a custom endpoint. For now, approximate via search + metadata filter.

    # 2. Apply decay to stale memories
    stale_resp = requests.get(
        f"{api_url}/v1alpha2/mem9s/memories",
        headers=headers,
        params={"state": "active", "stale_days": 30, "limit": 1000}
    )

    if stale_resp.status_code == 200:
        memories = stale_resp.json().get("memories", [])
        for mem in memories:
            meta = mem.get("metadata", {})
            new_confidence = meta.get("confidence", 1.0) * decay_factor

            if new_confidence < archive_threshold:
                # Archive it
                requests.patch(
                    f"{api_url}/v1alpha2/mem9s/memories/{mem['id']}",
                    headers=headers,
                    json={"state": "archived"}
                )
            else:
                # Decay confidence
                requests.patch(
                    f"{api_url}/v1alpha2/mem9s/memories/{mem['id']}",
                    headers=headers,
                    json={"metadata": {**meta, "confidence": new_confidence}}
                )

    print(f"Decay engine run complete. Processed {len(memories)} stale memories.")
```

---

## Build Day Timeline (AGI House, 1 day)

### Hour 0–1: Infrastructure setup
- [ ] Provision TiDB Cloud Starter free tier (25GiB, 250M RUs/month)
- [ ] Deploy mnemo-server: `MNEMO_DSN="..." go run ./cmd/mnemo-server`
- [ ] Provision API key: `curl -sX POST http://localhost:8080/v1alpha1/mem9s`
- [ ] Verify: `curl http://localhost:8080/health`
- [ ] Enable auto-embedding: set `MNEMO_EMBED_AUTO_MODEL=tidbcloud_free/amazon/titan-embed-text-v2`

### Hour 1–2: BS3 adapter — core write path
- [ ] Implement `add_thing()` → `_store_thing()` → mem9 POST
- [ ] Implement `add_relationship()` → `_store_relationship()`
- [ ] Test: create 100 Things, verify they appear in mem9 dashboard

### Hour 2–3: BS3 adapter — read path + working set
- [ ] Implement `get_thing()` with UKS miss → mem9 search → lazy load
- [ ] Implement `_evict_lowest_scored()` with write-back
- [ ] Test: fresh UKS with empty working set, query Things that are only in mem9

### Hour 3–4: Semantic search integration
- [ ] Implement `semantic_search()` — expose to BS3 modules
- [ ] Wire into BS3's natural language query path (if applicable)
- [ ] Test: query "what do dogs do?" → returns Fido, Rex, Lassie etc.

### Hour 4–5: Decay engine + cross-agent memory
- [ ] Implement and schedule `run_decay_engine()`
- [ ] Test with artificial stale data
- [ ] Demo: two BS3 agents sharing the same mem9 tenant — agent 2 learns what agent 1 knows

### Hour 5–6: Polish + demo
- [ ] mem9 dashboard showing BS3 knowledge graph
- [ ] Side-by-side: BS3 before (RAM-only, lost on restart) vs after (persistent, semantic)
- [ ] Ship announcement

---

## Self-Hosted vs TiDB Cloud Starter

| Option | Setup time | Cost | Semantic search | Recommended for |
|--------|-----------|------|-----------------|-----------------|
| TiDB Cloud Starter (free) | 5 minutes | Free (25GiB) | Auto-embedding (no OpenAI) | Build day, demos |
| Self-hosted TiDB | 30–60 min | Infra only | Manual embedding pipeline | Production |
| TiDB Cloud Dedicated | 10 min | $~200+/mo | Auto-embedding | Production at scale |

**Recommendation: TiDB Cloud Starter for the build day.** Free tier is generous enough for a demo. If you need self-hosted for data residency, the same schema.sql works on self-hosted TiDB — just swap the DSN.

---

## Connection to Broader Research

This adapter is the concrete implementation of:
- **AGP's RSPL layer** — Things/Relationships as versioned, lifecycle-managed resources
- **MemGPT's virtual memory model** — working set as RAM, mem9 as disk, agent controls what's in scope
- **LARQL's longer horizon** — if model weights become queryable (LARQL), Things in mem9 could eventually map to weight-space features, not just text records

**For Mitosis agents:** replace "BS3 UKS" with "agent context/memory" and this is the same architecture. mem9 is the shared memory substrate; every Mitosis agent gets a `memory_store` / `memory_search` / `memory_get` tool pointing at the same mnemo-server. Cross-agent knowledge sharing is a config flag (`X-Mnemo-Agent-Id`), not a protocol.

---

## Quick Start Commands

```bash
# 1. Clone mem9
git clone https://github.com/mem9-ai/mem9.git
cd mem9/server

# 2. Set up TiDB Cloud (get DSN from tidbcloud.com → free tier)
export MNEMO_DSN="user:pass@tcp(gateway01.us-west-2.prod.aws.tidbcloud.com:4000)/mnemos?tls=true&parseTime=true"
export MNEMO_EMBED_AUTO_MODEL="tidbcloud_free/amazon/titan-embed-text-v2"

# 3. Run mnemo-server
go run ./cmd/mnemo-server

# 4. Provision your API key
export MEM9_API_KEY=$(curl -s -X POST http://localhost:8080/v1alpha1/mem9s | jq -r '.id')
echo "API Key: $MEM9_API_KEY"

# 5. Test it
curl -s -H "X-API-Key: $MEM9_API_KEY" \
  -X POST http://localhost:8080/v1alpha2/mem9s/memories \
  -H "Content-Type: application/json" \
  -d '{"content": "Fido is-a dog, has legs 4, likes fetch", "source": "bs3-uks", "tags": ["thing","dog"]}' | jq .

# 6. Search it
curl -s -H "X-API-Key: $MEM9_API_KEY" \
  "http://localhost:8080/v1alpha2/mem9s/memories/search?q=dog+fetch" | jq '.memories[].content'
```

---

*For questions: ping Jared or drop in the mem9 GitHub.*
