# FoundationDB — Complete Primer (read straight through)

**Paper:** Zhou et al., FoundationDB SIGMOD paper. **[P]**  
**Concurrency control:** [`DiscussionXactCC-1.pdf`](DiscussionXactCC-1.pdf). **[CC]**  
**Graph + eBay:** [`2026_FoundationDB_and_Its_Use_for_A_Graph_Database_v3.pdf`](2026_FoundationDB_and_Its_Use_for_A_Graph_Database_v3.pdf). **[G]**

| If you want… | Read… |
|--------------|--------|
| One continuous story | This file, Parts 0–7 |
| Lookup tables | [`FoundationDB_Study_Guide.md`](FoundationDB_Study_Guide.md) |
| Mock exam | [`FoundationDB_Exam_QA.md`](FoundationDB_Exam_QA.md) |

---

# Part 0 — Why concurrency control matters (before FoundationDB)

Modern servers have **many CPU cores** and **large main memory**. A database management system that runs **one transaction at a time** leaves most cores idle. For example, on a 10-core machine, single-threaded execution wastes about **90%** of the CPU capacity. **[CC]**

**Goal of concurrency control:** run many transactions **at the same time** on different cores, but make the result **look as if** transactions ran **one after another** (no overlapping), when that is required for correctness.

A **serial schedule** is an order where each transaction finishes before the next starts. With **n** transactions there are **n!** possible serial orders. A schedule is **serializable** if its final database state equals **some** serial schedule. **[CC]**

**Strict serializability** (what FoundationDB targets) is stronger: if transaction B **starts after** transaction A **commits**, B must not behave as if it ran **before** A. Real-time order is respected. **[CC][P]**

### Classic bank example (why interleaving is dangerous)

- **T0:** transfer 50 from account A to B.  
- **T1:** transfer 10% of A’s balance to B.  

If A starts at 1000 and B at 2000, running **T0 then T1** or **T1 then T0** both keep total balance **3000**. But **interleaved** reads and writes without control can leave A=950, B=2100 (total **3050**) — money appears from nowhere. **[CC]**

**Conflict intuition** on the same data item **Q**:

- read + read: order usually does not matter.  
- read + write, write + read, write + write: **order matters** for correctness.

### Three families of protocols (lecture map to FoundationDB)

1. **Lock-based (two-phase locking):** shared (read) and exclusive (write) locks; **two-phase locking** = all locks acquired in a growing phase, then released in a shrinking phase → serializable schedules. Risks: **deadlocks** (detect cycles in wait-for graph), **dirty reads**, **unrepeatable reads**, **lost updates** if locking is too weak. **[CC]**

2. **Timestamp ordering:** each transaction gets a unique timestamp; reads/writes compared to per-item read/write timestamps; violations → **rollback**. No deadlocks, but more aborts. **[CC]**

3. **Optimistic concurrency control:** assume conflicts are **rare**. **Read phase** (work on private copies), **validation phase** (check conflicts), **write phase** (publish). FoundationDB uses **optimistic** conflict checking at commit time. **[CC][P]**

4. **Multi-version concurrency control:** keep **several versions** of each key; each transaction reads the version visible at its **read timestamp**. Writes create new versions. FoundationDB combines **multi-version** reads with **optimistic** validation. **[CC][P]**

---

# Part 1 — What is FoundationDB?

**FoundationDB** is an open-source **ordered transactional key-value store**. Keys are sorted; you get **multi-key strictly serializable transactions** across the **entire** key space. **[P][G]**

It sits in the **lower half** of a database stack:

- **Not** bundled with one fixed relational schema or query language.  
- **Yes:** a small, careful feature set so **layers** build document, relational, or **graph** systems on top (Record Layer, JanusGraph storage backend, CouchDB rebuild, etc.). **[P][G]**

**Atomicity, consistency, isolation, durability (ACID)** with **strict serializability** by default; applications can relax isolation for **snapshot reads** when they accept fewer conflicts. **[P]**

Used in production at **Apple**, **Snowflake**, and others; open-sourced under Apache 2.0 (2018). **[P][G]**

**Design themes:** **[P]**

- **Divide and conquer (unbundled):** separate **transaction system** (writes), **log system**, **storage system** (reads); scale each independently.  
- **Treat failure as normal:** on failure, **shut down the transaction system** and **recover** through one well-tested path; aim for **mean time to recovery** under ~5 seconds in production.  
- **Deterministic simulation:** test the real code under randomized faults in one process; bugs are **reproducible**.

**Fault tolerance without 2f+1 everywhere:** for many components, **f** failures tolerated with **f+1** replicas (not always **2f+1** quorum masking), because failures trigger **reconfiguration** instead of only quorum reads. **[P]**

---

# Part 2 — Deployment and architecture

### Shared-nothing cluster **[G]**

