# BG Paper — Flashcards

**Paper:** Barahmand & Ghandeharizadeh, CIDR 2013  
**Companion:** `BG_Paper_Study_Guide.md` (full detail) · `BG_Paper_Exam_QA.md` (exam answers)

**How to use:** Cover the answer, recite aloud, then check. Cards are grouped by topic; shuffle within a deck for exams.

---

## Deck A — Big picture & motivation


| #   | Question                                           | Answer                                                                                                                            |
| --- | -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| A1  | What does **BG** stand for / measure?              | A **benchmark** for **interactive social networking actions** on data stores.                                                     |
| A2  | Who wrote BG and where was it published?           | **Sumita Barahmand** & **Shahram Ghandeharizadeh**, **USC**; **CIDR 2013**.                                                       |
| A3  | What research question does BG address?            | How to **quantitatively compare** stores/designs for **interactive** social networking, including **caches** and **consistency**. |
| A4  | What is an **SNA**?                                | **Social Networking Action** — one logical user operation (e.g. View Profile).                                                    |
| A5  | What does “small touch of big data” mean in BG?    | Each action touches a **tiny fraction** of a **huge** dataset (millions of members).                                              |
| A6  | Is BG a bulk analytics benchmark?                  | **No** — many concurrent **small** reads/writes, not warehouse scans.                                                             |
| A7  | Why isn’t **TPC-C** enough?                        | OLTP **orders/warehouses** — wrong schema and access patterns for SN graphs.                                                      |
| A8  | Why isn’t **YCSB** enough?                         | Generic keys — no **social actions**, no **unpredictable-read** SLA.                                                              |
| A9  | What is **LinkBench** vs BG (one line)?            | LinkBench (Facebook, later) is another SN benchmark; BG emphasizes **SLA + unpredictable data + CASQL**.                          |
| A10 | Main headline result across SQL-X, CASQL, MongoDB? | **No system wins every action**; **mixed** workloads often favor **CASQL** (RDBMS + memcached + invalidation).                    |


---



## Deck B — Metrics & SLA


| #   | Question                                  | Answer                                                                                                    |
| --- | ----------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| B1  | Define **SoAR**.                          | **Social Action Rating** — max **throughput** (actions/sec) while **SLA holds**.                          |
| B2  | Define **Socialites**.                    | Max **concurrent threads** (socialites) while **SLA holds** (**closed** model).                           |
| B3  | Name three parts of a typical BG **SLA**. | **Latency percentile** (e.g. 95% ≤ 100 ms), cap on **% unpredictable**, **duration** (e.g. 10 min).       |
| B4  | Example SLA numbers to memorize?          | **95%** < **100 ms**, ≤ **0.01%** (or **0.001%**) unpredictable, **10 minutes**.                          |
| B5  | What is **unpredictable data**?           | Client gets data **stale, invalid, or inconsistent** vs true DB state (per validation).                   |
| B6  | Is a fast wrong answer OK for SLA?        | **No** — wrong body with 200 OK still counts as unpredictable.                                            |
| B7  | What is **freshness confidence**?         | Metric for **weak consistency** (e.g. TTL) — how trustworthy cached reads are.                            |
| B8  | Why percentiles, not mean latency?        | **Tail latency** drives user experience; good average can still fail **p95**.                             |
| B9  | **Closed** vs **open** workload?          | **Closed:** thread waits for reply before next request (BG **Socialites**). **Open:** fixed arrival rate. |
| B10 | How is SoAR found experimentally?         | **Search** load (binary/heuristic): increase until SLA breaks, report max feasible.                       |


---



## Deck C — Architecture


| #   | Question                          | Answer                                                                                           |
| --- | --------------------------------- | ------------------------------------------------------------------------------------------------ |
| C1  | What does **BGCoord** do?         | **Coordinates** experiment: config, workload split, aggregate metrics, **SoAR/Socialites**, viz. |
| C2  | What does **BGClient** do?        | **Worker** that issues SNAs as **threads/socialites**, reports latency/throughput.               |
| C3  | What is a **socialite**?          | **Simulated user** in the workload; often **one thread ≈ one socialite**.                        |
| C4  | What is **logical partitioning**? | Graph split into **fragments** so parallel clients scale without all hitting same rows.          |
| C5  | Why logical partitions matter?    | **Horizontal scale** to many nodes; reduce **contention** on hot members.                        |
| C6  | What is the **validation phase**? | Compares responses to **ground-truth logs** to detect **unpredictable** reads.                   |
| C7  | Does BG include monitoring?       | Yes — **visualization deck** for live bottleneck spotting.                                       |
| C8  | Is BG tied to one DB vendor?      | **No** — **modular**, same logical actions on SQL-X, CASQL, MongoDB, etc.                        |


---



## Deck D — Data model


