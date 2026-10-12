# HRPS: Hybrid-Range Partitioning Strategy

**Full citation:** Shahram Ghandeharizadeh and David J. DeWitt. *Hybrid-Range Partitioning Strategy: A New Declustering Strategy for Multiprocessor Database Machines.* VLDB 1990, Brisbane. University of Wisconsin–Madison. **[P]**

**Lecture:** CSCI 585 deck [`HRPS.pdf`](HRPS.pdf), same authors. The deck reviews storage and indexes, then HRPS, then the follow-on **MAGIC** (SIGMOD 1992) and closed vs open performance models. **[L]**

This guide is for **CS 585** exam prep. Each section follows from the last. Claims from the paper are tagged **[P]**; lecture-only claims are tagged **[L]**.

**Also in this folder:**

- **[HRPS_Full_Primer.md](HRPS_Full_Primer.md)** — start here if you want every concept from scratch
- [HRPS_Flashcards.md](HRPS_Flashcards.md) — Q/A tables by deck
- [HRPS_Lecture_Slides.md](HRPS_Lecture_Slides.md) — bullets mapped to the 97-slide deck
- [HRPS_Exam_QA.md](HRPS_Exam_QA.md) — example exam/oral questions with model answers
- [HRPS_References.md](HRPS_References.md) — source index

---

## Table of contents

