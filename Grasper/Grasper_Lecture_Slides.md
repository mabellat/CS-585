# Grasper — Lecture-style slides (maps to GrasperLecture.pdf)

**Chen et al. · SoCC 2019 · CSCI 585 lecture**  
One “slide” per section — pair with [`Grasper_Study_Guide.md`](Grasper_Study_Guide.md).

---

## Slide 1 — Exam context

- Exam 1: **six papers** including **Grasper**
- Closed book · Oct 12 schedule on slide 1

---

## Slide 2 — Title

- **Grasper: A Distributed System for Property Graphs**
- Chen et al. · Shahram Ghandeharizadeh · CSCI 585

---

## Slide 3 — Outline

- Background · PG model · Gremlin · Grasper query processor · Conclusions

---

## Slides 4–9 — Networking motivation

- Classic stack: app → socket → kernel → NIC
- **Ethernet: 4 copies** (2× sender, 2× receiver)
- **RDMA: zero-copy** — verbs, registered memory
- **IPoIB** still pulls in socket/IP → copies return
- RDMA: one-sided vs two-sided; **order of magnitude** latency win (small payloads)
- NIC price table: RDMA ≈ 10 GbE at low tiers

---

## Slides 12–15 — JanusGraph vs Grasper

- **JanusGraph + FDB:** stateless graph model over **transactional KV**
- **Grasper:** **stateful** engines, graph partition with compute
- **Why separate storage?** Scale storage vs compute **independently** (JanusGraph story)
- Grasper: RDMA + TCP/IP on one network?

---

## Slides 17–20 — Property graph & ecosystems

- V, E, labels, properties, **no fixed schema**, **directed** edges
- Neo4j/Cypher · Gremlin/TinkerPop · SPARQL/RDF
- Gremlin = **procedural** steps; simple (indexed) vs complex (whole graph)

---

## Slides 21–26 — Gremlin FUN

- Air-routes GraphML + Modern Graph setup
- `marko` / `knows` / `created` / `java` examples
- Top airports: `order().by(out('route').count(), desc).limit(3)`

---

## Slides 27–31 — eBay-scale queries

- **Fraud:** user → linked IPs/devices → p95 **&lt; 50 ms**
- **Recommendations:** item vertex + `probability` edges → p99 **&lt; 50 ms**, **100M+** V
- Gremlin step taxonomy: Map, flatMap, Filter, sideEffect, Branch

---

## Slides 32–34 — Plans & SQL contrast

- Gremlin → **DAG**
- Procedural/streaming vs SQL declarative/set-oriented
- Joins vs explicit `out`/`in`

---

## Slides 35–44 — Physical data design

- Server: **normal** + **RDMA** memory
- Vertices by **vid**, ordered edges, **in/out adjacency**
- Compact **label/property IDs**
- Slide Q: **edge between two nodes?** → topology local, properties remote

---

## Slide 47 — Fast RDMA is a MUST

- Split storage design assumes cheap remote access

---

## Slides 48–53 — Architecture & HDFS

- Master, engines, clients
- **Normal:** topology + intermediates
- **RDMA mem:** distributed KV for labels/properties
- Load from **HDFS**; hash partition for **balance**
- HDFS NameNode / DataNode / blocks

---

## Slides 54–57 — Query engine

- **Messages** (max size → split)
- Flow types: **Sequential, Barrier, Branch**
- Gremlin DAG
- **Experts** per step category; hash-aware fetch; per-expert cache (LRU/FIFO)

---

## Slides 58–62 — Parallelism & stealing

- **Adaptive parallelism** = #messages
- Thread ↔ **core binding**; shared expert caches
- **Stragglers** → **work stealing** priority order
- Expert model three bullets (parallelism, optimizations, locality/LB)

---

## Slides 63–71 — Experiments

- 10 × (16 cores, 128 GB, 40 Gb RDMA, 24 threads)
- LDBC, AMiner, Twitter
- Ablation: RDMA, optimization, stealing
- Decompose / combine / reorder
- **Sublinear** scale-up; Twitter **OOM** on 2 nodes
- w/o APC, TS&LB comparisons
- vs other systems — fair comparison?

---

## Slide 72 — Summary

- In-memory PG OLAP · Gremlin · data-flow · Experts · RDMA · stealing · optimization
- Limits: faults, writes, ACID, durability, “streaming?”

---

## 45-minute presentation outline

| Min | Content |
|-----|---------|
| 0–5 | PG + why OLAP is hard; Titan/JanusGraph pain |
| 5–10 | RDMA 4-copy vs zero-copy (one diagram) |
| 10–15 | Architecture: stateful engines vs JanusGraph+FDB |
| 15–22 | Storage split + edge lookup tradeoff |
| 22–30 | Messages, flow types, DAG, three plan rules |
| 30–38 | Expert Model + work stealing + stragglers |
| 38–43 | Eval highlights + sublinear scale |
| 43–45 | Limitations + one Gremlin example |
