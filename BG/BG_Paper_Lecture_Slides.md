# BG Paper — Lecture-style slides

**Barahmand & Ghandeharizadeh · CIDR 2013 · USC**  
Short bullets per “slide” — use with `BG_Paper_Study_Guide.md` for depth.

---

## Slide 1 — Title

- **BG:** A Benchmark to Evaluate Interactive Social Networking Actions
- USC Database Lab · **bgbenchmark.org**
- Compare data stores for **real SN workloads**, not generic OLTP

---

## Slide 2 — Why we care

- Facebook, LinkedIn, Twitter = **data store + app** at huge scale
- Choices: **SQL**, **NoSQL**, **memcached**, **CASQL**, graph designs
- Need **fair quantitative comparison**, not marketing benchmarks

---

## Slide 3 — What went wrong before BG

- **TPC-C / TPC-E:** warehouses & orders — not friend graphs
- **YCSB:** generic keys — no social **actions**
- **OO7:** obsolete object benchmark
- Gap: **interactive social networking actions (SNAs)**

---

## Slide 4 — BG in one sentence

- Domain-specific, **SLA-driven** benchmark
- Measures **throughput**, **concurrency**, and **correctness of reads** (unpredictable data)
- **Small reads/writes** on **very large** social graphs

---

## Slide 5 — Six design decisions

1. Ratings: **SoAR** + **Socialites** under **SLA**
2. **Unpredictable data** in SLA (first-class)
3. Realistic workload: **tiny touch** of **big data**
4. **Store-agnostic** (SQL-X, CASQL, MongoDB, …)
5. **Logical partitioning** for scale-out
6. **Visualization** during runs

---

## Slide 6 — Key vocabulary

- **SNA** = one user-visible action (VP, LF, …)
- **Socialite** = simulated user (≈ one thread)
- **SLA** = latency % + unpredictability cap + duration
- **SoAR** = max **actions/sec** meeting SLA
- **Socialites** = max **concurrent threads** meeting SLA

---

## Slide 7 — Example SLA

- **95%** of actions ≤ **100 ms**
- ≤ **0.01%** (or **0.001%**) **unpredictable** reads
- Held for **10 minutes** (not a 30-second cherry-pick)

---

## Slide 8 — SoAR vs Socialites

| | SoAR | Socialites |
|---|------|------------|
| Optimizes | **Throughput** | **Concurrency** |
| Model | Load search | **Closed** (wait for reply) |
| Unit | actions/sec | # threads |

---

## Slide 9 — Architecture overview

- **BGCoord:** config, split workload, aggregate, report
- **BGClient(s):** generate SNAs, measure latency
- **Target:** SQL-X / CASQL / MongoDB
- **Validation:** logs → detect stale/wrong reads

---

## Slide 10 — BGCoord details

- Reads Table-2-style **parameters**
- Assigns **fraction of workload** per BGClient
- Periodic **throughput + latency** aggregation
- Feeds **visualization deck**

---

## Slide 11 — BGClient details

- Runs **many threads** (socialites)
- Issues actions per **mix** + **Zipfian** skew
- **Closed loop:** next action only after response

---

## Slide 12 — Logical partitioning

- Split members into **fragments**
- Parallel clients → **shared-nothing** scaling
- Avoid two threads **conflicting** on same member where needed

---

## Slide 13 — Conceptual schema (4 tables)

- **Members** — users + **thumbnails**
- **Friends** — edges + **status** (pending/confirmed)
- **Resources** — wall posts
- **Manipulation** — change history (audit)

---

## Slide 14 — Members entity

- Identity: `userid`, `username`, …
- **profileImage**, **thumbnail** → **payload size** drives VP/LF results
- Experiments: **2 KB / 12 KB / 500 KB**

---

## Slide 15 — Friends & graph

- `(userid1, userid2, status)`
- Pending → confirmed via **AFR**
- Drives **read** actions (LF, LPFR) and **write** churn

---

## Slide 16 — Resources & Manipulation

- **Resource** = post on a **wall**
- **Manipulation** = who changed what, when
- Supports **CR** and **validation** after writes

---

## Slide 17 — Three implementations

| Label | Stack |
|-------|--------|
| **SQL-X** | RDBMS only |
| **CASQL** | RDBMS + **memcached** |
| **MongoDB** | Document store |

Same **11 actions**, different physical layout

---

## Slide 18 — SQL-X

- Normalized tables, SQL + joins
- **ACID** for friend updates
- VP/LF pay cost of **LOBs** and **joins**

---

## Slide 19 — MongoDB

- **Document per member**, embedded friend arrays
- VP: often **1 read**
- LF: may be **N reads** (one per friend doc)
- AFR: **2 document updates**

---

## Slide 20 — CASQL

- DB = source of truth; **memcached** for hot keys
- Must **invalidate** or use **TTL** on writes
- **Value size limit ~1 MB** — design constraint

---

## Slide 21 — Eleven actions (list)

1. VP · 2. LF · 3. SFR · 4. AFR · 5. RFR  
6. UF · 7. LPFR · 8. WR · 9. RR · 10. CR · 11. SR  

Reads vs writes configurable in **mixes**

---

## Slide 22 — View Profile (VP)

- Read one member + **thumbnail**
- **Image size** dominates latency
- 500 KB → SQL-X may **fail SLA** (SoAR → 0)

---

## Slide 23 — List Friends (LF)