| #   | Question                                            | Answer                                                                             |
| --- | --------------------------------------------------- | ---------------------------------------------------------------------------------- |
| D1  | Four main **conceptual** entities?                  | **Members**, **Friends**, **Resources**, **Manipulation**.                         |
| D2  | Key fields on **Members**?                          | `userid`, names, email, `profileImage`, `thumbnail` (size matters in experiments). |
| D3  | What does **Friends** represent?                    | Edge between users with `status` (e.g. **pending** vs **confirmed**).              |
| D4  | What is a **Resource**?                             | **Wall post** — `rid`, `creatorid`, `wallUserid`, `body`, `doc`.                   |
| D5  | Why **Manipulation**?                               | **Audit/history** of resource changes; helps **validation** of reads after writes. |
| D6  | Relationship sketch Members–Friends?                | Members linked by **Friends** edges (graph).                                       |
| D7  | Which attributes are the main **performance knob**? | **Thumbnail / image sizes** (2 KB, 12 KB, 500 KB in paper).                        |


---



## Deck E — Systems: SQL-X, CASQL, MongoDB


| #   | Question                              | Answer                                                                                    |
| --- | ------------------------------------- | ----------------------------------------------------------------------------------------- |
| E1  | What is **SQL-X**?                    | **RDBMS-only** implementation of BG schema/actions (no memcached).                        |
| E2  | What is **CASQL**?                    | **Cache-Augmented SQL** — SQL-X + **memcached** with app-managed keys.                    |
| E3  | MongoDB storage pattern (high level)? | Often **one document per member** with **embedded** friend arrays.                        |
| E4  | **AFR** in SQL vs MongoDB?            | SQL: **UPDATE** `Friends.status`. Mongo: **two document** updates (arrays on both users). |
| E5  | **AFR** in CASQL extra step?          | SQL update + **invalidate** cache keys (profiles, friend lists, pending, etc.).           |
| E6  | memcached **value size** trap?        | Default **~1 MB** max — oversized values (e.g. LF + many thumbnails) → **cache unused**.  |
| E7  | Symptom when cache values too large?  | memcached **idle**, traffic to SQL, **SoAR collapses** or **zero**.                       |
| E8  | VP on MongoDB vs SQL-X (typical)?     | Mongo: often **one doc read**; SQL: row + **LOB** for image.                              |


---



## Deck F — Eleven actions (acronyms)


| #   | Question             | Answer                                                                 |
| --- | -------------------- | ---------------------------------------------------------------------- |
| F1  | **VP**?              | **View Profile** (read).                                               |
| F2  | **LF**?              | **List Friends** — list + per-friend info/thumbnail (**N+1** pattern). |
| F3  | **SFR**?             | **Send Friend Request** (write).                                       |
| F4  | **AFR**?             | **Accept Friend Request** (write).                                     |
| F5  | **RFR**?             | **Reject Friend Request** (write).                                     |
| F6  | **UF**?              | **Unfriend** (write).                                                  |
| F7  | **LPFR**?            | **List Pending Friend Requests** (read).                               |
| F8  | **WR**?              | **Write Resource** / post on wall (write).                             |
| F9  | **RR**?              | **Read Resource** / view wall (read).                                  |
| F10 | **CR** / **SR**?     | **Comment** / **Share** resource (writes).                             |
| F11 | How many SNAs total? | **11**.                                                                |


---



## Deck G — Deep dives (VP, LF, AFR)


| #   | Question                                             | Answer                                                                 |
| --- | ---------------------------------------------------- | ---------------------------------------------------------------------- |
| G1  | Why does **500 KB** thumbnail kill VP SoAR on SQL-X? | **Huge payload** → tail latency → fails **95% < 100 ms**.              |
| G2  | Why is **LF** harder than **VP**?                    | **Many** friends × **round trips** / bytes (thumbnails per friend).    |
| G3  | LF + CASQL failure mode?                             | Single cache value **> 1 MB** → cannot cache list → poor SoAR.         |
| G4  | Read-heavy mix stresses which actions?               | **VP, LF, RR** → favors **cache** and fast reads.                      |
| G5  | Write-heavy mix stresses what?                       | **SFR, AFR, WR, CR** → **invalidation**, unpredictability, write path. |
| G6  | What access skew does BG use?                        | **Zipfian / D-Zipfian** — hot celebrities, realistic hotspots.         |
| G7  | Three action **mixes** in paper (Table 3)?           | **Read-heavy**, **balanced**, **write-heavier** (exact % in PDF).      |
| G8  | Write % examples in mixed experiments?               | **0.1%, 1%, 10%** writes.                                              |


---



## Deck H — Caching & unpredictability