1. [Why this paper exists](#1-why-this-paper-exists)
2. [Glossary](#2-glossary)
3. [Shared-nothing and Gamma’s three strategies](#3-shared-nothing-and-gammas-three-strategies)
4. [Where hash wins and where range wins](#4-where-hash-wins-and-where-range-wins)
5. [What HRPS does](#5-what-hrps-does)
6. [The response-time formula and M](#6-the-response-time-formula-and-m)
7. [How fragments are built and placed](#7-how-fragments-are-built-and-placed)
8. [Three cases: M = N, M > N, M < N](#8-three-cases-m--n-m--n-m--n)
9. [Directory search cost](#9-directory-search-cost)
10. [Workload and Gamma setup](#10-workload-and-gamma-setup)
11. [Experimental results](#11-experimental-results)
12. [Small relations and skewed keys](#12-small-relations-and-skewed-keys)
13. [Limitations and related work](#13-limitations-and-related-work)
14. [Lecture extras: indexes, imbalance, MAGIC, models](#14-lecture-extras-indexes-imbalance-magic-models)
15. [Exam cheat sheet](#15-exam-cheat-sheet)

---

## 1. Why this paper exists

In a **shared-nothing** database machine, each node has its own CPU, memory, and disk. A relational operator runs **on the nodes that store the tuples it needs**. **[P][R1]**

Spreading a relation across more nodes (**declustering**) usually shortens a heavy query: each node does less work. For a **light** query, startup, communication, and termination overhead can be a large fraction of the runtime. Using more nodes then **increases** response time and **lowers** system throughput. A heavy query left on one node creates a **hot spot** when many users run at once. **[P]**

The declustering strategy is therefore a **physical design** choice. Logical design decides tables and normal forms. Physical design decides, for each table, the partitioning strategy and the indexes. **HRPS is a physical design technique for partitioning tables.** **[L]**

Gamma already offered three strategies: **round-robin**, **hash**, and **range**. None of them chooses the degree of parallelism from the **workload** and the **hardware**. HRPS does. **[P]**

The lecture’s one-line contribution: place data so that parallelism’s overhead does not dominate response time, and so that the degree of parallelism still balances load across nodes. LevelDB, FoundationDB, and Nova-LSM later use similar declustering; the lecture says hybrid-range predates them (the paper is VLDB 1990). **[L]**

---

## 2. Glossary

| Term | Meaning |
|------|---------|
| **Declustering / horizontal partitioning / sharding** | Split a relation’s tuples across nodes. Same idea, three names in the lecture. **[L]** |
| **Partitioning attribute** | The column the hash or range function looks at. The DBA must choose it. **[L][P]** |
| **N** | Number of processors (nodes) in the configuration. |
| **M** | Desired number of processors that should **participate in one average query**. Not “how many nodes the table lives on.” **[P]** |
| **FC** | Fragment cardinality. Tuples in one fragment. **FC = TuplesPerQ / M**. **[P]** |
| **Fragment** | A slice of the relation: about FC tuples and a **distinct range** of the partitioning attribute. **[P]** |
| **Range table** | One-dimensional directory: fragment range → processor. **[P]** |
| **CP (CostOfPart)** | Overhead of using **one more** processor. In Gamma, **26 ms**. Linear in the number of processors because Gamma starts and stops operators sequentially. **[P]** |
| **CS (CostOfSearch)** | Cost of examining **one** range-table entry. Measured at **0.243 ms**. **[P]** |
| **Intra-query parallelism** | Nodes used to run **one** user’s query. **[L]** |
| **Inter-query parallelism** | Nodes used because **different users’** queries run at once. **[L]** |
| **Service time μ** | Time to process a query with **no other load**. **[L]** |
| **Response time τ** | Service time **plus queuing delay** from concurrent queries. **[L]** |
| **Throughput λ** | Rate of completed requests when the system is busy. In a closed system, **λ = 1/μ**. Independent of queuing delay. **[L]** |
| **MPL** | Multiprogramming level: how many queries are in the system at once. The x-axis of the Gamma plots. **[P]** |

---

## 3. Shared-nothing and Gamma’s three strategies

**Shared-nothing** (Stonebraker): no shared memory and no shared disk. Parallelism comes from partitioning the data. **[P][R1]**

Gamma’s catalog stores, for each relation, which strategy was used. For hash and range it also stores the partitioning attribute. For range it stores the **range table**. **[P]**

| Strategy | How tuples are placed | Exact match on the partitioning attribute | Range predicate on that attribute |
|----------|----------------------|-------------------------------------------|-----------------------------------|
| **Round-robin** | Next tuple goes to the next disk. No key. | All nodes | All nodes. Degree of parallelism = number of disks. |
| **Hash** | Hash the key to pick a node. | **One node** | **All nodes** (hash destroys range locality) |
| **Range** | DBA assigns a key range to each processor. One coarse range per node. | **One node** (or a small subset) | **Only the nodes whose ranges overlap the predicate** — often 1–2 nodes even when the query is heavy |

**[P][L]**

Round-robin and hash cannot aim a **small range** at a few nodes. Range can, but a **large** range still runs on too few nodes, so response time is worse than hash or round-robin. **[P]**

---

## 4. Where hash wins and where range wins

The paper draws “space A”: every possible query. **[P]** The lecture repeats this as “Ideal Execution.” **[L]**

- **Range region:** queries with **tiny** CPU and I/O. Range keeps them on one processor and avoids multi-processor overhead.
- **Hash region:** queries with **large** CPU and I/O. Hash sends them to every processor and intra-query parallelism pays off.
- **Shaded overlap:** **equality** predicates on the partitioning attribute with tiny resource use. **Both** hash and range send these to one node.

The two regions are otherwise disjoint. Hash is the wrong tool for a tiny range query. Range is the wrong tool for a huge range query. HRPS has to cover **both** regions, and it has to win on a **mix** whose queries want opposite placements. **[P]**

### Hash failure modes (lecture)

A selective range such as `sal > 61K AND sal < 61.1K` still goes to **all N nodes**, even if a B+-tree would touch one page. N−1 nodes search an index and find nothing. Overhead dominates. **[L]**

A **non-selective exact match** (`salary = 60K` when thousands of employees earn 60K) still goes to **one** node. That node’s service time explodes and it becomes a hotspot, so throughput falls. **[L]**

### Range failure mode (lecture): the tail

Range-partition grades into `0–19`, `20–39`, `40–59`, `60–79`, `80–100`. A predicate that spans three buckets does **not** wrap from `80–100` back to `0–19`. **[L]**

- `0 < grade < 45` hits nodes 1–3.
- `45 < grade < 90` hits nodes 3–5.

Node 3 (`40–59`) is in **both**. If access to those queries is even, node 3 is at **100%**, nodes 2 and 4 at **66%**, nodes 1 and 5 at **33%**. Node 3 is the bottleneck and throughput drops. **[L]**

Sending a query that reads **many** disk pages to one node instead of N nodes does two things: service time goes up, and a hotspot forms. **[L]**

---

## 5. What HRPS does

HRPS cuts the relation into **many small logical fragments**. Each fragment holds about **FC** tuples and a **unique range** of the partitioning attribute. The number of fragments is **independent of N**. **[P]**

The optimizer can then:

- run a **cheap** range query on **one** processor (it overlaps one fragment, maybe two), and
- run an **expensive** query on **many** processors (it overlaps about M fragments, which were placed on different processors).

Four inputs decide FC: **[P]**

1. Resource needs of the queries (CPU, disk, network).
2. Processing capability of the system (those times already reflect the hardware).
3. Overhead of each extra processor (**CP**).
4. Cost of searching the range table (**CS**). The last one lets HRPS **degenerate toward range** when the query is so small that directory search would dominate — the lecture’s example is a **Debit-Credit** transaction. **[P]**

**Assumption:** selections use a **range or equality** predicate and an index. HRPS does not target full file scans. For sequential scans, range declustering is the best of the earlier strategies. The selection matters because later operators in a dataflow plan cannot be more parallel than the selection that feeds them. **[P]**

---

## 6. The response-time formula and M

Build one **average query** Q_avg from the mix. For each query Qi with frequency FreqQi: **[P]**

- CPU_avg = Σ (CPU_i × Freq_i)
- Disk_avg, Net_avg, TuplesPerQ_avg the same way

**Assumption used to add the three resources:** one query does not overlap CPU, disk, and network on a single node. Gamma does use read-ahead, so this is false for one user. In a busy multiuser system the chance that the *same* query overlaps its own resources is low, because other queries occupy the other devices. **[P]**

Single-node time:

```text
ExecutionTime = CPU_avg + Disk_avg + Net_avg
```

With M processors, work splits by M and each extra processor costs CP:

```text
RT(M) = (CPU_avg + Disk_avg + Net_avg) / M  +  M × CP
```

Set the derivative to zero. The M that minimizes response time is: **[P][L]**

```text
M = sqrt( (CPU_avg + Disk_avg + Net_avg) / CP )
```

**Reinterpretation that makes throughput work.** If M were “decluster the whole relation onto M nodes,” a tiny workload (M = 1) would park every tuple on one node. Concurrent users would queue there. Range, with one coarse range per node, would spray those concurrent tiny queries across all N nodes. So M is **not** the number of nodes that store the relation. M is the number of nodes that should **run one query**. Each fragment holds **[P]**

```text
FC = TuplesPerQ_avg / M
```

Fragments are then **round-robin**’d across **all N** nodes. One query touches about M fragments (hence about M nodes). Many concurrent queries can still use the whole machine. That is intra-query parallelism set to M, and inter-query parallelism set to N. **[P][L]**

Lecture pictures: **[L]**

- **Few disk pages:** service time **rises** as you add nodes. Overhead wins. Do not parallelize.
- **Many disk pages:** service time **falls**, then flattens, then creeps up. The bottom of the curve is **M**.

---

## 7. How fragments are built and placed

Steps: **[P][L]**

1. Let M be the desired intra-query parallelism of the average query.
2. Let TuplesPerQ be the average number of tuples that query touches.
3. **FC = TuplesPerQ / M**.
4. Set each fragment’s cardinality to FC.
5. **F = Cardinality(Rel) / FC** fragments.
6. **Sort** the relation on the partitioning attribute.
7. Cut F ranges of about FC records each. Ranges do not overlap.
8. Assign fragments to processors **round-robin**, so **M adjacent fragments land on different processors** whenever N ≥ M.
9. On each processor, all fragments of that relation sit in the **same physical file**.
10. Record the mapping in the **range table**.

A query’s predicate overlaps **at least M and at most M+1** fragments, so it runs on M or M+1 processors. **[P]**

The running example in the paper and on the slides: **[P][L]**

- 10,000 tuples, keys 0 … 9,999, unique.
- Average query reads **500** tuples. **M = 5**.
- **FC = 500/5 = 100**. **F = 10,000/100 = 100** fragments.

---

## 8. Three cases: M = N, M > N, M < N

Same example: 100 fragments of 100 tuples, query reads 500 tuples so it overlaps **5 or 6** fragments. **[P][L]**

| Case | Picture | HRPS | Range | Hash |
|------|---------|------|-------|------|
| **M = N** (N = 5) | 5 columns of small ranges vs 5 fat ranges | Query hits 5 or 6 fragments → **all 5 nodes** | Query usually sits inside **one** fat range, sometimes two → **1 or 2 nodes** | Range query → **all N** |
| **M > N** (N = 2) | Two columns; each node holds many small ranges (0–99 and 200–299 on node 1, and so on) | M adjacent fragments were forced onto different nodes, so the query still uses **all N** | Relation split into **N = 2** fat ranges → usually **1 node** | **All N** |
| **M < N** (N = 10) | 100 small ranges spread across 10 nodes; range has 10 fat cylinders | Query still overlaps only **5 or 6** fragments → **about M nodes**, but fragments occupy **every** node so concurrent queries spread out | **1 or 2 nodes** — too little parallelism | **All 10** — more overhead than the query needs |

**[P][L]**

For **M < N** the lecture asks which wins. **Response time:** HRPS, because each query uses about M nodes, not 1 and not N. **Throughput:** HRPS as well, in the multiuser case, because the fragments cover all N nodes (inter-query parallelism) while each query avoids hash’s extra control messages. Hash burns startup and termination on nodes that do not need to run. Range under-parallelizes the heavy part of the work. **[P][L]**

---

## 9. Directory search cost

Worst case, FC = 1 and the directory has one entry per tuple (reasonable only for very expensive work, such as image processing). If the query is cheap, a linear scan of that directory **is** the query. So search cost belongs inside the model. **[P]**

Number of directory entries ≈ **M × Cardinality(Rel) / TuplesPerQ**.

With a **linear** search, CS per entry: **[P][L]**

```text
RT(M) = (CPU + Disk + Net) / M  +  M × CP
        +  M × Cardinality(Rel) × CS / TuplesPerQ

M = sqrt( (CPU + Disk + Net) / (CP + Cardinality(Rel) × CS / TuplesPerQ) )
```

That second formula is the paper’s **equation (4)**. The Gamma experiments use it. **[P]**

The optimizer actually **binary-searches** the range table. The lecture writes the corresponding M (the positive root of the quadratic) as: **[L]**

```text
M = ( -CS/ln(2) + sqrt( (CS/ln(2))^2 + 4 × CP × (CPU+Disk+Net) ) ) / (2 × CP)
```

**Worked check for the tiny query (this one does match equation (4)).** Single-node time **0.08 s = 80 ms**, CP = **26 ms**, CS = **0.243 ms**, 1,000,000 tuples, TuplesPerQ = **10**: **[P]**

```text
denom = 26 + (1,000,000 × 0.243) / 10 = 26 + 24,300 = 24,326
M = sqrt(80 / 24,326) ≈ 0.057
```

M rounds to **one processor**. FC = 10 / 0.057 ≈ **175**. Fragments = 1,000,000 / 175 ≈ **5,700**. The directory term dominates the denominator, which is why M collapses below 1. **[P][L]**

---

## 10. Workload and Gamma setup

**Why multiuser throughput, not just single-user response time.** A 1% range selection may have almost the **same** response time under hash and under HRPS, because extra control messages are small next to the real work. Hash still **occupies every node**. Throughput under concurrency is what exposes that waste. **[P]**

**Database.** 10 relations. Each is a Wisconsin Benchmark relation with **1,000,000 tuples × 208 bytes**. Access to relations is uniform. **[P][R4]**

**Three query shapes, all selections through a clustered index on the partitioning attribute.** **[P]**

| Query | Tuples | Single-node, single-user time on Gamma | What it represents |
|-------|--------|----------------------------------------|--------------------|
| **0.001%** range | **10** | **0.08 s** | Tiny. Range should win. Equality with tiny work is a special case of this. |
| **10%** range | **100,000** | **54.33 s** | Heavy. Hash should win. |
| **Mix** | half and half, then a sweep of the mix | — | Conflicting placement. HRPS should win. |

The lecture titles the heavy query “Fetch 10K Records” but prints **M = 90** and **895 fragments**, which are the paper’s **10% / 100,000-tuple** numbers. Use the paper’s figures. **[P][L]**

**Gamma.** 32-node Intel **iPSC/2** hypercube. Each node: Intel 80386, **8 MB** RAM, **330 MB** Maxtor disk. Experiments use **24** nodes for Gamma and **8** nodes to play users (the link from the hypercube to the Unix host is too slow to drive the load from outside). **8 KB** pages, **2 MB** buffer pool, **LRU**, elevator disk scheduler. Storage is the Wisconsin Storage System (WiSS): B-trees, files, page and file locks. **[P][R2]**

Each disk’s SCSI controller has a **45 KB** read cache (read-ahead). In a **single-user** sequential scan it nearly **doubles** speed. In a **multiuser** mix, the next I/O is rarely sequential, and a seek **invalidates** the cache. You see this cache in the 10% curves and not in the 0.001% curves. **[P]** The lecture’s disk-cache slide is the same idea: if the controller is idle it prefetches the next page. **[L]**

**Control messages.** A single-node query goes **straight** to that node. A multi-node query goes to a **Query Manager (QM)**: start, “query done” from each node, then commit (or abort). That is why a tiny query hurts so much when hash fans it out. Benchmark (BM) and QM were placed on the same processor; one pair per simulated user; CPU on the 8 driver nodes stayed under 100%. **[P][L]**

**Measured constants.** CostOfPart **CP = 26 ms**. CostOfSearch **CS = 0.243 ms per range-table entry**. **[P]**

An **8 KB** page holds about **36** of the 208-byte tuples. The 10-tuple query touches **one page most of the time, two pages sometimes**. The 100,000-tuple query touches a long sequential run. **[P]**

---

## 11. Experimental results

Predicates are on the **partitioning attribute**. Hash and range spread tuples uniformly over all 24 nodes. HRPS is approximately uniform. **[P]**

### 11.1 Tiny query: 0.001% (Figure 4)

M = **0.057** → **1** processor. **5,700** fragments, **~175** tuples each, round-robin over the nodes. **[P][L]**

- Hash must visit **all 24** nodes. At MPL = 1, range and HRPS throughput is about **8×** hash.
- Hash’s CPUs hit **100%** around MPL = **20**, and its throughput flattens.
- Range and HRPS are almost the same at every MPL (gap **< 2.5%**). Searching 5,700 entries instead of 24 is noise next to 0.08 s of real work.
- They do **not** scale linearly from MPL 1 to 24. Past about MPL **12** the curve bends. Random predicates collide: with 24 nodes and 12 queries, two queries can land on the same node. The lecture asks this explicitly. **[P][L]**
- At MPL = **160**, range and HRPS throughput is about **30×** hash.
- SCSI cache does **not** show up: one page, and the index probes are random. **[P]**

### 11.2 Heavy query: 10% (Figure 5)

Equation (4) gives **M = 90**. HRPS builds **895** fragments. Only 24 nodes exist, so each node does about **90/24 ≈ 3.7×** the work it would do at the optimum. The query still goes to **all 24** nodes. **[P][L]**

Range builds **24** fragments. A 10% predicate overlaps **at most 3**, so range uses **≤ 3** nodes. Hash uses **all 24**. **[P][L]**

- At MPL = 1, hash and HRPS throughput is **16%** above range. This is hash’s best case and range’s worst case. HRPS matches hash because it also uses full intra-query parallelism.
- Range barely improves from MPL 1 to 2: two big queries on one node destroy the SCSI cache, and each query reads a huge sequential run.
- Hash and HRPS jump from MPL 1 to 2, then **drop at MPL 3** (cache stops helping), then climb again from processor sharing and the elevator scheduler.
- The lecture’s question “how many nodes does hash use?” Answer: **all N = 24**. **[L]**

### 11.3 Mixed workload (Figures 6–9)

Equal mix of the two queries. They want opposite plans: the tiny one wants **1** node, the heavy one wants **all** nodes. Neither hash nor range can do both. **[P]**

HRPS builds **1,688** fragments (equations 5 and 6) and round-robins them. The optimizer sends the 0.001% query to **one** node and the 10% query to **all** nodes. Hash sends **both** to all nodes. Range sends the tiny query to one node and the heavy query to **at most 3**. **[P]**

- At MPL = 1, HRPS is **11%** above hash and **46%** above range.
- The 10% query dominates the average, because it is so much slower. Past MPL **4**, hash and HRPS **converge**. The win on the tiny query is a small slice of the average.
- Sweep the mix at **MPL = 50** (Figure 7). Once the heavy query is **more than about 10%** of the mix, hash and HRPS are nearly tied, and HRPS is slightly ahead because it still localizes the tiny queries.
- Improvement over range **grows** as the heavy query becomes more common, then **levels off near 40%** heavy queries because the disk is saturated (Figure 8).
- Zoomed in (Figure 9): when the heavy query is **under 5%** of the mix, **range beats hash**, because more than 95% of queries are tiny and range keeps them on one node. When the heavy query is **5–7%** of the mix, **HRPS beats both by close to 30%**. That band is where neither pure strategy is appropriate. **[P]**

**How to generalize on an exam.** More tiny queries → range and HRPS converge and both beat hash. More heavy queries → hash and HRPS converge and both beat range. In the middle, HRPS beats both. **[P]**

---

## 12. Small relations and skewed keys

### Small relations

On a machine with hundreds or thousands of nodes, a small relation should occupy a **subset** of the nodes. HRPS does this automatically: fragment count depends on the workload and the hardware, **not on N**. If F < N, the relation lands on F nodes. **[P][L]**

Many small relations at once is **bin packing** (NP-complete). One heuristic: put the next fragment on the processor with the **most free disk** that does **not** already hold a fragment of that relation. Also consider each fragment’s **temperature** (how hot it is). Copeland’s Bubba heuristics are the ones the paper points at. **[P][R3]**

### Non-uniform keys

FC is a **tuple count**, not a width in key space. Sort, then cut every FC tuples. **[P][L]**

Example: 100,000 tuples, FC = 1,000, and **4,000** tuples have partitioning value **2**. After the sort, value 2 spans **4 or 5** fragments, placed on **different** processors. An equality query for 2 runs on **4** nodes. Hash and range would have put every “2” on **one** node. **[P]**

The lecture states the same idea as “many duplicates of the partitioning attribute”: HRPS builds several ranges for that duplicate value and spreads them. Hash and plain range do not. **[L]**

---

## 13. Limitations and related work

**Open issues in the paper.** **[P]**

1. HRPS assumes the query’s work **can** be parallelized. Some queries cannot.
2. The strategy is **one-dimensional**. A predicate on any attribute **other than** the partitioning attribute is sent to **every** node that holds a fragment.

**Related work, and how HRPS differs.** **[P]**

| Work | What it did | Why it is not HRPS |
|------|-------------|--------------------|
| **XPRS** (shared-memory, RAID) | Fragments a relation into files, each extent is Wi disks by Si tracks | HRPS chooses **how many fragments** (intra-query parallelism). Wi and Si are a different problem. |
| **FX distribution** (Kim & Pramanik) | Optimal placement for **partial-match** queries | No useful support for **range** queries. |
| **Bubba / Copeland** | **Heat** and **temperature**; balance load; decide disk vs memory-resident. Heuristic, NP-complete | Maximizes throughput, does **not** minimize response time, and does not compare declustering strategies. **Orthogonal** to HRPS: cut fragments with HRPS, then place them with Copeland’s heuristics. The same heuristics can reorganize after updates and workload changes. |

Gamma, the Wisconsin benchmark, and shared-nothing are the platform, not competitors. **[P][R1][R2][R4]**

---

## 14. Lecture extras: indexes, imbalance, MAGIC, models

The first half of the deck is storage review. Exam questions can still use it, because the paper’s results depend on a **clustered index**. **[L]**

### Storage, in one pass

- A **disk page** is a **disk block**. Fixed size; the lecture’s example and Gamma’s page are **8 KB**. Pages move from disk into a **buffer pool**. A hit avoids a disk I/O. **[L][P]**
- Units: 1024 bytes = 1 KB, then MB, GB, TB, PB, EB, ZB. **[L]**
- A “1 TB” drive can show up as about **907 GB**. Manufacturers use **1000**-based TB; operating systems use **1024**-based. The gap on the slide is **92.67 GB**. **[L]**
- **Indexed heap:** each page has an array of pointers to records. **RID / TID** = block number + slot. Moving a record inside the page does not change the RID. **[L]**
- **Heap scan** of σ evaluates the predicate on every record of every page. **[L]**
- **Hash index:** exact match. **B+-tree:** range. A query that touches one page should go to **one** node. **[L]**
- **h(x) = x mod B** is the remainder. It maps x into **0 … B−1**. **[L]**
- **Clustered** index: heap order matches index order, so a range is sequential I/O. **Non-clustered:** the index leaf points at RIDs scattered across the heap. The paper’s selections use a **clustered** index: random I/O to walk the B-tree to the endpoints, then **sequential** I/O for the tuples in between. **[P][L]**

### MAGIC and BERD (slides 77–93)

The lecture asks what HRPS still cannot do: **[L]**

- **Heterogeneous** hardware → more than one RT(M) curve. One derivative does not fit every node.
- Predicates on **different attributes** → you need **more than one** partitioning attribute. That limitation is also the paper’s conclusion. **[P]**

**MAGIC** (Ghandeharizadeh, DeWitt, Qureshi, **SIGMOD 1992**) is the follow-on. **[L][R5]**

Running example: `Emp(name, age, salary)`, **9 nodes**, two queries at 50% each. **[L]**

- **Query A:** age range, about **5** tuples, non-clustered B+-tree on age. Ideally **1** node.
- **Query B:** salary range, about **10** tuples, clustered B+-tree on salary. Ideally **1** node.

**Range on age:** A uses 1 node, B uses all **9**. Average nodes per query = 0.5×1 + 0.5×9 = **5.5**. Ideal average is **1**. **[L]**

**MAGIC** builds a **grid directory**. Each axis is a partitioning attribute. Each **cell** is a fragment, labeled with a node id. **[L]**

| Correlation of age and salary | MAGIC avg nodes | Range avg nodes | Ideal |
|-------------------------------|-----------------|-----------------|-------|
| **Low** | **3** | **5.5** | 1 |
| **High** | **1** | **5.5** | 1 |

High correlation lets one cell cover both predicates, so MAGIC matches the ideal. **[L]**

**BERD**, as the lecture draws it: range-partition Emp on **salary** (the primary attribute). Build an **auxiliary relation** of `(age, node)` and range-partition **that** on age. A query on age looks up the auxiliary relation, then reads Emp on the listed nodes. **[L]**

| Correlation | BERD avg nodes | Range avg nodes |
|-------------|----------------|-----------------|
| **Low** | **5.5** (same as range) | 5.5 |
| **High** | **1** | 5.5 |

The lecture asks whether the auxiliary lookup can be skipped. When age and salary are highly correlated, the salary partition already localizes the age query, so the extra lookup buys less. **[L]**

Throughput for a low/low query mix: **MAGIC above BERD above range** at both low and high correlation. The gap is larger when correlation is high (the slide’s curves reach much higher queries/second). **[L]**

**MAGIC’s listed advantages:** beats BERD and range; the grid is shaped from the **workload** and from how often each attribute is used; it keeps parallelism overhead down; it can **partially** decluster a relation on a large machine. **[L]**

### Closed vs open models (slides 94–97)

These match the way the paper plots results. **[L]**

- **Closed:** a fixed set of clients. A client issues the next request only after the previous one returns, sometimes after a **think time**. A better technique is the one with **higher throughput** as the client count (MPL) grows. The slide draws hybrid-range above range.
- **Open:** arrivals at rate **λ**, whether or not earlier requests finished. A better technique supports a **higher arrival rate before a queue forms**. The slide draws response time vs λ, with hash’s knee to the **left** of hybrid-range’s knee.

The Gamma plots (throughput vs MPL) are the **closed** experiment. **[P][L]**

---

## 15. Exam cheat sheet

**One sentence:** HRPS declusters a relation into many small **ranges** whose size comes from the workload, so a light query runs on one node and a heavy query runs on about **M** nodes, while the fragments still cover the whole machine.

**Three strategies:** round-robin (everywhere), hash (equality → 1 node, range → all nodes), range (predicate → the overlapping nodes, often too few).

**Two letters:** **M** = nodes per query. **N** = nodes in the machine. **FC = TuplesPerQ / M**.

**One formula:** RT(M) = (CPU + Disk + Net) / M + M × CP, so M = sqrt(work / CP). Equation (4) puts directory search in the denominator.

**Three numbers from Gamma:** **24** worker nodes, **CP = 26 ms**, **CS = 0.243 ms/entry**. Tiny query: **M ≈ 0.057**, **5,700** fragments, **~8×** hash at MPL 1 and **~30×** at MPL 160. Heavy query: **M = 90**, **895** fragments, hash and HRPS **16%** above range at MPL 1. Mixed, MPL 1: HRPS **+11%** vs hash, **+46%** vs range. Sweet spot: heavy query is **5–7%** of the mix → HRPS beats both by **~30%**.

**Two extras:** small relations and duplicate-heavy keys. **One limit:** one partitioning attribute.

**Follow-on name:** **MAGIC**, a grid on several attributes (SIGMOD 1992).