Each node has its own CPU, memory, and disk; no shared RAM or shared disk between nodes. Same big idea as Gamma and Grasper lectures.

**Guest lecture deployment picture:** hundreds of nodes per data center; example **3 data centers, 2 regions**; cross–data-center latency ~20–30 ms, inner data center ~0.2 ms; tolerate **one data center failure**; key-value **triple-replicated per data center** (six copies total) so each data center can survive **two storage node failures**. **[G]**

### Control plane (metadata and orchestration) **[P][G]**

| Component | Role |
|-----------|------|
| **Coordinators** | Disk **Paxos** group; store critical cluster metadata |
| **ClusterController** | Monitors servers; recruits singleton roles |
| **DataDistributor** | Failure detection; **balances shards** across storage servers |
| **RateKeeper** | **Overload protection** |

### Data plane **[P][G]**

| Component | Role |
|-----------|------|
| **Sequencer** (singleton per epoch) | Assigns **read version** and **commit version** (timestamps); recruits proxies, resolvers, log servers |
| **Proxies** (stateless) | Hand out read versions; **orchestrate commit** |
| **Resolvers** (stateless, range-partitioned) | **Optimistic conflict check** on key ranges |
| **Log servers** | Durable **write-ahead log** shards (replicated queues) |
| **Storage servers** | Hold **key-range shards**; serve **versioned reads**; distributed **B-tree** (SQLite-based engine per server) |

**Read-write separation:** clients **read directly from storage servers** at a version; writes go through **transaction system + log**; storage servers **pull** and apply logs in the background. More storage servers → more read throughput; more proxies, resolvers, log servers → more write throughput. **[P]**

---

# Part 3 — One transaction, step by step

FoundationDB uses **optimistic concurrency control** plus **multi-version concurrency control**. **[P][G]**

### Phase A — Get a read version

1. Client asks a **proxy** for a **read version** (a logical timestamp).  
2. Proxy asks **sequencer** for a version **at least as large as** all prior **commit versions**.  
3. Client caches which **storage servers** own which **key ranges**.

### Phase B — Reads

4. Client reads keys **directly from storage servers** at that **read version** (multi-version data in memory for recent window, durable latest value on disk).  
5. **Writes are buffered only on the client** until commit. **Read-your-writes:** client merges local write buffer with read results.

### Phase C — Commit

6. Client sends **read set** and **write set** (key ranges touched) to a proxy.  
7. Proxy gets **commit version** from sequencer (sequencer advances versions at ~**1 million per second**).  
8. Proxy sends transaction to **resolvers** (one per key-range partition). Resolvers check **read-write conflicts** (similar in spirit to write-snapshot isolation, but commit version is chosen **before** validation).  
9. If **any** resolver reports conflict → **abort** (client may retry). Production conflict rate often **under 1%**. **[P]**  
10. If clean → proxy appends to **log servers**; commit completes when required replicas acknowledge **durable** log.  
11. Proxy tells sequencer the committed version; client gets success.  
12. **Storage servers** asynchronously pull log records and apply updates.

**Read-only transactions:** can **commit locally** without full write protocol if there are **no writes** — important because most transactions are read-only. **[P][G]**

**Snapshot reads:** relax isolation so concurrent writes do not conflict with the read (fewer aborts, weaker guarantees). **[P]**

### Strict serializability story **[P]**

Read and commit versions come from one **sequencer**. Commit version acts as **log sequence number** ordering commits. A transaction’s read version sees all commits before it; commit version places it in a total order → **strict serializability**.

**Resolver nuance (quiz from guest lecture):** a transaction may touch **multiple resolvers**. **All** must admit. If one sees conflict, transaction aborts; other resolvers might have already recorded state → rare **false-positive** conflicts for later transactions within the short **multi-version window (~5 seconds)**. Correctness is preserved. **[G][P]**

---

# Part 4 — Storage server reads, logging, recovery, simulation

### How a storage server answers `Get(key, version)` **[G]**

- Picks the **largest stored version ≤ read version**.  
- **Future version** error if read version is ahead of what is applied yet (client may wait, try another replica, or restart).  
- **Version too old** if outside the **retention window** (lecture: ~**5 seconds** of versions in memory).  
- Latest committed value also persisted (e.g. SQLite). **[G][P]**

If a replica lags, client can **wait**, **retry another replica**, or **restart transaction**. Proxies may send **empty transactions** to advance **known committed version** when idle (write-path scalability detail). **[G]**

### Logging **[P]**

Commit path: proxy makes log **durable on log servers** (synchronous replication **f+1**). Shipping log to storage servers is **not** on the critical commit path. Storage servers often apply redo to **in-memory** index before new read versions are handed out, so reads usually see fresh data quickly.

### Recovery (failure → new epoch) **[P][G]**

When transaction system or log system fails, **sequencer terminates**; **cluster controller** recruits a new sequencer → new **epoch**.

