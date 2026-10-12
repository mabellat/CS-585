# Gamma — Complete Primer (zero background assumed)

**Paper:** David J. DeWitt et al. *The Gamma Database Machine Project.* **IEEE TKDE**, March 1990. **[P]**  
**Lecture:** CSCI 585 [`GammaLecture.pdf`](GammaLecture.pdf). **[L]**

| If you want… | Read… |
|--------------|--------|
| Full teaching from scratch | This file, Parts 0–11 in order |
| Shorter reference | [`Gamma_Study_Guide.md`](Gamma_Study_Guide.md) |
| Drill | [`Gamma_Flashcards.md`](Gamma_Flashcards.md) |
| Mock exam | [`Gamma_Exam_QA.md`](Gamma_Exam_QA.md) |
| Slide pass | [`Gamma_Lecture_Slides.md`](Gamma_Lecture_Slides.md) |

Tags: **[P]** = paper, **[L]** = lecture.

---

# Part 0 — What is Gamma?

### Q: What is it in one sentence?

**A:** Gamma is a **parallel relational database** on many **shared-nothing** computers. You write **SQL**; the system **declusters** tables, runs **operators** on many nodes, and uses **dataflow** so tuples stream between operators without a central router on every page. **[P][L]**

### Q: Why should I care in 2026?

**A:** The lecture argues Gamma targeted problems like today’s web-scale warehouses **before** those companies existed. **SQL hides parallelism**—same idea as **Amazon Redshift** and **Google BigQuery**. Gamma won the **ACM Software System Award (2008)**. **[L]**

### Q: What did DIRECT teach them?

**A:** DIRECT (earlier Wisconsin machine) proved parallelism could help, but **shared memory**, **heavy control traffic**, and using parallelism **instead of indexes** failed when **I/O** was the bottleneck. Gamma went **shared-nothing**, **hash-based** algorithms, and **dataflow** with few control messages. **[P]**

---

# Part 1 — The machine picture

### Q: What is shared-nothing?

**A:** Each **node** has its own CPU, RAM, and disk. Nodes talk over a **network**. No shared disk, no shared RAM across the cluster. Stonebraker’s “case for shared nothing.” **[P][R3][L]**

### Q: What is a fragment / shard?

**A:** One **horizontal slice** of a table stored on one node’s disk. Gamma may also build **B+-tree** or **hash** indexes **on that fragment only**. **[P][L]**

### Q: Virtual nodes (lecture)?

**A:** One physical 8-core server can be split into **eight logical nodes**, each with a fraction of CPU, memory, and disk—same programming model as many small machines. **[L]**

### Q: Speedup vs scaleup?

**A:** **Speedup:** same data size, **more nodes** → hope query finishes faster (e.g. 10% raise for all employees). **Scaleup:** data grows **10×**, add **10× nodes** → hope response time **stays** similar (e.g. average salary query as DB grows). **[P][L]**

---

# Part 2 — Three big ideas (memorize)

1. **Horizontal declustering** — spread a table across many disks so scans use **aggregate bandwidth**. **[P]**
2. **Hash-based parallel join/aggregate** — partition work by hash so **no central** join bottleneck. **[P]**
3. **Dataflow scheduling** — operators connected by **tuple streams**; scheduler sends **control only** (~3 messages per operator per participating processor). **[P][R1][L]**

---

# Part 3 — How rows are placed (declustering)

### Q: Three strategies?

**A:** **Round-robin** (tuple i → disk i mod N), **hash** (hash(key) mod N), **range** (DBA defines value bands per site). Stored in the **catalog**; optimizer uses them to decide **which nodes** run the query. **[P][L]**

### Q: No partitioning attribute?

**A:** **Random** or **round-robin**. Any query that needs the whole table hits **all** nodes. **[L]**

### Q: Hash + predicates (lecture examples)?

**A:** `salary = 60K` on partitioning attribute → **one** node. `salary < 50K` (range) or `age = 20` (not partition key) → **all** nodes. **[L]**

### Q: Range?

**A:** Predicates on the partitioning attribute can go to **only nodes whose ranges overlap**—good for selective range queries (HRPS extends this idea later). **[P][L]**

### Q: Query results?

**A:** Paper default: store results **round-robin** (or hash) so every result fragment has ~equal size. **[P]**

---

# Part 4 — Hardware generations

### Q: Gamma 1.0 (VAX)?

**A:** 17 VAX 11/750s, 2 MB RAM, token ring but **Unibus** bottleneck; 8 nodes with disks, 9 **diskless** for join experiments. Lesson: **2 KB pages** were too small for network packets. **[P]**

### Q: Gamma 2.0 (iPSC/2)?

**A:** **32** nodes, **80386**, **8 MB** RAM, **330 MB** disk, **45 KB** SCSI read cache, hypercube links ~2.8 MB/s. **NOSE** = thread package inside one process per node. **[P][L]**

---

# Part 5 — Who runs what (processes)

### Q: Main processes?

**A:** **Catalog Manager** (schemas), **Query Manager** per user (parse, optimize, compile), **Scheduler** per **multi-site** query (start operators, collect “done”), **operator processes** on nodes (select, join, store, …), plus deadlock detector and recovery/log managers. **[P]**

