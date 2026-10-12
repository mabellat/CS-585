# HRPS — Lecture-style slides

**Ghandeharizadeh & DeWitt · VLDB 1990 · CSCI 585**  
Bullets follow [`HRPS.pdf`](HRPS.pdf). Depth is in [`HRPS_Study_Guide.md`](HRPS_Study_Guide.md).

---

## Slides 1–3 — Title and claim

- **Hybrid-Range Partitioning Strategy**, Ghandeharizadeh and DeWitt, CSCI 585
- Place data using the **hardware** and the **workload**
- Parallelism has overhead: do not let it dominate response time
- Still use enough parallelism to balance load
- Lecture names LevelDB, FoundationDB, Nova-LSM as later systems with similar declustering
- **Logical design** = tables and normal forms. **Physical design** = partitioning and indexes. HRPS is physical design

---

## Slides 4–13 — Disk, buffer, cache

- Shared-nothing sketch: nodes 0 … N−1, each with CPU, memory, disk
- Disk pages are staged into a **buffer pool**. A hit skips disk I/O
- Page = block. Example size **8 KB** (Gamma uses 8 KB)
- **Disk cache / read-ahead:** if the controller is idle it prefetches the next page. Helps sequential reads

---

## Slides 14–29 — Units, RID, heap scan

- bit → byte → KB → MB → GB → TB → PB → EB → ZB (1024 each step)
- “1 TB” drive shows up as ~**907 GB**. Vendors use 1000^4; the OS uses 1024^4. Gap on the slide: **92.67 GB**
- **Indexed heap:** pointer array in the page. **RID** = block number + slot. Moving a row inside the page does not change the RID
- **σ by heap scan:** open the file, read every page, emit rows that match. Fetch the page if it is not in the buffer pool

---

## Slides 30–39 — Indexes, and when parallelism hurts

- **Hash index** → exact match. **B+-tree** → range
- **mod:** `x mod N` is the remainder, in `0 … N−1`. `h(age) = age % B`
- Non-clustered hash/B+ store **(key, RID)** pairs. Clustered hash/B+ keep the **rows** in index order
- A query that touches **one page** should go to **one node**. Sending it to N nodes makes N−1 nodes search and find nothing
- Overhead of parallelism dominates when the query reads only a few pages

---

## Slides 40–48 — Hash vs range

- No partitioning attribute → random or round-robin. One attribute → hash or range. DBA picks the attribute
- **Hash:** `salary = 60K` → 1 node. `61K < sal < 61.1K` → **all** nodes, even if the B+-tree wants one page
- Many duplicates of 60K: exact match still hits **one** node
- **Range:** equality and small ranges hit a **subset**. A non-selective range can still be stuck on **one** node’s fat range and should have used several
- Why one node is wrong for a query that reads many pages:
  - **Service time μ** goes up (no other load)
  - **Response time τ** = μ + queue
  - **Throughput λ** falls. Closed system: **λ = 1/μ**
  - Hotspot

---

## Slides 49–57 — Imbalance, intra vs inter

- Random assignment is rarely uniform. Lecture picture: 3 requests, 3 nodes, **27** assignments, only **6** perfectly even
- **Range tail:** grades `0–19 … 80–100`. Predicates do not wrap from 80–100 back to 0–19
- `0 < grade < 45` and `45 < grade < 90` both include **40–59**. That node at **100%**, neighbors **66%**, ends **33%**
- **Intra-query** = nodes inside one user’s query. **Inter-query** = nodes used by concurrent independent queries
- **Ideal execution (space A):** range wins on cheap queries; hash wins on expensive queries; both win on cheap **equality**

---

## Slides 58–66 — HRPS and the three pictures