Key versions:

- **Known committed version (KCV):** max log sequence number this proxy has committed.  
- **Durable version (DV):** max log sequence number persisted on a log server.  
- **Previous epoch end version (PEV):** max of KCVs from old log servers — fully replicated history.  
- **Recovery version (RV):** min of DVs — end of redo; discard after RV.

Logs in **[PEV+1, RV]** may be **copied** to heal replication. **No ARIES-style undo** on data pages: rollback = drop in-memory versions **after RV**. Recovery is cheap because **redo forward** is the normal path.

### Deterministic simulation **[P]**

Before trusting production, FoundationDB runs **real server code** in a **single-process discrete-event simulator**: simulated network, disk, time, randomness. **Flow** language (actors on one thread per core in production). **Fault injection:** machine, rack, data center, partitions, latency, disk corruption. **Swarm testing:** random cluster sizes, workloads, faults. When a bug is found, **determinism** makes it **reproducible**. Simulation does not fully replace **performance** testing.

---

# Part 5 — Programming model (guest lecture)

FoundationDB exposes a simple **key-value application programming interface**: **get**, **set**, **getRange**, **clear** (point or range). Higher layers map relational or graph rows to **ordered keys**. **[G]**

Example mental model: employee and department tables encoded as keys; **getRange** scans a prefix for “all employees in department toy.”

**Layers on FoundationDB:** Record Layer (relational), document layer, **FDBKeeper** (coordination), **JanusGraph** (graph). **[G][P]**

---

# Part 6 — JanusGraph on FoundationDB (eBay graph service)

### Architecture **[G]**

- **JanusGraph:** open-source graph engine; **Gremlin** query language via **Apache TinkerPop**; storage backend can be FoundationDB, Cassandra, HBase, etc.  
- **eBay:** graph **service** with JanusGraph using **FoundationDB** as durable store; **billions** of vertices and edges.  
- Contrast **Grasper:** Grasper keeps graph in **engine RAM** for OLAP; JanusGraph+FoundationDB is **transactional durable key-value** under a graph layer — good for **online transaction processing** graph workloads, extra hop per storage access.

### Data layout **[G]**

JanusGraph maps graph elements to **keys and values** (and sometimes key-column-value). Adding a vertex, edge, or property becomes **set** / **getRange** operations. Example: outgoing routes from vertex 1 → **getRange** on prefix like `/1/route_out`. **Composite index** (e.g. airport code) → one **get** to find vertex id. **Vertex-centric index** on edges (e.g. route distance sorted) → narrower **getRange** on a supernode like LAX.

**Index quiz (guest deck):**

- **Composite index:** hash-style equality; **not** general range on arbitrary fields alone.  
- **Vertex-centric index:** **supports range** on that vertex’s edges (local).  
- **Mixed index** (Elasticsearch, Solr): **global** text/range search, external to FoundationDB.

### Multi-hop queries and latency **[G]**

Gremlin runs in **waves** of parallel **getRange** calls. One **large wave** (e.g. fetch all edges from a **supernode**) often dominates latency.

**Per-hop subquery result cache (eBay):**

- Split query into **templates** (e.g. `outE('route').has('IsActive', ?).inV().has('country', ?)`).  
- Cache key includes template id and parameters.  
- **Read path:** if cache hit, skip huge **getRange**; else traverse and optionally **enqueue** background fill.  
- **Write path (write-around):** graph **mutations** and **cache invalidations** in the **same FoundationDB transaction** → **strong consistency** (Middleware 2014 cache-augmented SQL idea).  
- Invalidation cost depends on case: edge add/remove (few keys), root vertex update (**clearRange** on prefix), leaf/supernode delete (expensive — batch edge deletes first).

**Full-query result cache:** great for **batch load then read-only** (machine learning pipelines); **invalidation expensive** for churning graphs.

**FoundationDB as cache storage:** cache entries can live in the same transactional store as graph keys.

---

# Part 7 — How the pieces connect for exams

| Topic | FoundationDB choice |
|--------|---------------------|
| Concurrency | **Optimistic** validation + **multi-version** reads |
| Ordering | **Sequencer** versions → strict serializability |
| Scale reads | **Storage servers**, direct client reads |
| Scale writes | **Proxies, resolvers, log servers** |
| Failure | New **epoch**, **recovery version**, fast redo |
| Testing | **Deterministic simulation** + fault injection |
| Graph | **JanusGraph layer** + optional **per-hop cache** |

**vs Titan/JanusGraph on HBase (Grasper paper):** external storage per step → low utilization; FoundationDB is the **native transactional** layer JanusGraph can call into. **[G]** (ties to Grasper folder)

---

**Next:** [`FoundationDB_Study_Guide.md`](FoundationDB_Study_Guide.md) · [`FoundationDB_Exam_QA.md`](FoundationDB_Exam_QA.md)
