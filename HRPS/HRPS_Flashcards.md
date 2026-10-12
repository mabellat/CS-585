# HRPS — Flashcards

**Paper:** Ghandeharizadeh & DeWitt, VLDB 1990  
**Companion:** `HRPS_Study_Guide.md` · `HRPS_Exam_QA.md`

Cover the answer, say it, then check. Shuffle within a deck.

---

## Deck A — Big picture

| # | Question | Answer |
|---|----------|--------|
| A1 | What does **HRPS** stand for? | **Hybrid-Range Partitioning Strategy**. |
| A2 | Authors, venue, year? | **Ghandeharizadeh** and **DeWitt**, **VLDB 1990**, Wisconsin. |
| A3 | What kind of design decision is it? | **Physical** design: how to partition a table. Not logical (normal forms). |
| A4 | What machine model? | **Shared-nothing**: operator runs on the nodes that store the rows. |
| A5 | Research vehicle? | **Gamma** on an Intel **iPSC/2** hypercube. |
| A6 | Why not always use more nodes? | Startup, communication, and termination can dominate a **small** query and **hurt throughput**. |
| A7 | Why not always use one node? | Heavy queries get a long service time and form a **hot spot**. |
| A8 | What does HRPS compromise between? | Range’s **local** execution and hash/round-robin’s **load balance / intra-query parallelism**. |
| A9 | One-sentence goal of M? | Use enough nodes to minimize response time **without** letting overhead dominate. |
| A10 | Lecture’s modern cousins? | **LevelDB, FoundationDB, Nova-LSM** use similar declustering; the paper is 1990. |

---

## Deck B — The three old strategies

| # | Question | Answer |
|---|----------|--------|
| B1 | Gamma’s three built-in strategies? | **Round-robin**, **hash**, **range**. |
| B2 | Round-robin and a range predicate? | Query goes to **all** disks. No key to prune. |
| B3 | Hash + exact match on the partitioning attribute? | **One** node. |
| B4 | Hash + range predicate? | **All** nodes. Hash destroys locality. |
| B5 | Range + exact match or small range? | **One or two** nodes (the overlapping ranges). |
| B6 | Range + a heavy predicate? | Often still **1–2** nodes, so too little parallelism. |
| B7 | Hash + many duplicates of one key? | Still **one** node → hotspot, high service time, low throughput. |
| B8 | What is the range “tail”? | Queries do **not wrap**. The middle partition sits in several predicates and becomes the bottleneck. |
| B9 | Grade example: who is the bottleneck? | The **40–59** node, used by both `0–45` and `45–90`. Utilization **100%** vs **66%** and **33%**. |
| B10 | Who must pick the partitioning attribute? | The **administrator**. |

---

## Deck C — M, FC, placement

| # | Question | Answer |
|---|----------|--------|
| C1 | What is **N**? | Nodes in the configuration. |
| C2 | What is **M**? | Nodes that should **run one average query**. |
| C3 | What is M **not**? | Not “store the whole relation on M nodes.” |
| C4 | RT(M)? | **(CPU + Disk + Net) / M + M × CP**. |
| C5 | M that minimizes RT? | **sqrt(work / CP)**. |
| C6 | Why a square root? | Set **dRT/dM = 0**. Overhead linear, work divided by M. |
| C7 | What is **CP**? | Overhead of **one extra** processor. Gamma: **26 ms**. Linear because startup is sequential. |
| C8 | What is **FC**? | Tuples per fragment. **FC = TuplesPerQ / M**. |
| C9 | How many fragments? | **Cardinality(Rel) / FC**. |
| C10 | Build steps? | **Sort** on the key, cut ~FC tuples per range, **round-robin** so M neighbors sit on different nodes. |
| C11 | Where do fragments live on a node? | Same **physical file**. Mapping is the **range table**. |
| C12 | How many fragments does one query overlap? | **M or M+1**. |
| C13 | Toy numbers? | 10,000 tuples, TuplesPerQ = 500, M = 5 → **FC = 100**, **100 fragments**. |

---

## Deck D — Three cases

| # | Question | Answer |
|---|----------|--------|
| D1 | **M = N** (N = 5): HRPS vs range? | HRPS uses **all 5**. Range uses **1 or 2**. |
| D2 | **M > N** (N = 2): does HRPS still parallelize? | **Yes**, all N. Adjacent fragments were placed on different nodes. |
| D3 | **M > N**: range? | Usually **one** node (only N fat ranges). |
| D4 | **M < N**: nodes per HRPS query? | About **M** (5 or 6), not all N. |
| D5 | **M < N**: why spread fragments over all N anyway? | **Inter-query** parallelism / multiuser throughput. |
| D6 | **M < N**: hash’s mistake? | Wakes **all N**, extra startup and termination. |
| D7 | **M < N**: who has better response time? | **HRPS**. |
| D8 | **M < N**: who has better throughput? | **HRPS** (right intra-query M, fragments on every node). |

---

## Deck E — Directory, vocabulary of time

| # | Question | Answer |
|---|----------|--------|
| E1 | What is the range table? | Directory: fragment **range → processor**. |
| E2 | **CS** in Gamma? | **0.243 ms** per entry searched. |
| E3 | Equation (4) in words? | Same square root, but the denominator adds **Cardinality × CS / TuplesPerQ**. |
| E4 | Which search does the **optimizer** use? | **Binary** search. |
| E5 | Which formula **sizes** fragments in the experiments? | Equation **(4)** (the linear-search cost model) with the measured CS. |
| E6 | When does HRPS collapse toward range? | When directory search would dominate a tiny query (lecture: **Debit-Credit**). |
| E7 | **μ**, **τ**, **λ**? | Service time; response time = service + queue; throughput. Closed system: **λ = 1/μ**. |
| E8 | Intra-query parallelism? | Nodes used for **one** query. |
| E9 | Inter-query parallelism? | Nodes used by **concurrent independent** queries. |

