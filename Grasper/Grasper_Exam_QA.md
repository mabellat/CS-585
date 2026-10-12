# Grasper — Example questions and answers

Use for **closed-book practice** and **oral explanations**. Answers are concise; expand with examples from [`Grasper_Study_Guide.md`](Grasper_Study_Guide.md) when presenting.

---

## Section A — Motivation & context

**Q1. What problem does Grasper solve?**  
**A:** Distributed **OLAP** (online analytical processing) on **property graphs** with strict **latency and throughput** requirements. Existing systems (e.g., Titan, JanusGraph) pay for **non-native storage**, **external DBs**, **low CPU/network utilization**, and **fixed parallelism** per query step. Grasper co-locates native graph storage with a **data-flow query engine**, uses **RDMA**, and applies the **Expert Model** for adaptive parallelism and operator-specific optimizations. **[R1][P]**

**Q2. How is Grasper different from JanusGraph on FoundationDB (as shown in lecture)?**  
**A:** JanusGraph **separates** the graph data model from a **transactional key-value store** (FDB); graph logic talks to remote storage. Grasper is **stateful**: each app service / engine holds **partitioned in-memory graph data** and processes Gremlin locally, using **fast RDMA** so extra remote fetches are acceptable. Scale storage vs. compute **together** on graph engines rather than independently scaling a separate KV tier. **[L][R1]**

**Q3. Why does the lecture say “separate storage from processing” is sometimes desirable, and does Grasper do that?**  
**A:** Separating lets you **scale storage and compute independently** (JanusGraph + FDB). Grasper **does not** use an external OLTP store for execution; it **integrates** store + engine on each node for OLAP, but still **partitions** topology vs. properties across memory types to balance load. **[L][R1]**

**Q4. Is Grasper OLTP or OLAP?**  
**A:** **OLAP** — complex analytical Gremlin queries (traversals, filters, aggregations) over large graphs, often with **millisecond** SLA on interactive workloads (LDBC, fraud/recommendation-style queries on slides). Not focused on short transactional updates with full **ACID** semantics. **[R1][L]**

---

## Section B — Property graphs & Gremlin

**Q5. Define a property graph.**  
**A:** Vertices **V** and edges **E**. Each vertex/edge has a **label** (role/type) and **properties** (key–value). **No fixed schema** across all entities. Edges are **directed**. Vertices often have descriptive properties (name, age); edges often have **quantitative** properties (weight, probability). **[R1][L]**

**Q6. Gremlin vs. SQL — three contrasts from lecture.**  
**A:** (1) **Procedural vs. declarative:** Gremlin specifies **how** to walk the graph (step chain); SQL states **what** set you want. (2) **Streaming vs. set-oriented:** Gremlin processes traversals step-by-step; SQL returns relations. (3) **Explicit graph relationships** via `out`/`in` vs. **joins** between tables. **[L]**

**Q7. What are the five Gremlin step categories (lecture)?**  
**A:** **Map** (`values`, `count`, `project`, …), **flatMap** (`out`, `in`, `both`, `unfold`, …), **Filter** (`has`, `hasLabel`, `or`, …), **sideEffect** (`store`, `aggregate`), **Branch** (`choose`, `union`, `coalesce`, …). Map and flatMap are sometimes grouped → **four** categories. **[L]**

**Q8. Write Gremlin: “Who does Marko know?” and “Who knows Marko?”**  
**A:**  
- Outgoing knows: `g.V().has('name','marko').out('knows').valueMap()`  
- Incoming knows: `g.V().has('name','marko').in('knows').valueMap()`  
**[L]**

**Q9. Simple vs. complex Gremlin query (lecture distinction).**  
**A:** **Simple:** touches few vertices/edges, often via **indexes**. **Complex:** may touch **large portions** of the graph (e.g., eBay fraud via shared IP/device/email, recommendations over hundreds of millions of vertices). **[L]**

**Q10. RDF/SPARQL vs. property graph (one sentence each).**  
**A:** RDF uses **(Subject, Predicate, Object)** triples; SPARQL queries them. PG uses labeled vertices/edges with properties. They are **logically convertible** but different ecosystems (Jena, Virtuoso vs. Neo4j, TinkerPop). **[L]**

---

## Section C — RDMA & networking

**Q11. Why does Ethernet “copy data 4×” while RDMA is “zero copy”?**  
**A:** Classic sockets: user buffer → kernel → NIC (sender) and NIC → kernel → user (receiver) = **two copies per side**. RDMA **verbs** let the NIC access **registered user memory** directly, **bypassing** much of the OS stack for data movement. **[L][R1]**