### Q: Single-site query shortcut?

**A:** If the plan uses **one** node, QM talks **directly** to that node—**no scheduler**. Same pattern as HRPS. **[P][L]**

### Q: Why primed operators?

**A:** Forking a process per operator was too slow; at init Gamma starts a **pool** of operator processes and assigns work via a **“new task”** port. **[P]**

### Q: Lecture view of a node?

**A:** Fixed **threads** share memory; all listen on a socket; scheduler message says which **role** (select, hash-join build, probe, …) via a switch. **[L]**

---

# Part 6 — Dataflow and the split table

### Q: What does an operator see?

**A:** Input = **stream of tuples** (like a pipe). Output tuples go to destinations chosen by a **split table** (demux). Code is written **as if single-node**. **[P][L]**

### Q: What does the scheduler do?

**A:** **Control only:** which operators start on which nodes, when the query is finished. It does **not** route every tuple. **[L]**

### Q: End of a stream?

**A:** Operator closes outputs → **end-of-stream** to downstream → **control message** to scheduler (“I’m done”). Batches are ~**8 KB**. **[P][L]**

### Q: vs DIRECT?

**A:** DIRECT needed control per **page** of data. Gamma needs about **3 control messages per operator per processor** for the whole query. **[P]**

---

# Part 7 — Joins (especially hybrid hash)

### Q: Parallel join idea?

**A:** Hash both sides on **join key** into buckets; only matching keys in the **same bucket** can join. **[P]**

### Q: Algorithms Gamma implemented?

**A:** Sort-merge, Grace, simple hash, **hybrid hash** (became **default** after SIGMOD ’89 study). **[P][R4]**

### Q: Hybrid hash in plain English?

**A:** Build a hash table from the **first bucket** of the inner relation in RAM; spill other buckets to disk; probe with first bucket of outer; then join remaining buckets from disk. **First buckets** can stay in memory without writing. **[P][L]**

### Q: Parallel hybrid hash?

**A:** **Split tables** route tuples to **join nodes** (can be **diskless**); partition step overlaps with build/probe; results stored back on data nodes (e.g. round-robin). **[P]**

### Q: Build vs probe (lecture)?

**A:** **Sequential phases** on one node: **build** hash table on inner, then **probe** with outer. Parallel version pipelines across nodes. **[L]**

### Q: Not enough RAM (100 GB Emp, 16 GB memory)?

**A:** Partition **both** relations into **M buckets** with same hash; for each i, load bucket i of inner, probe with bucket i of outer. **[L]**

---

# Part 8 — Selection, aggregate, update

### Q: Parallel selection?

**A:** Run on all **relevant** nodes; if predicate uses **partitioning attribute** with hash/range, may be **subset**; one-page read-ahead on scans. **[P]**

### Q: Parallel aggregate?

**A:** Partial aggregates locally → **redistribute** by hash on GROUP BY → finalize. **[P]**

### Q: Update changes partition key?

**A:** Tuple is **not** updated in place; **split table** sends it to the node that owns the new key. **[P]**

---

# Part 9 — Transactions and recovery

### Q: Concurrency?

**A:** **2PL** at file/page granularity (S, X, IS, IX, SIX). **Local** lock manager + wait-for graph; **central** deadlock detector merges graphs periodically and aborts the transaction with **fewest locks**. **[P]**

### Q: Recovery (paper status)?

**A:** Design toward **WAL**, **LSN** in log records, **ARIES**-style undo; group commit. Survey describes direction while implementation was in progress. **[P]**

---

# Part 10 — Staying up when a disk node dies

### Q: Chained declustering?

**A:** Primary fragment on disk i; **backup** on disk **(i+1) mod M**. On failure, reads use primary/backup mix; load of failed node shifts to neighbors (~**1/(M−1)** extra per survivor). **No data copy** on failover—update **fragment map** in memory. **[P][R5][L]**

### Q: Interleaved declustering (Teradata)?

**A:** Backup spread across cluster; **two** failures can hurt availability; Teradata suggests **small** clusters (4–8). Lecture: failed node’s load spreads to **0, 2, 3**; chained pushes along the **chain**. **[P][L]**

---

# Part 11 — Did it work? (evaluation)

### Q: Benchmark?

**A:** Wisconsin benchmark, 208-byte tuples, 8 KB pages, 2 MB buffer pool; often hash on `Unique1`; up to **30** disk nodes; results **stored in DB** (not shipped to host). **[P][R7]**

### Q: Superlinear speedup—why?

**A:** One disk holds **most** of a big relation; declustered over 30 disks each holds **~2%** → **shorter seeks** (seek ∝ √ distance). Multi-zone disks: more spindles → more **outer-zone bandwidth** too. **[P][L]**

### Q: Indexed selection insight?

**A:** Clustered index time tracks **rows retrieved** more than raw table size; single-tuple select on partition key → **one node**. **[P]**

---

# Part 12 — Contributions checklist (lecture)

- Declustering (incl. **HRPS**, **MAGIC** next lecture)
- **Hybrid hash-join**
- **Chained declustering**
- **Dataflow** execution
- **Split table** / split operator
- Nodes **with** and **without** disk
- **Speedup & scaleup** methodology
- **SQL hides parallelism**

**[L]**
