# Grasper — Full primer (paper + lecture)

Read this if you want to **explain Grasper from zero** in a meeting or exam. Factual claims tie to **[R1]** (SoCC paper), **[L]** (lecture PDF), or **[R4]** in `[Grasper_References.md](Grasper_References.md)`.

---

## Part 0 — What you are learning

### Q: What is Grasper in one sentence?

**A:** Grasper is a **cluster of in-memory query servers** that run **Gremlin** analytics on **property graphs**, using **RDMA** for fast remote access and an **Expert Model** so each type of Gremlin step gets the right parallelism, cache, and threading. **[R1]**

### Q: What should I compare it to in this course?

**A:**

- **JanusGraph + FoundationDB (lecture):** graph **API** separated from **durable transactional KV** — good for scaling storage and OLTP-style backends, extra hop on every read.
- **Pregel/PowerGraph (paper intro):** **offline** bulk graph algorithms (PageRank), not millisecond **Gremlin OLAP**.
- **Neo4j:** often single-cluster Cypher; Grasper paper benchmarks against several **distributed** PG stores. **[R1][L]**

---



## Part 1 — Property graphs and Gremlin



### Q: What is a property graph?

**A:** Think “**nodes and arrows with stickers**.” Each **vertex** is an entity; each **edge** is a **directed** relationship. Stickers are **labels** (`person`, `knows`) and **properties** (`name=marko`, `weight=0.5`). Unlike a rigid SQL schema, one vertex can have `age` while another has `lang` only. **[R1][L]**

### Q: Why Gremlin for Grasper?

**A:** Gremlin is the **lingua franca** of TinkerPop systems (JanusGraph, etc.). It expresses analytics as a **pipeline of steps** — easy for traversals and pattern-like logic. Grasper **parses Gremlin** but executes its **own parallel plan** (DAG + messages). **[R1]**

### Q: Procedural vs declarative — intuition?

**A:**

- **SQL:** “Give me all customers who bought X” — optimizer picks joins.
- **Gremlin:** “Start at customers, walk `bought` edges, filter `product=X`” — **you** chose the walk order (though Grasper **reorders** steps when safe). **[L]**



### Q: Walk through “Who does Marko know?”

**A:**

1. `g.V()` — all vertices (start traversal).
2. `.has('name','marko')` — filter to Marko.
3. `.out('knows')` — follow **outgoing** `knows` edges.
4. `.valueMap()` — return properties of neighbors.

Swap `.out` → `.in` for “Who knows Marko?” **[L]**

### Q: What makes a query “complex” in the lecture?

**A:** It may touch **huge** graph regions — fraud rings through shared attributes, or recommendation walks over **hundreds of millions** of vertices under **50 ms** tails. That needs **parallel steps**, **RDMA**, and **load balancing**, not a single thread. **[L]**

---



## Part 2 — Why existing systems were not enough (paper §3)



### Q: What did Chen et al. measure on Titan/JanusGraph?

**A:** On LDBC **IC4**, time is **skewed** across steps — e.g. `hasLabel` and `in` dominate. Fixed parallelism **wastes** cores on easy steps and **starves** hard ones. Under mixed **IS1–IS4** load, **CPU and network utilization stayed low** — architecture + external storage + one-query-one-thread hurt throughput. **[R1]**

### Q: Four design goals Grasper takes from that?

**A:**

1. Execution model tuned for **OLAP Gremlin** (traversal, filter, aggregate).
2. **High parallelism inside** heavy queries and **many concurrent** queries.
3. **Native graph store** integrated with engine — no HBase round-trip per step.
4. **RDMA** for communication; components designed around one-sided ops. **[R1]**

---



## Part 3 — RDMA (lecture + paper §2)



### Q: Why does the lecture spend slides on copies?

**A:** To justify **why a graph system would reorganize around RDMA**. If every property fetch paid **four memcpy-style hops** over Ethernet, splitting topology and properties across nodes would be suicide. With **~microsecond** one-sided reads, remote property access is viable. **[L][R1]**

### Q: One-sided read in plain English?

**A:** Engine A says: “NIC, please **read 64 bytes** from Engine B’s **pre-published address** into my buffer.” CPU on B is not running recv() for that operation. Grasper’s **RecvBuf ring buffers** are still used for **thread messaging**, but data movement uses RDMA reads/writes. **[R1]**

---



## Part 4 — Architecture



### Q: What are the main components?

**A:**

- **Clients** (TCP/IP): submit Gremlin, get results.
- **Master**: monitoring + **assign queries** to engines by load.
- **Engines**: each has **data store** + **query engine**, linked by **RDMA**.
- **HDFS**: bulk **load** graph files; engines partition data in memory. **[R1][L]**



### Q: Stateful vs stateless (lecture diagram)?