**Q12. Two-sided vs. one-sided RDMA verbs?**  
**A:** **Two-sided:** send/recv pairs (both sides participate). **One-sided:** sender **read/write** directly to a **remote memory address** without remote CPU involvement. Grasper prefers **one-sided** read/write for bandwidth/latency. **[R1][L]**

**Q13. What is IPoIB and why mention it?**  
**A:** **IP over InfiniBand** — still goes through socket/IP stack, so you lose much of RDMA’s zero-copy benefit; latency behaves more like traditional networking. Contrasts with **native RDMA**. **[L][R1]**

**Q14. Why is a fast RDMA network “a MUST” for Grasper’s design?**  
**A:** Topology and properties are **hash-partitioned separately**; queries **pull remote label/property data** via RDMA and exchange **many small messages**. Without low-latency RDMA, that traffic would dominate. **[L][R1]**

---

## Section D — Storage & physical design

**Q15. Normal Memory vs. RDMA Memory on each node?**  
**A:** **Normal Memory:** local **graph topology** (adjacency), **intermediate results**, indexes/meta. **RDMA Memory:** **distributed KV** for vertex/edge **labels and property values** remotely readable via one-sided RDMA. **[R1][L]**

**Q16. Why partition vertices/edges separately from their properties?**  
**A:** Co-locating topology + all properties on the same node causes **skew** (hot vertices, uneven property sizes) → **unbalanced load and CPU**. Separating enables **balanced hash partitioning**; extra network is OK with RDMA. **Consequence:** finding an edge or property may require **remote RDMA read** to the engine that owns that KV shard. **[L][R1]**

**Q17. How are vertices ordered on disk/in memory (lecture)?**  
**A:** By **vertex id (`vid`)**; edges ordered similarly; **in/out adjacency lists** for traversal. Labels/properties use **compact integer IDs** (string → id maps) to save memory and bandwidth. **[L][R1]**

**Q18. How do you find an edge between two nodes in Grasper’s layout?**  
**A:** Use local topology (adjacency / edge records keyed by endpoints) to identify **edge id** and neighbors; **property/label values** may require **RDMA fetch** from the engine holding that edge’s KV partition. (Lecture emphasizes this as a design tradeoff of split storage.) **[L]**

**Q19. Where does durable graph data come from at load time?**  
**A:** **HDFS** — graph loaded into cluster; vertices/edges/labels hash-partitioned across nodes’ normal and RDMA memory. **[L][R1]**

**Q20. HDFS roles (NameNode vs. DataNode) — one line each.**  
**A:** **NameNode (master):** metadata, namespace, block locations. **DataNode (slave):** stores **blocks** (default 128–256 MB), replicates per master instructions. Optimized for **sequential throughput**. **[L]**

---

## Section E — Query engine

**Q21. What is a “message” in Grasper’s data-flow engine?**  
**A:** A unit of **intermediate or final result** from one **query step**, plus **metadata** (e.g., step type). **Max size** enforced; oversized results **split** into multiple messages. **[R1][L]**

**Q22. Three flow types and their meaning?**  
**A:** **Sequential:** step consumes a message, emits message(s) to next step (`in`, `out`, `has`). **Barrier:** **collect + aggregate** before continuing (`count`, `max`). **Branch:** **duplicate** input into **parallel sub-queries** (`and`, `or`, `union`), then merge. **[R1][L]**

**Q23. What is an Expert?**  
**A:** Physical operator for **one category** of Gremlin steps with similar function/access pattern (e.g., Filter, Traversal). One expert per category per engine; **shared caches/indexes** across threads; **hash-aware** remote fetches; **tailored** cache policy (e.g., LRU for Filter, FIFO for Traversal). **[R1][L]**

**Q24. Adaptive parallelism control — how is workload measured?**  
**A:** By **number of messages** a step receives at an engine (message size bounded). Expert sets **thread count ≈ message count** for that step. Heavy steps (e.g., `in`, `hasLabel` in LDBC IC4) get more parallelism. **[R1][L]**

**Q25. Three query plan optimizations (lecture/paper)?**  
**A:** **Decomposition** — parallel sub-queries for branch steps. **Combination** — merge adjacent filters into one scan. **Reordering** — push selective filters **forward** to shrink intermediate results. **[R1][L]**

