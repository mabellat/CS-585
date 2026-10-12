# FoundationDB — Study Guide

**Paper:** Zhou et al., SIGMOD Record 2022 (SIGMOD ’21). **[P]**  
**Concurrency:** [`DiscussionXactCC-1.pdf`](DiscussionXactCC-1.pdf). **[CC]**  
**Graph lecture:** eBay / Hieu Nguyen deck. **[G]**

Narrative: **[FoundationDB_Full_Primer.md](FoundationDB_Full_Primer.md)**.

---

## Table of contents

1. [Concurrency control glossary](#1-concurrency-control-glossary)
2. [What FoundationDB is](#2-what-foundationdb-is)
3. [Architecture](#3-architecture)
4. [Transaction lifecycle](#4-transaction-lifecycle)
5. [Storage, logging, recovery](#5-storage-logging-recovery)
6. [Replication and fault tolerance](#6-replication-and-fault-tolerance)
7. [Simulation testing](#7-simulation-testing)
8. [JanusGraph, keys, indices](#8-janusgraph-keys-indices)
9. [Graph query caching (eBay)](#9-graph-query-caching-ebay)
10. [Exam cheat sheet](#10-exam-cheat-sheet)

---

## 1. Concurrency control glossary

| Term | Meaning |
|------|---------|
| **Schedule** | Order in which interleaved transaction operations run **[CC]** |
| **Serializable** | Same final state as some serial execution **[CC]** |
| **Strict serializable** | Serializable + real-time commit order respected **[CC][P]** |
| **Two-phase locking** | Growing locks, then shrinking; serializable if locks held to end **[CC]** |
| **Shared / exclusive lock** | Read vs write lock; compatibility matrix **[CC]** |
| **Optimistic concurrency control** | Read → validate → write; abort on conflict **[CC][P]** |
| **Multi-version concurrency control** | Multiple versions; read at timestamp **[CC][P]** |
| **Read-your-writes** | Transaction sees its own uncommitted writes **[P]** |

---

## 2. What FoundationDB is

| Property | Detail |
|----------|--------|
| Model | **Ordered** transactional **key-value** store **[P][G]** |
| Transactions | Multi-key, **strict serializability** default **[P]** |
| Philosophy | **Layers** for relational / document / graph **[P]** |
| Workload target | **Online transaction processing**: read-mostly, small key sets, low contention **[P]** |
| Not | Full SQL engine in the core product **[P]** |

---

## 3. Architecture

### Control plane **[P][G]**

Coordinators (Paxos) → ClusterController → recruits Sequencer, DataDistributor, RateKeeper.

### Data plane **[P][G]**

| Role | Function |
|------|----------|
| Sequencer | Read/commit **versions**; epoch boundary on failure |
| Proxy | Read version + **commit orchestration** |
| Resolver | **Conflict detection** per key-range shard |
| Log server | Durable **write-ahead log** |
| Storage server | **Shard** data; **versioned reads** |

**Scaling:** reads ∝ storage servers; writes ∝ proxies + resolvers + log servers.

---

## 4. Transaction lifecycle

| Step | Action |
|------|--------|
| 1 | Client → proxy: **get read version** |
| 2 | Client → storage servers: **reads at version** |
| 3 | Client buffers **writes** locally |
| 4 | Client → proxy: commit with **read/write sets** |
| 5 | Proxy → sequencer: **commit version** |
| 6 | Proxy → resolvers: **conflict check** (all must pass) |
| 7 | Proxy → log servers: **durable append** |
| 8 | Storage servers **pull** and apply log |

**Read-only, no writes:** client can skip full commit protocol. **[P][G]**

**Abort:** client retries; optimistic design accepts wasted work on conflict.

---

## 5. Storage, logging, recovery

| Concept | Meaning |
|---------|---------|
| **MVCC window** | Recent versions in memory (~5 s lecture) **[G]** |
| **Future version** | Read version not applied yet on replica **[G]** |
| **Version too old** | Outside retained versions **[G]** |
| **KCV** | Known committed version (per proxy) **[G][P]** |
| **DV** | Durable version on log server **[G][P]** |
| **PEV / RV** | Previous epoch end / recovery version for redo trim **[G][P]** |

Recovery: new transaction system epoch; **no undo replay on pages** like ARIES; discard versions **after RV**.

---

## 6. Replication and fault tolerance

| Data | Strategy |
|------|----------|
| Coordinator metadata | Paxos **majority** **[P]** |
| Log records | **f+1** synchronous replicas on commit **[P]** |
| Storage shards | **Teams** of f+1 storage servers; async copy; DataDistributor rebalances **[P]** |

**f failures, f+1 replicas** (reconfigure rather than only quorum read path). **[P]**

---

## 7. Simulation testing

- Deterministic discrete-event **simulator**; real FoundationDB code. **[P]**
- **Flow** actors; one process per core in production. **[P]**
- Fault injection + **swarm** random configs. **[P]**
- Bugs **reproducible**; performance not fully validated in sim. **[P]**

---

## 8. JanusGraph, keys, indices

| Operation | FoundationDB pattern **[G]** |
|-----------|-------------------------------|
| Add vertex / edge | **set** keys |
| Scan neighbors | **getRange** on prefix |
| Delete vertex | **getRange** edges + **clear** / **clearRange** |
| Lookup by code | **Composite index** → **get** |
| Filter edges on supernode | **Vertex-centric index** → narrow **getRange** |
| Text search | **Mixed index** (external) |

---

## 9. Graph query caching (eBay)

| Technique | When |
|-----------|------|
| **Per-hop template cache** | Dominant hop (e.g. LAX out-edges); key = template + params |
| **Full-query cache** | Batch load, then read-only |
| **Consistency** | Graph update + **cache invalidation** same transaction |
| **Write-around** | Update graph; invalidate cache keys (not write-through on every read) |

**Waves:** Gremlin latency ≈ sum of **request waves**; caching cuts **getRange** count.

---

## 10. Exam cheat sheet

1. **Unbundled:** transaction system / log / storage scale separately. **[P]**
2. **Strict serializability:** sequencer versions + resolver admit + durable log. **[P]**
3. **Optimistic + multi-version:** reads at version; validate at commit. **[P][CC]**
4. **Client reads storage directly**; writes through proxy. **[P]**
5. **Failure → sequencer dies → recovery epoch** (seconds). **[P]**
6. **Simulation** before trust in production. **[P]**
7. **JanusGraph** = layer; graph → keys/ranges. **[G]**
8. **Per-hop cache** + transactional invalidation for eBay latency. **[G]**
