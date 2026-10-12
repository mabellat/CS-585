# Gamma — Flashcards

**Paper:** DeWitt et al., IEEE TKDE 1990 · **Lecture:** [`GammaLecture.pdf`](GammaLecture.pdf)  
**Companion:** [`Gamma_Study_Guide.md`](Gamma_Study_Guide.md) · [`Gamma_Exam_QA.md`](Gamma_Exam_QA.md)

Cover the answer, say it, then check.

---

## Deck A — Big picture

| # | Question | Answer |
|---|----------|--------|
| A1 | What is Gamma? | Parallel **SQL** RDBMS on **shared-nothing** nodes; **declustering** + **hash algorithms** + **dataflow**. **[P]** |
| A2 | Authors / venue / year? | **DeWitt** et al., **IEEE TKDE** 2(1), **March 1990**, Wisconsin. **[P]** |
| A3 | ACM award? | **Software System Award 2008**. **[L]** |
| A4 | Modern analogues (lecture)? | **Amazon Redshift**, **Google BigQuery** — SQL hides parallelism. **[L]** |
| A5 | What failed in DIRECT? | Shared memory, centralized control, parallelism **instead of** indexes for I/O-bound work. **[P]** |
| A6 | Three key technical ideas? | **Declustering**, **hash-based** parallel join/aggregate, **dataflow** scheduling. **[P]** |
| A7 | Shared-nothing citation? | **Stonebraker**, “Case for Shared Nothing,” 1986. **[R3]** |
| A8 | Speedup vs scaleup? | **Speedup:** more nodes, fixed data. **Scaleup:** data ∝ nodes, constant time. **[P][L]** |

---

## Deck B — Hardware & storage

| # | Question | Answer |
|---|----------|--------|
| B1 | iPSC/2 node specs (typical)? | **386**, **8 MB** RAM, **330 MB** disk, **45 KB** SCSI read cache. **[P]** |
| B2 | How many iPSC/2 nodes? | **32**. **[P]** |
| B3 | VAX prototype lesson on page size? | **2 KB** pages too small; **larger** pages better. **[P]** |
| B4 | NOSE? | Lightweight **threads**, shared memory, hypercube messaging on iPSC/2. **[P]** |
| B5 | WiSS? | **Wisconsin Storage System** — files, B+ trees, scans; predicates → **386 code**. **[P][R8]** |
| B6 | Three declustering strategies? | **Round-robin**, **hash**, **range**. **[P]** |
| B7 | Default for **query results**? | **Round-robin** (or hash) for even fragment sizes. **[P]** |
| B8 | Index placement? | **Per fragment**; clustered index need not be on partition key. **[P]** |
| B9 | Diskless nodes used for? | **Join** (and aggregate) experiments — extra CPU without local data disk. **[P]** |
| B10 | Tuple batch size on wire? | ~**8 KB** batches. **[P]** |

---

## Deck C — Processes & dataflow

| # | Question | Answer |
|---|----------|--------|
| C1 | QM role? | Per-user parse, optimize, compile; embedded SQL or **gdl**. **[P]** |
| C2 | When is scheduler skipped? | **Single-site** (one-node) query — QM → node directly. **[P][L]** |
| C3 | Scheduler’s job? | **Control** — start operators, detect completion; **not** per-tuple routing. **[P][L]** |
| C4 | Split table? | **Demux** — each output tuple → destination processor/port per plan. **[P][L]** |
| C5 | Control messages per operator per processor? | About **3** (initiate, done, plus stream close semantics). **[P][R1]** |
| C6 | End of input stream? | Close outputs → **end-of-stream** → control **done** to scheduler. **[P][L]** |
| C7 | Operator pool trick? | Processes **primed at init**; assigned via new-task port. **[P]** |
| C8 | Lecture: thread role message? | Scheduler tells thread to act as select, build, probe, etc. **[L]** |

---

## Deck D — Algorithms

| # | Question | Answer |
|---|----------|--------|
| D1 | Default join algorithm? | **Hybrid hash**. **[P][R4]** |
| D2 | Other joins implemented? | Sort-merge, Grace, simple hash. **[P]** |
| D3 | Hybrid hash idea? | In-memory **first bucket** + spill rest; probe first bucket; join rest from disk. **[P][L]** |
| D4 | Parallel join partitioning? | Hash join attribute → **join nodes** via split table. **[P]** |
| D5 | Parallel aggregate pattern? | Local partials → **redistribute** by GROUP BY hash → finalize. **[P]** |
| D6 | Update changes partition key? | Route via **split table** to correct node. **[P]** |
| D7 | Hash + equality on partition attr? | **One** node. **[L]** |
| D8 | Hash + range on partition attr? | **All** nodes (declustered by hash). **[P][L]** |
| D9 | Non-equijoin at paper time? | **Not** supported initially. **[P]** |
| D10 | Optimizer tree shape (VAX)? | **Left-deep** (memory limits). **[P]** |

---

## Deck E — Availability & transactions

| # | Question | Answer |
|---|----------|--------|
| E1 | Chained declustering backup site? | **(i+1) mod M** next disk in ring. **[P][R5]** |
| E2 | Failover data movement? | **None** — update in-memory **fragment map**. **[P]** |
| E3 | Load increase per survivor (chained)? | Roughly **1/(M−1)** of failed node’s load each. **[P]** |
| E4 | Interleaved declustering risk? | **Two** failures can lose availability; small clusters recommended. **[P][L]** |
| E5 | Locking? | **2PL**, file/page, S/X/IS/IX/SIX. **[P]** |
| E6 | Deadlock handling? | **Local** graphs + **central** periodic merge; abort **fewest locks**. **[P]** |
| E7 | Recovery direction? | **WAL**, **LSN**, **ARIES**-style undo. **[P]** |

---

## Deck F — Evaluation & lecture extras

| # | Question | Answer |
|---|----------|--------|
| F1 | Why superlinear speedup on scan? | Smaller **per-disk** fragments → shorter **seeks** (∝ √ distance). **[P][L]** |
| F2 | One disk vs 30 disks (1M tuples)? | ~**66%** of one disk vs ~**2%** each. **[P]** |
| F3 | Multi-zone disk effect? | More disks → more **outer-zone** bandwidth. **[L]** |
| F4 | Results returned to host in benchmarks? | **No** — stored in DB to avoid host/link noise. **[P]** |
| F5 | Single-tuple select on partition key? | **One** node; time rises if B-tree **taller**. **[P]** |
| F6 | Push σ before join (lecture)? | Read **Dept**, filter `dname=Toy`, **then** join Emp — fewer tuples to join. **[L]** |
| F7 | HRPS / MAGIC on contributions slide? | **Next lecture** — alternative declustering. **[L]** |
| F8 | Virtual nodes? | Split one multicore machine into **N logical** nodes. **[L]** |