**Q26. What is a straggler and why is it bad?**  
**A:** One operator/thread that takes **much longer** than peers on the same query → extends **end-to-end latency** (tail latency), underuses other cores. Example: one expensive step in a DAG (lecture: operator 8 ~10× others). **[L]**

**Q27. How does work stealing work in Grasper?**  
**A:** Idle threads take messages from **other threads’ RDMA RecvBufs** in priority order: same **region** (e.g., Traversal) on same node → other regions (Filter) → **other engines (nodes)**. Improves load balance; lecture asks about **race conditions** and **stealing complexity** (design uses ring buffers / per-thread recv queues to limit contention). **[L]**

**Q28. Thread binding — why bind a thread to a core?**  
**A:** Avoid **OS context-switch** overhead and improve **cache/NUMA locality**; experts grouped into **regions** so threads don’t bounce across cores. **[R1][L]**

**Q29. Gremlin query → execution artifact?**  
**A:** Parsed to **step objects**, optimized plan, then a **DAG** describing parallelism and message dependencies; Expert Model executes DAG with RDMA messaging patterns: **split, transfer, merge, spawn**. **[R1][L]**

---

## Section F — Evaluation & limitations

**Q30. Experimental cluster (lecture numbers)?**  
**A:** **10 machines**, 2× 8-core Xeon @ 2.1 GHz, **128 GB RAM**, Mellanox ConnectX-3 **40 Gbps RDMA**, **24** compute threads per machine. Benchmarks: **LDBC** (IC/IS queries), **AMiner** (real PG), **Twitter** (synthetic). **[L][R1]**

**Q31. Ablation study components?**  
**A:** Contributions of **RDMA**, **query optimization** (decompose/combine/reorder), and **work stealing** — each removed to show impact. **[L]**

**Q32. Why is scalability “sublinear” (e.g., 7× on 10 nodes)?**  
**A:** **Coordination**, **barrier steps**, **skew**, **network**, and **non-parallelizable** phases limit ideal 10× speedup; lecture shows per-query speedups below 10. **[L]**

**Q33. Twitter benchmark OOM on 2 nodes?**  
**A:** Grasper is **in-memory**; Twitter dataset doesn’t fit in **2-node** cluster memory — needs more nodes or smaller graph. **[L]**

**Q34. Lecture limitations / open questions on Grasper?**  
**A:** **Failure handling**? **Writes** and **ACID transactions**? **Durability** if HDFS is source of truth? Slide also jokes “no streaming/dataflow” despite data-flow engine — meaning not **unbounded stream processing** like Flink. **[L]**

**Q35. Compare Grasper to Neo4j / JanusGraph / TigerGraph — what should you say in exam?**  
**A:** Grasper targets **distributed OLAP** with **RDMA-native** storage and **Expert Model**; others often use **single-node or storage-backend** designs with lower CPU/network utilization under mixed Gremlin workloads (paper shows low utilization on Titan/JanusGraph). Apples-to-oranges if datasets/cluster differ — focus on **architecture differences**, not just raw numbers. **[R1][L]**

---

## Section G — Short “define in one line” (rapid fire)

| Term | Answer |
|------|--------|
| PG | Labeled, property-bearing directed graph |
| Gremlin | Procedural traversal language (TinkerPop steps) |
| Expert Model | Category-specific executors + adaptive parallelism + locality/load balance |
| V-KVS / E-KVS | RDMA-backed KV for vertex vs. edge labels/properties |
| Coordinator engine | Engine that owns query plan merge/barrier for a query |
| Ring buffer RecvBuf | Per-thread RDMA mailbox for incoming messages |
| IC4 | LDBC interactive **complex** query (heavy step skew in paper) |
| Stateful (lecture) | Graph partition lives with compute on engine |

---

## Section H — “Explain to the class” (2–3 minute prompts)

**H1.** Walk through **one Gremlin query** (e.g., top 3 airports by outgoing routes) and map each step to **Map/Filter/flatMap** and to a **sequential vs. barrier** flow.

**H2.** Draw **JanusGraph+FDB** vs. **Grasper** and explain **when** you’d separate storage from compute.

**H3.** Explain **why** Grasper splits **topology (normal)** and **properties (RDMA KV)** and give **one benefit** and **one cost**.

**H4.** Describe **Expert Model** three features and give **one example optimization** per feature.

**H5.** End with **honest limitations** (fault tolerance, writes, durability) — shows you read the last slide.