| #   | Question                                      | Answer                                                                              |
| --- | --------------------------------------------- | ----------------------------------------------------------------------------------- |
| H1  | **Invalidation** on write?                    | **Delete/update** affected memcached keys after DB write.                           |
| H2  | **TTL** cache maintenance?                    | Entries expire after **N seconds**; writes may leave **stale** data until expiry.   |
| H3  | TTL values tested in paper?                   | **30, 60, 120** seconds (with ~**10%** writes in that experiment).                  |
| H4  | Higher TTL + writes → ?                       | **More unpredictable data** (stale keys live longer).                               |
| H5  | Invalidation unpredictability in experiments? | Often **< 0.001%**.                                                                 |
| H6  | What is **Gumball**?                          | USC **race-condition prevention** for cache+DB; can drive unpredictability → **0**. |
| H7  | Example unpredictable read after AFR?         | Friend list **missing** new friend (stale cache).                                   |
| H8  | How does BG detect unpredictability?          | **Read/write logs** → define acceptable values per read → compare response.         |


---



## Deck I — Experiments & conclusions


| #   | Question                                    | Answer                                                                         |
| --- | ------------------------------------------- | ------------------------------------------------------------------------------ |
| I1  | Three systems compared in main study?       | **SQL-X**, **CASQL**, **MongoDB**.                                             |
| I2  | Thumbnail sizes in VP experiments?          | **2 KB**, **12 KB**, **500 KB** (representative).                              |
| I3  | Scaling knob for drivers?                   | **# of BGClients** (1, 2, 4, 8, 16, …).                                        |
| I4  | What happens as threads **T** increase?     | Throughput up until **latency** or **unpredictability** violates SLA.          |
| I5  | TTL experiment: unpredictability at high T? | Can reach **~80–90%** unpredictable (bad configs); invalidation stays **low**. |
| I6  | Who wins on **every** action?               | **Nobody** — depends on action and mix.                                        |
| I7  | Who often wins **combined mix**?            | **CASQL** with **invalidation** (not TTL-only under writes).                   |
| I8  | BG design decision: unpredictable data?     | **First-class SLA metric**, not optional.                                      |


---



## Deck J — Related work & limits


| #   | Question                            | Answer                                                                                     |
| --- | ----------------------------------- | ------------------------------------------------------------------------------------------ |
| J1  | YCSB vs BG in one phrase?           | YCSB = generic cloud; BG = **social actions + unpredictability in SLA**.                   |
| J2  | CAP — what can BG study?            | Behavior under failures as **CP** vs **AP** systems.                                       |
| J3  | Feed following paper relevance?     | Motivates **hard-to-scale read-heavy** SN patterns.                                        |
| J4  | Facebook memcached / TAO relevance? | Real world uses **caches**; BG quantifies **consistency cost**.                            |
| J5  | One limitation of BG?               | **Implementation-dependent**; 11 actions ≠ entire Facebook.                                |
| J6  | Future work direction?              | More stores, **partitions/failures**, richer weak consistency, **Gumball**-style fixes.    |
| J7  | **ACID** vs **BASE** (context)?     | ACID = strong TX; BASE = eventual consistency for scale — SN sites often relax strictness. |
| J8  | Project URL?                        | **bgbenchmark.org**                                                                        |


---



## Deck K — Rapid fire (mixed exam style)


| #   | Question                                 | Answer                                                                                             |
| --- | ---------------------------------------- | -------------------------------------------------------------------------------------------------- |
| K1  | SoAR unit?                               | **Actions per second** (under SLA).                                                                |
| K2  | Socialites unit?                         | **Count of concurrent threads** (under SLA).                                                       |
| K3  | CASQL authoritative store?               | Still **SQL-X (RDBMS)**; memcached is **not** source of truth.                                     |
| K4  | Error rate vs unpredictable?             | **Different** — unpredictable = **wrong data**, not necessarily HTTP error.                        |
| K5  | BG modular means?                        | Same **logical** benchmark, pluggable **physical** implementations.                                |
| K6  | Why Manipulation for exams?              | Ties **wall writes** to **correctness validation**.                                                |
| K7  | Closed model models whom?                | **Interactive users** waiting on each action.                                                      |
| K8  | D-Zipfian purpose?                       | **Skewed** popularity — realistic load on hot members.                                             |
| K9  | 1000 friends × 2 KB thumbs vs memcached? | **~2 MB** > 1 MB limit — **caching breaks** unless value limit raised.                             |
| K10 | One-sentence BG summary?                 | **SLA-driven** SN action benchmark reporting **SoAR**, **Socialites**, and **unpredictable data**. |


---



## Self-test checklist

Before the exam, you should be able to:

- [ ] Draw BGCoord → BGClients → data store (+ memcached if CASQL)
- [ ] State SLA with example numbers
- [ ] Define SoAR and Socialites and how they differ
- [ ] Explain unpredictable data with one concrete example
- [ ] List all four entities and three storage styles
- [ ] Name ≥ 5 of 11 actions and explain VP, LF, AFR
- [ ] Contrast invalidation vs TTL under 10% writes
- [ ] Explain CASQL + 1 MB limit failure on LF
- [ ] State why no single system wins all actions