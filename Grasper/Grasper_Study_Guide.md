# Grasper: A Distributed System for Property Graphs

**Citation:** Hongzhi Chen, Changji Li, Juncheng Fang, Chenghuan Huang, James Cheng, Jian Zhang, Yifan Hou, Xiao Yan. *Grasper: A High Performance Distributed System for OLAP on Property Graphs.* SoCC 2019, CUHK.  
**Lecture:** `GrasperLecture.pdf` (CSCI 585)  
**Also:** [`Grasper_Full_Primer.md`](Grasper_Full_Primer.md) · [`Grasper_Exam_QA.md`](Grasper_Exam_QA.md) · [`Grasper_Lecture_Slides.md`](Grasper_Lecture_Slides.md)

---

## Table of contents

1. [Story in one page](#1-story-in-one-page)
2. [Glossary](#2-glossary)
3. [Property graphs & query languages](#3-property-graphs--query-languages)
4. [Gremlin essentials](#4-gremlin-essentials)
5. [RDMA background](#5-rdma-background)
6. [Architecture overview](#6-architecture-overview)
7. [Storage: normal vs. RDMA memory](#7-storage-normal-vs-rdma-memory)
8. [RDMA messaging](#8-rdma-messaging)
9. [Query engine: flow types & DAG](#9-query-engine-flow-types--dag)
10. [Expert Model](#10-expert-model)
11. [Optimizations & work stealing](#11-optimizations--work-stealing)
12. [Evaluation](#12-evaluation)
13. [Limitations & exam cheat sheet](#13-limitations--exam-cheat-sheet)

---

## 1. Story in one page

**Problem:** Industry wants **online graph analytics** on **property graphs** (billions of entities) with **low latency** (e.g., p95 &lt; 50 ms on fraud/recommendation-style queries). Systems like **Titan** and **JanusGraph** use **external storage** (e.g., HBase/Cassandra), **non-native** graph layouts, and often **one thread per query**, yielding **low CPU and network utilization** under Gremlin workloads.

**Grasper’s answer:** A **distributed, in-memory OLAP** system that:

- Stores PG data **natively** on each **query engine** (not a separate transactional DB during execution).
- Splits **topology** (local) from **labels/properties** (RDMA KV) for **load balance**.
- Uses **one-sided RDMA** for cheap remote reads and **thread-level mailboxes**.
- Executes Gremlin as a **data-flow** of **messages** over a query **DAG**.
- Uses **Experts** (per operator family) with **adaptive parallelism**, **specialized caches**, **core binding**, and **work stealing**.

**Contrast from lecture:** **JanusGraph + FoundationDB** = stateless graph layer over transactional KV (scale tiers independently). **Grasper** = **stateful** engines with partitioned graph + RDMA mesh.

---

## 2. Glossary

| Term | Meaning |
|------|---------|
| **PG (property graph)** | Directed graph; vertices/edges have labels + key–value properties; schema-flexible |
| **OLAP (here)** | Interactive **analytical** Gremlin queries, not bulk offline PageRank-only systems |
| **Gremlin** | Procedural graph traversal language (TinkerPop **steps**) |
| **Engine / query server** | Machine that holds a graph partition + runs query threads |
| **Normal memory** | Local RAM: topology, indexes, intermediate results |
| **RDMA memory** | Pre-registered RAM exposed as **distributed KV** for remote one-sided access |
| **Message** | Intermediate/final result of one query step + metadata; may be split if too large |
| **Expert** | Executor for one **category** of steps (Filter, Traversal, Group, …) |
| **Flow type** | Sequential, Barrier, or Branch — defines synchronization pattern |
| **Straggler** | Slow task/thread that delays entire query |
| **HDFS** | Load-time source for graph files; not the hot path for every read during query |

---

## 3. Property graphs & query languages

### 3.1 Data model

- **Vertices** = entities; **edges** = directed relationships.
- **Label** = type/role (`person`, `knows`, `airport`, `route`).
- **Properties** = arbitrary KV pairs; different nodes may have different keys.
- **RDF** = triple store + **SPARQL**; logically related to PG but different tooling.

### 3.2 Systems map (lecture)

| Stack | Query language |
|-------|----------------|
| Neo4j | Cypher |
| Grasper, JanusGraph, TinkerPop | Gremlin |
| Jena, Virtuoso, Wukong (RDMA) | SPARQL |

### 3.3 Workload examples (lecture)

| Domain | Idea | Latency target (slides) |
|--------|------|-------------------------|
| eBay fraud | Expand from user over shared IP, email, device, … | p95 **&lt; 50 ms** |
| Recommendations | Middle vertex (item); neighbors via edges with `probability` | p99 **&lt; 50 ms**, **100M+** vertices |

---

## 4. Gremlin essentials

**Nature:** **Procedural** — you specify the **path** through the graph (contrast SQL **declarative**).

**Step families:**

1. **Map** — transform (`valueMap`, `count`, `project`, …)
2. **flatMap** — expand (`out`, `in`, `both`, `unfold`, …)
3. **Filter** — `has`, `hasLabel`, `or`, …
4. **sideEffect** — `store`, `aggregate`
5. **Branch** — `union`, `choose`, `coalesce`, …

**Canonical examples (Modern Graph):**

```text
g.V().has('name','marko').out('knows').valueMap()
g.V().has('name','marko').in('knows').valueMap()
g.V().has('lang','java').in('created').valueMap().dedup()
```

**Air routes (top 3 outgoing flights):**

```text
g.V().hasLabel('airport')
  .order().by(__.out('route').count(), desc)
  .project('airportName','outgoingFlights').by('code').by(__.out('route').count())
  .limit(3)
```

**Simple vs. complex:** few indexed lookups vs. large graph touches — drives need for **parallelism** and **load balancing**.

---

## 5. RDMA background

| Path | Copies (lecture) | Notes |
|------|------------------|-------|
| Ethernet sockets | **4×** (2 sender + 2 receiver) | User ↔ kernel ↔ NIC |
| Native RDMA | **Zero-copy** (registered user memory) | Bypasses OS for data path |
| IPoIB | Back toward socket/IP behavior | Loses much RDMA benefit |

**Verb types:** **two-sided** send/recv vs. **one-sided** read/write to remote address. Grasper uses **one-sided** for performance.

**Why it matters for Grasper:** Many **small messages** and **remote property reads**; paper: RDMA RTT **order of magnitude** lower than Ethernet for small payloads (&lt; 4 KB).

**Cost note (lecture, Oct 2025 pricing slide):** RDMA NICs comparable to 10 GbE at similar speeds; high bandwidth tiers still expensive.

---

## 6. Architecture overview

```
Clients  --TCP/IP-->  Load balancer / Master  --assigns queries-->
     multiple App Services / Client libs
              |
    +---------+---------+
    |  Engine 0   Engine 1 ... Engine N-1   (RDMA mesh)
    |  [Data store + Query engine per node]
    +-------------------+
              ^
         Load from HDFS at ingest
```

**Master:** heartbeats, **load-aware** query assignment.  
**Clients:** lightweight; submit Gremlin strings, receive results.  
**Engines:** **data store** + **query engine**; connected by **RDMA**; clients use **TCP/IP**.

**Design goals (paper):**

1. High **CPU + network** utilization for OLAP Gremlin.
2. **Adaptive parallelism** within heavy queries + **concurrency** across queries.
3. **Native** graph store co-located with engine — no external DB round-trips.
4. **RDMA-aware** layout and messaging.

---

## 7. Storage: normal vs. RDMA memory

### 7.1 Partitioning philosophy

- **Hash-partition** vertices/edges into **normal memory** (topology).
- **Hash-partition** labels/properties into **RDMA KV** separately.
- **Reason:** co-locating everything on one node → **skew**; separate partitions → **balanced CPU**; RDMA makes extra remote access cheap.

### 7.2 Per-vertex layout (conceptual)

- **`vid`**, **in-adjacency**, **out-adjacency**, property-key IDs, label id.
- **String → integer ID** maps for labels and property names (compact on wire).

### 7.3 Edge region

- Similar split: **local edge topology** in normal memory; **edge label/properties** in RDMA KV.

### 7.4 Edge lookup between two nodes

- Local adjacency identifies connectivity; **property values** may require **RDMA read** to owner engine.

### 7.5 Other normal-memory zones

- **Index buff** — property indexes.
- **Meta heap** — query metadata, temporaries.

### 7.6 RDMA zones

- **V-KVS**, **E-KVS**, **SendBufs**, **RecvBufs** (per-thread receive rings).

**Remote topology:** Engines do **not** randomly read remote adjacency; non-local expansion is expressed via **messages** + RDMA writes (paper §4.1).

---

## 8. RDMA messaging

- **Mailbox:** `w` SendBufs, `w × n` RecvBufs (threads × engines).
- **RecvBuf[i][j]:** messages from engine `j` for thread `i` — **ring buffer** (FaRM-style).
- **Patterns:** **split** (oversized result), **transfer**, **merge**, **spawn** (branch parallelism).

---

## 9. Query engine: flow types & DAG

| Flow type | Behavior | Example steps |
|-----------|----------|----------------|
| **Sequential** | 1 message in → message(s) out; parallel across messages | `out`, `in`, `has` |
| **Barrier** | Collect all messages → **aggregate** → continue | `count`, `max`, `groupCount` |
| **Branch** | Fork sub-queries → **merge** | `and`, `or`, `union` |

**Plan optimizations:**

- **Decomposition** — branch → parallel subQs.
- **Combination** — merge consecutive filters.
- **Reordering** — selective filters first.

**Artifact:** Gremlin → **step_obj** list → **DAG** of parallelism (lecture + paper Figure 8–9).

---

## 10. Expert Model

**Three features:**

1. **Adaptive parallelism** — threads ≈ #messages at step.
2. **Tailored optimizations** per step **category** (cache size, LRU vs FIFO, routing).
3. **Locality-aware binding** — thread ↔ core; experts in **regions**; load-aware routing.

**Expert pool examples:** Init, Traversal, Filter, Range, Order, Group, Math, BranchFilter, End.

**Assumption (lecture Observation 1):** Steps in same group share **access patterns** → one optimization policy per expert.

---

## 11. Optimizations & work stealing

**Stragglers:** one slow operator dominates query time → bad for tail latency.

**Work stealing:** idle thread pulls work from other **RecvBufs** in order:

1. Same **region** (e.g., Traversal) on same node  
2. Other regions (e.g., Filter) on same node  
3. Other **engines**

**Thread model:** expert pool initialized at engine start; **one thread per core**; each thread has own RDMA recv buffers; experts **share** caches/indexes among their threads.

**Without adaptive parallelism (w/o APC):** one thread per message — underutilizes cores on hot steps.

---

## 12. Evaluation

**Setup:** 10 nodes, 40 Gbps RDMA, 24 threads/node, LDBC + AMiner + Twitter.

**Findings (qualitative):**

- Beats Titan/JanusGraph/others on many latency/throughput metrics (paper).
- **Ablation:** RDMA, query optimization, work stealing each matter.
- **Scale-out:** **sublinear** speedup (communication, barriers, skew).
- **Memory:** Twitter OOM on **2 nodes** — needs cluster RAM.

**Motivation figure (paper):** LDBC **IC4** — `hasLabel` and `in` dominate time → need **step-level** parallelism.

---

## 13. Limitations & exam cheat sheet

**Lecture open issues:**

- Failure recovery?
- **Writes** / **ACID**?
- Durability model with **HDFS** ingest only?
- Not general **stream processing** (Kafka/Flink sense)

**Cheat sheet bullets:**

- **OLAP PG** + **Gremlin** + **in-memory** + **RDMA** + **Expert Model**
- **Split topology / properties** → balance load, pay RDMA reads
- **Messages + DAG + flow types**
- **vs JanusGraph:** integrated engine vs external KV
- **vs Pregel:** online Gremlin steps vs bulk synchronous graph programs

**Practice:** [`Grasper_Exam_QA.md`](Grasper_Exam_QA.md) (35+ Qs).