- Few pages: service time **rises** with more nodes
- Many pages: service time **falls, then bottoms**. The bottom is **M**
- `RT(M) = (CPU_avg + Disk_avg + Net_avg) / M + M × CP`
- `M = sqrt(work / CP)`
- Toy: 10,000 tuples, 500 tuples/query, M = 5 → **FC = 100**, **100 fragments**
- **M = N:** HRPS query uses all nodes (5 or 6 fragments). Range uses 1–2 fat ranges
- **M > N:** HRPS still uses every node. Range usually uses one
- **M < N:** HRPS uses ~M nodes per query but fragments cover all N. Ask: better response time? better throughput? Both favor HRPS
- Directory: linear-search RT adds `M × Cardinality × CS / TuplesPerQ`. Binary-search M is the quadratic formula on the slide

---

## Slides 67–76 — Execution and Gamma numbers

- **1 node:** BM → CPU → disk, no query manager
- **Many nodes:** BM and QM; start, query-done, commit, on private ports
- Algorithm on the HRPS slide: M, TuplesPerQ, **FC = TuplesPerQ/M**, F = Cardinality/FC, cut F ranges
- HRPS should cover **both** the range region and the hash region of space A
- **10-tuple** clustered range: M = **0.057**, FC = **175**, F = **5,700**, N = 24
  - Throughput linear-ish until MPL ~12, then sublinear (collisions)
  - Hash is far below (all 24 nodes)
- **Heavy clustered range** (paper: 10%, 100,000 tuples): M = **90**, **895** fragments. Range uses **≤ 3** nodes. Hash uses **24**
  - SCSI cache shows up here and not on the 10-tuple query (one page)

---

## Slides 76–77 — Other features and what is still missing

- **Small tables:** fewer fragments than nodes → partial declustering
- **Duplicates:** several fragments for one popular key, on different nodes. Hash and plain range do not do this
- Still open: **heterogeneous** hardware (one derivative does not fit), and predicates on **different attributes**

---

## Slides 78–93 — MAGIC and BERD

- **MAGIC**, Ghandeharizadeh, DeWitt, Qureshi, **SIGMOD 1992**
- `Emp(name, age, salary)`, 9 nodes, 50/50
  - A: age range, ~5 tuples, ideally 1 node
  - B: salary range, ~10 tuples, ideally 1 node
- Range on age: average nodes = **5.5**. Ideal = **1**
- **MAGIC:** grid. Axis = attribute. Cell = fragment (a node id)
  - Low correlation: average **3**
  - High correlation: average **1**
- **BERD:** range-partition Emp on **salary**. Auxiliary `(age, node)` range-partitioned on age
  - Low correlation: average **5.5**
  - High correlation: average **1** (can you skip the auxiliary lookup?)
- Low/low throughput curves: **MAGIC > BERD > range**, both correlations
- MAGIC: workload-shaped grid, less parallelism overhead, partial declustering on large machines

---

## Slides 94–97 — How they score a technique

- **Closed:** fixed clients, next request after the reply, optional **think time**. Better = higher throughput vs MPL. Slide: hybrid-range above range
- **Open:** arrival rate **λ**. Better = higher λ before a queue. Slide: hash’s response time blows up sooner than hybrid-range

---

### Suggested lecture flow (45 min)

| Block | Minutes | Slides | Say this |
|-------|---------|--------|----------|
| Claim | 3 | 1–3 | Physical design. Overhead vs balance |
| Storage in one breath | 5 | 6–13, 26, 30, 39 | Page, buffer, hash vs B+, one page → one node |
| Hash and range fail | 8 | 41–55 | Equality vs range; grade tail; μ, τ, λ |
| Formula | 8 | 56–66 | Intra vs inter. RT(M). Toy. Three cases |
| Gamma | 12 | 67–75 | 5,700 vs 895. 8×, 30×, 16%. Why the curve bends |
| Limits + MAGIC | 6 | 76–93 | One attribute. Grid. 5.5 vs 3 vs 1 |
| Models | 3 | 94–97 | Closed = the paper’s plots |

---

### Where to go deeper

- Narrative and numbers: **`HRPS_Study_Guide.md`**
- Teach from zero: **`HRPS_Full_Primer.md`**
- Drill: **`HRPS_Flashcards.md`**
- Exam answers: **`HRPS_Exam_QA.md`**
- Figures: **`Hybrid-Range.PDF`** (Figure 1 placement; Figures 4–9 throughput)