---

## Deck F — Gamma setup

| # | Question | Answer |
|---|----------|--------|
| F1 | Nodes in the experiment? | **24** database + **8** load generators. Hypercube has 32. |
| F2 | Page size and buffer pool? | **8 KB** pages, **2 MB** buffer, **LRU**, elevator scheduler. |
| F3 | Tuple and table size? | **208** bytes; **10** relations × **1,000,000** tuples (Wisconsin). |
| F4 | Tuples per 8 KB page? | About **36**. |
| F5 | Index in the experiments? | **Clustered** index on the partitioning attribute. |
| F6 | SCSI cache? | **45 KB** read-ahead. ~**2×** on a single-user sequential scan. Weak once seeks start. |
| F7 | Single-node vs multi-node control path? | Single node: **direct**. Multi-node: **Query Manager** (start, done, commit/abort). |
| F8 | Why multiuser throughput, not just response time? | Extra messages can be hidden inside a long query, but they still **occupy nodes**. |

---

## Deck G — Results

| # | Question | Answer |
|---|----------|--------|
| G1 | Tiny query? | **0.001%**, **10** tuples, **0.08 s** on one node. |
| G2 | M, FC, fragments for the tiny query? | **M ≈ 0.057** → 1 node. **FC ≈ 175**. **~5,700** fragments. |
| G3 | Tiny, MPL = 1: range/HRPS vs hash? | About **8×** throughput. |
| G4 | Tiny, MPL = 160? | About **30×**. |
| G5 | Range vs HRPS on the tiny query? | Within **2.5%**. Both usually use **one** node. |
| G6 | Why does the tiny curve bend after MPL ~12? | Random predicates **collide** on one of the 24 nodes. |
| G7 | Heavy query? | **10%**, **100,000** tuples, **54.33 s** on one node. |
| G8 | M and fragments for the heavy query? | **M = 90**, **895** fragments. Each of 24 nodes does ~**3.7×** ideal work. |
| G9 | How many nodes for the heavy query? | Hash and HRPS: **all 24**. Range: **at most 3**. |
| G10 | Heavy, MPL = 1: hash/HRPS vs range? | **16%** higher throughput. |
| G11 | Why no SCSI-cache wiggle on the 10-tuple query? | It touches **one page** (sometimes two). |
| G12 | Equal mix, MPL = 1? | HRPS **+11%** vs hash, **+46%** vs range. |
| G13 | Fragments in the equal mix? | **1,688**. Tiny → 1 node. Heavy → all nodes. |
| G14 | Why do hash and HRPS meet after MPL 4 in the equal mix? | The **10%** query dominates the average. |
| G15 | MPL = 50, heavy query is 5–7% of the mix? | HRPS beats **both** by about **30%**. |
| G16 | When does range beat hash? | Heavy query **under ~5%** of the mix. |
| G17 | When do hash and HRPS tie, ahead of range? | Heavy query **above ~10%**, until the disk saturates (~40%). |

---

## Deck H — Limits, skew, MAGIC

| # | Question | Answer |
|---|----------|--------|
| H1 | Small relation on a huge cluster? | If fragments **< N**, HRPS occupies a **subset** of nodes. |
| H2 | Many small relations? | **Bin packing**. Heuristic: fullest free disk that does not already hold this relation. |
| H3 | Skew example? | 4,000 rows with key 2, FC = 1,000 → **4–5 fragments on different nodes**. Equality uses 4 nodes, not 1. |
| H4 | Two limitations in the paper? | Work is not always parallelizable. **One-dimensional**: other attributes visit every fragment-holding node. |
| H5 | HRPS vs Copeland/Bubba? | **Orthogonal**. HRPS cuts fragments. Bubba heuristics **place** them (heat, temperature) and can reorganize. |
| H6 | What is **MAGIC**? | SIGMOD **1992** multi-attribute **grid**. Each cell is a fragment. |
| H7 | Range-on-age, two 50% queries, 9 nodes? | Average **5.5** nodes. Ideal is **1**. |
| H8 | MAGIC, low vs high correlation? | Low: average **3**. High: average **1**. |
| H9 | BERD in one line? | Primary **range on one attribute** plus an **auxiliary (attribute → node)** relation for the other. |
| H10 | BERD low vs high correlation? | Low: average **5.5** (like range). High: average **1**. |
| H11 | Throughput order on the MAGIC slides? | **MAGIC > BERD > range**. |
| H12 | Closed model: what is “better”? | **Higher throughput** as the client count grows. |
| H13 | Open model: what is “better”? | Higher arrival rate **λ** before response time blows up. |
| H14 | Are the Gamma plots closed or open? | **Closed** (throughput vs multiprogramming level). |

---

## Self-test checklist

- [ ] Contrast hash and range on an equality and on a small range
- [ ] Derive M from RT(M) = work/M + M×CP
- [ ] Explain why M is not the number of nodes that store the table
- [ ] Compute FC for the 10,000-tuple toy
- [ ] Walk M = N, M > N, and M < N
- [ ] Quote the tiny-query and heavy-query fragment counts
- [ ] State the mixed-workload sweet spot (about 5–7% heavy)
- [ ] Name the one-dimensional limit and what MAGIC changes
