# Gamma: The Database Machine Project

**Full citation:** David J. DeWitt, Shahram Ghandeharizadeh, Donovan A. Schneider, Allan Bricker, Hui-I Hsiao, Rick Rasmussen. *The Gamma Database Machine Project.* **IEEE TKDE** 2(1): 44–62, March 1990. University of Wisconsin–Madison. **[P]**

**Lecture:** CSCI 585 [`GammaLecture.pdf`](GammaLecture.pdf) (132 slides). Much of the deck is SQL/relational-algebra review; Gamma-specific material starts ~slide 40 and returns ~slide 103. **[L]**

**Also in this folder:** [`Gamma_Full_Primer.md`](Gamma_Full_Primer.md) · [`Gamma_Flashcards.md`](Gamma_Flashcards.md) · [`Gamma_Lecture_Slides.md`](Gamma_Lecture_Slides.md) · [`Gamma_Exam_QA.md`](Gamma_Exam_QA.md) · [`Gamma_References.md`](Gamma_References.md)

---

## Table of contents

1. [Why Gamma matters](#1-why-gamma-matters)
2. [From DIRECT to Gamma](#2-from-direct-to-gamma)
3. [Three key ideas](#3-three-key-ideas)
4. [Hardware: VAX prototype vs iPSC/2](#4-hardware-vax-prototype-vs-ipsc2)
5. [Horizontal declustering](#5-horizontal-declustering)
6. [Software processes](#6-software-processes)
7. [Dataflow execution and the split table](#7-dataflow-execution-and-the-split-table)
8. [Query path: single-site vs multi-site](#8-query-path-single-site-vs-multi-site)
9. [WiSS and NOSE](#9-wiss-and-nose)
10. [Parallel algorithms](#10-parallel-algorithms)
11. [Transactions, logging, availability](#11-transactions-logging-availability)
12. [Speedup, scaleup, superlinear effects](#12-speedup-scaleup-superlinear-effects)
13. [Lecture-only topics](#13-lecture-only-topics)
14. [Gamma contributions (slide checklist)](#14-gamma-contributions-slide-checklist)
15. [Exam cheat sheet](#15-exam-cheat-sheet)

---

## 1. Why Gamma matters

Gamma is a **relational database machine** that runs **SQL** while executing queries in **parallel** across many nodes. The user writes normal SQL; the system chooses plans, partitions, and message patterns. **[P][L]**

**Modern relevance (lecture):** Built before today’s web giants, but aimed at the same shape of problem—large data, parallel scans and joins. Comparable ideas appear in **Amazon Redshift**, **Google BigQuery**, and other MPP SQL warehouses. **[L]**

**ACM Software System Award (2008):** Lasting influence on concepts and commercial systems. **[L]**

**Scale target:** Shared-nothing designs like Gamma (and Teradata) are argued to scale to **hundreds or thousands** of processors; the TKDE paper reports **linear speedup** on selection/join in many cases and encouraging **scaleup**. **[P]**

---

## 2. From DIRECT to Gamma

**DIRECT** (earlier Wisconsin prototype) showed parallelism **could** work for databases but had fatal limits: **shared memory**, **centralized control**, and using parallelism **instead of indexes**—bad when I/O bandwidth is the bottleneck (Bitton et al. evaluation). **[P][R7]**

**Gamma’s fixes (straightforward in hindsight):**

| DIRECT problem | Gamma response |
|----------------|----------------|
| Shared memory / centralized scheduler | **Shared-nothing** + **hash-based** decentralized algorithms **[P][R3]** |
| Control messages per page | **Dataflow**: tuples stream between operators; **~3 control messages per operator per processor** **[P][R1]** |
| I/O bottleneck | **Horizontal declustering** across many disks **[P]** |

---

## 3. Three key ideas

Memorize as a list for exams:

1. **Horizontal partitioning (declustering)** — Relations split across **many disk drives** so scans and selections can run **in parallel** with little coordination. **[P]**
2. **Hash-based parallel algorithms** — Joins and aggregates use **partitioning by hash** so work is decentralized (no central bottleneck). **[P]**
3. **Dataflow scheduling** — Multi-operator queries run as **pipelines** of processes; **data moves without centralized routing**; scheduler sends only **control** (start/done). **[P][L]**

---

## 4. Hardware: VAX prototype vs iPSC/2

### Gamma 1.0 (VAX) **[P]**

- **17** VAX 11/750s, **2 MB** RAM each (no virtual memory in Gamma’s OS—tight memory).
- **Token ring** (~80 Mbit/s) but **Unibus ~4 Mbit/s** bottleneck; small network buffers caused retransmits.
- **8** nodes with **333 MB** Fujitsu disks; **9 diskless** nodes used for join/aggregate experiments.
- Lessons: **2 KB** pages tied to network packets was a mistake; **larger pages** won.

### Gamma 2.0 (iPSC/2 hypercube) **[P][L]**

- **32** processors: **80386**, **8 MB** RAM, **330 MB** Maxtor disk each.
- Hypercube links ~**2.8 MB/s** per channel; small messages as datagrams, large messages over a **circuit**.
- **45 KB** SCSI read cache on each drive (read-ahead).
- Gamma runs **NOSE** as a **thread package** inside one NX/2 process per node (Intel OS had few heavyweight processes).
- Port pain: VAX nodes numbered from **1**, hypercube from **0**—many off-by-one bugs in message targets.

**HRPS experiments** (separate paper) often use **24** DB nodes + **8** load-generator nodes on the same hypercube class machine. **[HRPS context]**

---

## 5. Horizontal declustering

**Goal:** Use **aggregate I/O bandwidth** of all disks; parallelize selection by starting an operator on each relevant node. **[P]**

**Three strategies** (stored in catalog; optimizer uses them to **prune** nodes): **[P][L]**

| Strategy | Placement | Predicate on partitioning attribute |
|----------|-----------|-------------------------------------|
| **Round-robin** | Next tuple → next disk | **All** nodes (default for query results) |
| **Hash** | `hash(key) mod N` | **Equality** → one node; **range** → all nodes |
| **Range** | DBA-defined bands per site | **Equality or range** → overlapping nodes only |

**Indices:** Index built **per fragment** (clustered or non-clustered). Clustered index **need not** be on the partitioning attribute (unlike some mainframe designs). **[P]**

**Retrospective mistake (paper):** Declustering **every** relation across **all** disk nodes; better to use relation **“heat”** and partial declustering (Bubba). **[P][R6]**

---

## 6. Software processes

```text
Host → Query Manager (per user) → Catalog Manager
              ↓
     Scheduler (per multi-site query) → Operator processes on each node
              ↓
     Deadlock detector, Recovery, Log managers
```

| Process | Role |
|---------|------|
| **Catalog Manager (CM)** | Schemas; loaded when DB opened; keeps caches consistent across users. **[P]** |
| **Query Manager (QM)** | Per active user; parse, optimize, compile; ad-hoc **gdl** (Quel variant) or embedded SQL in C. **[P]** |
| **Scheduler** | Per **multi-site** query; **activates** operators on chosen nodes; almost no CPU. **[P]** |
| **Operator process** | Runs select/join/store/etc.; **primed at init** to avoid fork cost; assigned via **“new task”** port. **[P]** |
| **Dispatcher** | Assigns idle scheduler to a query; caps active schedulers (load control). **[P]** |

**Lecture alignment:** Threads on a node share memory; fixed pool of threads listen on a socket; scheduler message tells thread which **role** (select, hash-join build, probe, …). **[L]**

---

## 7. Dataflow execution and the split table

**Core idea:** Operator code is written **as if single-node**. Input = **stream of tuples**; output routed by a **split table** (demux). **[P][L]**

- Tuples sent in **8 KB batches** (except last batch). **[P]**
- On **end of input stream**: close outputs → **end-of-stream** to destinations → **control message** to scheduler (“done”). **[P][L]**
- **Scheduler does NOT route data**—only **control** (initiate operator, collect done). **[L]**

**Split table types:** Three kinds depending on operator (e.g. join: hash join attribute → destination processor/port). **[P]**

**vs DIRECT:** DIRECT needed control per data page; Gamma ≈ **3 × (operators) × (processors per operator)** control messages for the query. **[P]**

**Hybrid hash-join example (paper):** Selection/scan on data nodes; tuples hashed to **diskless** join nodes; build then probe; results **round-robin** stored back on data nodes. **[P]**

---

## 8. Query path: single-site vs multi-site

1. Parse, optimize (**left-deep** trees on VAX due to memory; bushy plans studied later). **[P]**
2. Optimizer embeds **partitioning info** → plan knows **which nodes** participate.
3. **Single-site query:** QM sends **directly** to that processor—**no scheduler**. **[P][L]** (Same idea as HRPS Query Manager bypass for one-node queries.)
4. **Multi-site:** QM connects to **dispatcher** → **scheduler** → initiate operators on all participating nodes → QM waits → returns results.

---

## 9. WiSS and NOSE

**NOSE:** Lightweight processes, **shared memory**, **non-preemptive** scheduling (avoid convoys), hypercube messaging. **[P]**

**WiSS (Wisconsin Storage System):** Sequential files, byte streams, **B+ trees**, long fields, sort, scan. **Clustered** attribute optional. Scan predicates can be compiled to **386 machine code** for speed. **[P][R8]**

**Lecture:** Each node stores its **fragment** locally; **local** concurrency control and recovery; indexes on the fragment. **[L]**

---

## 10. Parallel algorithms

### Selection **[P]**

- Parallelize by running on **all relevant** nodes with disks.
- If predicate on **partitioning attribute** and hash/range → **subset** of nodes.
- **One-page read-ahead** on sequential/clustered-index scans.

### Join **[P][L]**

- Partition both inputs into **buckets** (same hash on join key); matching tuples in same bucket.
- Implemented: **sort-merge, Grace, simple hash, hybrid hash**; **hybrid hash default** after study **[R4]**.
- **Hybrid hash (central idea):** Build in-memory table from **first bucket** of inner relation while spilling other buckets; probe with first bucket of outer; join remaining buckets from disk. **Parallel version:** partitioning split table + **joining split table**; **diskless nodes** can execute join; overlap partition with build/probe.
- **Non-equijoins** were not supported initially (design in progress in paper).

### Aggregate **[P]**

- Partial aggregates per node → **redistribute** by hash on **GROUP BY** → finalize per partition.

### Update **[P]**

- If update changes **partitioning attribute**, tuple routed through **split table** to correct site (not written locally).

---

## 11. Transactions, logging, availability

### Concurrency **[P]**

- **Two-phase locking**; granularities **file** and **page**; modes S, X, IS, IX, SIX.
- **Per-site** lock manager + local wait-for graph.
- **Centralized** deadlock detector periodically collects local graphs; interval adapts (1–60 s); abort transaction with **fewest locks**.

### Recovery (in progress at paper writing) **[P]**

- Log records with **LSN** (node id + sequence); log managers merge streams; **WAL**; **ARIES**-style undo; group commit.

### Chained declustering **[P][L][R5]**

- **Primary** fragment `Ri` on disk `i mod M`; **backup** `ri` on disk `(i+1) mod M`—fragments “chained” around the ring.
- On node failure: reads use mix of primary/backup; **reassign** fraction of load to neighbors—**~1/(M−1)** increase per surviving node when cluster size M.
- **No data movement** on failover—update **in-memory** fragment map.
- vs **Interleaved declustering** (Teradata): backup split across cluster; **two** failures can lose availability; Teradata recommends **small** clusters (4–8). Chained can grow cluster without that availability hit.
- **Lecture:** Interleaved spreads failed node’s load on **0,2,3**; chained pushes load along the chain. **[L]**

---

## 12. Speedup, scaleup, superlinear effects

**Metrics [P]:**

- **Speedup:** More processors, **fixed** data size → response time should drop (ideal: linear).
- **Scaleup:** Data size grows **with** processor count → response time **constant** (ideal).

**Setup:** Wisconsin benchmark, **208-byte** tuples, **8 KB** pages, **2 MB** buffer pool; often **hash** on `Unique1`; results **stored in DB** (not returned to host). Tests on up to **30** disk nodes.

**Selection highlights:**

- Clustered index: time depends on **tuples retrieved**, not only table size (1% of 10M ≈ 10% of 1M).
- **Single-tuple** select on partitioning attribute → **one node** (~0.08 s on 1M tuples); B-tree height adds cost at 10M.
- **Superlinear speedup** on some scans: one disk holds **66%** of relation vs **2%** per disk when declustered over 30 nodes → **shorter seeks** (seek ∝ √ distance). **[P][L]**

**Lecture:** Multi-zone disks—outer zones faster; more disks → higher aggregate bandwidth; also shorter seeks with smaller fragments. **[L]**

---

## 13. Lecture-only topics

**Virtual nodes:** One 8-core machine → eight logical nodes (split CPU, memory, disk fraction). **[L]**

**Speedup vs scaleup examples:** 10% raise for all employees (speedup); keep query time as DB grows 10× by adding 10× nodes (scaleup). **[L]**

**Pipelined toy query:** `TMP2 ← σ_{dname=Toy}(Emp ⋈ Dept)` — processes for store, select, join; can **reorder** join vs select (push σ on Dept first). **[L]**

**Hash-join lecture:** Build = hash table on inner; probe = scan outer; **sequential** phases; **graceful degradation** when build does not fit RAM → partition into M buckets (same hash), spill to disk, join bucket-by-bucket. **[L]**

**SQL / relational algebra block (slides 51–101):** DDL, `Emp`/`Dept`, σ, π, joins, query trees—standard undergrad DB material; exam may use it to read Gamma plans. **[L]**

---

## 14. Gamma contributions (slide checklist)

From lecture slide 132 **[L]** (cross-check paper):

- Declustering strategies (incl. **HRPS**, **MAGIC** in next lecture)
- **Hybrid hash-join**
- **Chained declustering** (high availability)
- **Dataflow** execution paradigm
- **Split operator** / split table (modular routing)
- Nodes **with** and **without** disk
- **Speedup & scaleup** methodology
- **SQL hides parallelism**

---

## 15. Exam cheat sheet

**One sentence:** Gamma is a **shared-nothing**, **SQL**, **dataflow** parallel RDBMS with **declustered** tables, **hash-parallel** joins, and **minimal** scheduler control messages.

**Three ideas:** decluster · hash algorithms · dataflow.

**Processes:** QM → (scheduler if multi-site) → operators; **split table** routes tuples.

**Control:** ~**3 messages per operator per processor**; scheduler **not** on data path.

**Partitioning:** hash → equality **1** node, range **all**; range → **subset** for key predicates.

**Join default:** **Hybrid hash**; build/probe; buckets when memory full.

**Availability:** **Chained declustering**—backup on next disk in ring.

**Award / legacy:** ACM **2008**; Redshift/BigQuery lineage (lecture).

**DIRECT lesson:** don’t centralize control; don’t skip indexes for I/O-bound work.