- List IDs + **per-friend** thumb/info → **N+1**
- High bytes + many round trips
- Worst case for **cache value size**

---

## Slide 24 — Accept Friend Request (AFR)

- SQL: update **Friends.status**
- Mongo: update **both** member docs
- CASQL: SQL + **invalidate** many keys
- Bad invalidation → **unpredictable** VP/LF

---

## Slide 25 — Workload knobs

- Scale (# members), # **BGClients**, # threads
- **Action mix** (3 presets in paper)
- SLA thresholds, run **duration**
- Write %: **0.1% / 1% / 10%**

---

## Slide 26 — Three mixes (Table 3)

- **Read-heavy:** VP, LF, RR → cache-friendly
- **Write-heavy:** SFR, AFR, WR, CR → invalidation stress
- **Combined:** realistic; **no single winner**

---

## Slide 27 — Access distribution

- **D-Zipfian / Zipfian** skew
- Few **hot** members & walls
- Tests cache effectiveness & contention

---

## Slide 28 — Unpredictable data defined

- Not “server crashed”
- Client sees **stale / invalid / inconsistent** data vs ground truth
- Fast **wrong** answer still fails SLA

---

## Slide 29 — Why unpredictability matters

- Production SN sites use **memcached**
- Speed without freshness breaks **friend graph** semantics
- BG forces tradeoff into the **score**

---

## Slide 30 — Detection mechanism

- **Read/write logs** during benchmark
- For each read: set of **acceptable values**
- Response outside set → **unpredictable** event

---

## Slide 31 — Invalidation (good CASQL)

- On write: update DB → **delete** stale cache keys
- Unpredictability **&lt; 0.001%** in paper runs
- **Gumball** can push toward **zero** races

---

## Slide 32 — TTL (weak consistency)

- Keys expire at **30 / 60 / 120 s**
- No immediate invalidation on write
- Higher TTL + **10% writes** → **high % unpredictable** (80–90% possible)

---

## Slide 33 — Invalidation vs TTL (exam)

| | Invalidation | TTL |
|---|--------------|-----|
| Freshness | Stronger | Weaker |
| Write cost | Invalidate work | Cheap |
| Unpredictable reads | Low | Can be **very high** |

---

## Slide 34 — CASQL trap: 1 MB limit

- LF caches **friends + thumbnails** as one value
- **1000 × 2 KB ≈ 2 MB** &gt; 1 MB limit
- memcached **unused** → SoAR **collapses**

---

## Slide 35 — VP results vs image size

- **2 KB:** all systems viable
- **12 KB:** stress visible
- **500 KB:** SQL-X **cannot** meet 100 ms p95

---

## Slide 36 — Mixed workload results

- Per-action leader **rotates** (SQL-X, Mongo, CASQL)
- **Combined mix:** **CASQL + invalidation** often **best SoAR**
- More writes → hurts **TTL** more than **invalidation**

---

## Slide 37 — Scaling BGClients

- Throughput ↑ with more clients until bottleneck
- SLA breaks on **tail latency** or unpredictability
- Shows queueing “cliff” in graphs

---

## Slide 38 — Computing SoAR (procedure)

1. Fix SLA + mix  
2. Trial at candidate load  
3. Measure latency + validation  
4. **Binary search** max feasible load  

Many experiments → figure on search cost

---

## Slide 39 — BG vs YCSB (table row)

- YCSB: generic, latency/throughput
- BG: **social schema + actions + unpredictable SLA + cache study**

---

## Slide 40 — Related citations (name-drop)

- **YCSB**, **TPC**, **memcached**, **CAP**
- **Feed following** (DBSocial)
- **Gumball** (USC cache races)
- **LDBC SNB** (later graph benchmark)

---

## Slide 41 — Limitations

- Winners depend on **implementation** details
- Only **11** actions (no search, ads, DM)
- SLA numbers are **parameters** — change them, change winner

---

## Slide 42 — Future work (paper §8)

- More data stores & **failure** modes (CP/AP)
- Richer weak-consistency models
- Broader adoption as SN standard

---

## Slide 43 — Exam cheat sheet (memorize)

- **95% / 100 ms / 0.01% unpredictable / 10 min**
- **SoAR** = throughput, **Socialites** = threads
- **SQL-X · CASQL · MongoDB**
- **No system wins all actions**
- **CASQL** wins many **mixed** cases with **invalidation**
- **VP** = images, **LF** = N+1 + 1 MB cache trap

---

## Slide 44 — One practice question

**Q:** Why not YCSB for Facebook-like workloads?  
**A:** No social action semantics, no unpredictable-read SLA, wrong schema.

---

## Slide 45 — Where to go deeper

- Full narrative: **`BG_Paper_Study_Guide.md`**
- Drill: **`BG_Paper_Flashcards.md`**
- Exam Q&A: **`BG_Paper_Exam_QA.md`**
- Figures: **`BG_Paper.pdf`** (SoAR vs T, TTL, write mix, BGClients)

---

### Suggested lecture flow (45 min)

| Block | Slides |
|-------|--------|
| Motivation | 1–4 |
| Design & metrics | 5–8 |
| Architecture | 9–12 |
| Data model | 13–16 |
| Implementations | 17–20 |
| Actions | 21–24 |
| Workload | 25–27 |
| Unpredictability & cache | 28–33 |
| Results | 34–37 |
| Method & wrap-up | 38–45 |
