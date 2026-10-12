# Gamma — Lecture-style slides

**DeWitt et al. · IEEE TKDE 1990 · CSCI 585**  
Bullets follow [`GammaLecture.pdf`](GammaLecture.pdf) (132 slides). Depth: [`Gamma_Study_Guide.md`](Gamma_Study_Guide.md).

**Note:** Slides **7–39** and **51–101** are mostly **systems/SQL review** `[L]`; Gamma-specific narrative runs **40–50**, **102–132**, with SQL block in the middle for query-plan literacy.

---

## Slides 1–6 — Title, relevance, award

- Course announcements (Brightspace, guest lecturers, office hours)
- **Gamma**, DeWitt et al., CSCI 585
- Assigned reading: TKDE survey
- Built for Internet-scale data **before** Facebook/Amazon-scale existed; **SQL hides parallelism** → **Redshift**, **BigQuery** today `[L]`
- **ACM Software System Award 2008** — lasting influence `[L]`

---

## Slides 7–29 — Disk, buffer pool, NIC, shared-nothing `[L]` review

- Disk vs DRAM: block vs byte addressable; volatile vs non-volatile
- Read/write through **buffer pool** frames; page lookup question
- Replacement (LRU-style discussion on deck)
- **NIC**, duplex, network-attached node
- **Shared-nothing:** Node 0 … N−1; theory vs practice (bottlenecks)
- **Virtual nodes:** 8-core box → eight logical nodes (CPU/memory/disk fractions)

---

## Slides 30–39 — Why many nodes, processes `[L]`

- **Speedup:** e.g. 10% raise for all employees; complex multi-dept aggregate
- **Scaleup:** DB grows 10× → add 10× nodes to hold response time
- Design: **partition data**, **decentralize**, **minimize coordination**
- Challenges: failures, network latency, hard algorithms
- Program vs **process** vs **thread**; Vert.x / Pekko as modern concurrency frameworks

---

## Slides 40–50 — Gamma architecture & declustering `[L]` + paper `[P]`

- Each node: local **fragment**, local **indexes** (B+, hash), local **CC/recovery**
- `SELECT * FROM Emp WHERE salary=60K` → routed node(s); user unaware of parallelism
- **Horizontal declustering** = partitioning = **sharding**
- No partition attribute: **random**, **round-robin**
- One attribute: **hash** or **range** — **admin** chooses column
- **MOD:** `x % N` → remainder in `0 … N−1`
- **Hash:** equality on partition key → **one** node; range or non-key predicate → **all** nodes
- **Fragment** = shard on one node

---

## Slides 51–101 — SQL & relational algebra review `[L]`

*Skim if you know undergrad DB; needed to read Gamma query trees.*

- **DDL:** `CREATE TABLE` Emp, Dept (ssnum, name, age, salary, dno; dept metadata)
- **SELECT:** target list, `FROM`, `WHERE`; `DISTINCT`; conceptual steps (qualify → project → dedupe)
- Quick review queries: age &lt; 18; floor = 1 join; manager of Toy dept; older than manager
- **Relational algebra:** σ, π, rename; joins, cartesian product, union, difference
- **Query trees** for Emp/Dept examples; equivalence of SQL and algebra plans
- *(Slides repeat Emp/Dept figures — use for tracing σ and ⋈ order.)*

---

## Slides 102–108 — Gamma architecture (repeat) `[L]`

- Same bullets as 40–41: fragment, indexes, local CC/recovery, SQL hides parallelism

---

## Slides 109 — Software on one node `[L]`

- Processes ≈ **threads** sharing memory
- Fixed thread pool at init; listen on socket
- Scheduler message assigns **operator identity** (switch: select, project, hash-join build/probe, …)

---

## Slides 110–112 — Pipelined execution & plan reorder `[L]`

- Query: `TMP2 ← σ_{dname=Toy}(Emp ⋈ Dept)` / `SELECT * INTO TMP2 …`
- Steps: store TMP2, select, join; stream Emp/Dept through join → select → store
- **Can we flip Emp and Dept?** — yes: **push σ on Dept first**, then join Emp (fewer probe tuples)

---

## Slides 113–115 — Operator streams & hash-join `[L]`

- Input/output = **record streams**; select may be full **scan** if no predicate
- Join = **build** (hash on join attr) then **probe** — **sequential** on one node
- Toy example: build on `dno` of filtered Dept; probe with Emp.dno

---

## Slides 116–120 — Scheduler vs data path `[L]` + paper `[P]`

- “How does operator know where to send tuples?” → configured up front (**split table**)
- Scheduler **coordinates** but is **NOT** on data path — **control messages only**
- On-demand vs breadth-first scheduling (store operator example)
- Scheduler knows completion when operators report **done**

---

## Slides 121–123 — Graceful hash-join & hybrid `[L]`

- Build table larger than RAM: partition into **M buckets**, same hash both sides, join bucket-by-bucket
- **First bucket** of inner/outer can stay in memory (hybrid idea)
- Reduces one big join to **M small joins**

---

## Slides 124–125 — Multi-node join & split table `[P][L]`

- Parallel hash-join across nodes (lecture figure)
- **Split table:** read input stream → process tuple → route output; on EOS close streams → **control** to scheduler

---

## Slides 126–127 — Availability `[P][L]`

- **Interleaved declustering:** backup spread; node 1 failure → load on 0, 2, 3; cluster size vs availability tradeoff
- **Chained declustering:** primary + backup on **adjacent** nodes; push failed load along chain; balance survivors

---

## Slides 128–131 — Speedup & scaleup experiments `[P][L]`

- Selection speedup: `TMP2 ← σ_{attr>x}(R)` — why **superlinear**?
- **Multi-zone disk:** outer zones faster; **N disks** vs one → higher aggregate transfer + shorter seeks
- **Scaleup:** input grows 1M → 6M tuples with more nodes
- **Join speedup:** `TMP2 ← A ⋈ B'`

---

## Slide 132 — Gamma contributions `[L]`

- Alternative declustering: **HRPS**, **MAGIC** (next lecture)
- **Hybrid hash-join**
- **Chained declustering** (high availability)
- **Dataflow** paradigm
- **Split operator** / modular routing
- Nodes **with** and **without** disk
- **Speedup & scaleup** methodology for multi-node DBMSs
- **SQL to hide parallelism**

---

## Suggested timed pass (~90 min)

| Block | Slides | Time |
|-------|--------|------|
| Motivation + shared-nothing | 5–6, 31–38 | 15 min |
| Declustering | 40–50 | 15 min |
| SQL/algebra (if rusty) | 51–101 | 30 min skip / 45 min full |
| Dataflow + scheduler | 109–120, 125 | 20 min |
| Hash-join + availability + eval | 121–131 | 15 min |
| Contributions | 132 | 5 min |