**A:** **JanusGraph** instances are **stateless** regarding graph bytes — truth lives in **FoundationDB**. **Grasper** engines are **stateful** — each holds a **shard** of the graph in RAM and executes traversals locally, coordinating via messages/RDMA. **[L]**

---



## Part 5 — Storage layout (the heart of the paper)



### Q: Normal vs RDMA memory?

**A:**


| Memory     | Holds                                  | Access pattern                    |
| ---------- | -------------------------------------- | --------------------------------- |
| **Normal** | Adjacency, local topology, query temps | Local CPU pointers                |
| **RDMA**   | Label/property **values** in **KV**    | Local + **remote one-sided read** |


**[R1][L]**

### Q: Why not put properties next to every vertex on the same node?

**A:** Real graphs **skew**: celebrity vertices, fat property blobs, uneven labels. Co-location → **some engines full, others idle**. **Independent hash** of topology and properties **spreads load**; RDMA pays the distance tax. **[L][R1]**

### Q: Consequence examiners love?

**A:** You might find **adjacency locally** but need an **RDMA fetch** for a property or label stored on another engine. Lecture slide: “How to find an edge between two nodes?” — topology navigation is local-first; **complete edge record** may be distributed. **[L]**

### Q: String-ID maps?

**A:** At load time, strings (`"knows"`, `"name"`) map to **small integers** — saves RAM and RDMA bytes. **[R1]**

---



## Part 6 — Query execution



### Q: What flows through the engine?

**A:** **Messages** — packaged outputs of a **single Gremlin step** (vertex IDs, partial results, metadata). Big results **split** to stay under a size tuned for **low RDMA latency**. **[R1][L]**

### Q: Sequential vs barrier vs branch — story?

**A:**

- **Sequential:** “For each friend of Marko, check age” — many independent messages, **parallelize**.
- **Barrier:** “Count all friends” — must **see everything** before one number — **sync**.
- **Branch:** `and( cond1, cond2 )` — **two sub-walks** in parallel, then **merge**. **[R1][L]**



### Q: What is the Expert Model?

**A:** Instead of one generic interpreter, Grasper has **specialists**:

- **Traversal expert** handles `in`/`out`/…
- **Filter expert** handles `has`/…

Each expert chooses **how many threads**, **which cache policy**, and **how to route** outgoing messages. **Assumption:** all steps in a category share an **access pattern**. **[R1][L]**

### Q: Adaptive parallelism — numeric intuition?

**A:** If a filter step receives **48 messages** on an engine, run up to **48 worker threads** for that step (bounded by cores and stealing). If another step gets **2 messages**, don’t spawn 48 threads — saves overhead. **[R1][L]**

### Q: Query optimizations you should name?

**A:**

1. **Decomposition** — `and`/`or` → parallel subqueries.
2. **Combination** — merge back-to-back `has()` into one scan.
3. **Reordering** — run selective filters **before** expensive expansions. **[R1][L]**



### Q: Stragglers and work stealing?

**A:** A **straggler** is the slowest worker in a parallel step — everyone waits at barriers. **Work stealing** lets idle threads drain **RecvBuf** queues from busier peers (local region first, then remote engines). **[L]**

---



## Part 7 — Evaluation & honesty



### Q: What hardware should you remember?

**A:** **10-node** cluster, **40 Gbps** RDMA NICs, **24** threads/engine, **128 GB** RAM — lecture numbers align with paper experiments. **[L][R1]**

### Q: What does ablation show?

**A:** Turning off **RDMA**, **plan optimizations**, or **work stealing** hurts performance — none is decorative. **[L]**

### Q: Why sublinear speedup?

**A:** **Barriers**, **coordination**, **skewed partitions**, and **non-parallel** pieces mean **10 nodes ≠ 10×** on every query. **[L]**

### Q: What should you say about limitations?

**A:** Grasper is optimized for **read-heavy OLAP** loaded from HDFS. The lecture explicitly asks about **failures**, **online writes**, **ACID transactions**, and **durability** — not Grasper’s focus in the paper. Admitting that shows depth. **[L]**

---



## Part 8 — How to present in 3 minutes

1. **Hook:** “Interactive Gremlin on billion-edge graphs needs low tail latency.”
2. **Pain:** External KV + low utilization on Titan/JanusGraph.
3. **Idea:** Native in-memory PG + **split topology/properties** + **RDMA**.
4. **Engine:** Messages, DAG, **Experts**, adaptive parallelism, stealing.
5. **Proof:** LDBC/AMiner/Twitter; ablations.
6. **Close:** Great for **OLAP reads**; not a general **transactional** graph DB.

---



## Where to drill next

- `[Grasper_Exam_QA.md](Grasper_Exam_QA.md)` — 35+ practice questions  
- `[Grasper_Flashcards.md](Grasper_Flashcards.md)` — closed-book rehearsal  
- `[Grasper_Lecture_Slides.md](Grasper_Lecture_Slides.md)` — slide-by-slide map to PDF

