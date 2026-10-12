# BG — Example questions and answers

Use for **closed-book practice** and **oral explanations**. Answers are concise; expand with examples from [`BG_Paper_Study_Guide.md`](BG_Paper_Study_Guide.md) when presenting.

**Paper:** Sumita Barahmand and Shahram Ghandeharizadeh. *BG: A Benchmark to Evaluate Interactive Social Networking Actions.* **CIDR 2013.** **[P]**

---

## Section A — Motivation & context

**Q1. What problem does BG solve?**  
**A:** There was no **domain-specific, SLA-driven** way to compare data stores for **interactive social networking**. TPC and YCSB measure the wrong workload. BG issues **social networking actions (SNAs)** — small reads and writes on a **huge** social graph — and scores systems by **throughput**, **concurrency**, and **correctness of reads**. **[P]**

**Q2. Why are TPC-C / TPC-E a poor fit?**  
**A:** They model **OLTP** (warehouses, orders, inventory). Schema and access patterns are not **friend lists, walls, thumbnails, or friend-request state**. **[P]**

**Q3. Why is YCSB a poor fit?**  
**A:** YCSB is **generic key-value / cloud serving**. It has no **social action semantics**, no friend-graph operations, and no **unpredictable-read** term in the SLA. **[P]**

**Q4. What does “small touch of big data” mean?**  
**A:** The dataset can hold **millions of members**, but each action touches a **tiny fraction** — one profile, one friend list, or one post. BG is **many concurrent small operations**, not a bulk scan or warehouse query. **[P]**

**Q5. Is BG a product, a database, or a graph-engine benchmark?**  
**A:** **None of those.** It is a **benchmark framework**: the same **11 logical actions** can be implemented on **SQL-X**, **CASQL**, **MongoDB**, or another store. **[P]**

**Q6. How does BG differ from LinkBench (one line)?**  
**A:** **LinkBench** (Facebook, later) is another social-graph benchmark. **BG** (USC, CIDR 2013) emphasizes **SLA ratings (SoAR / Socialites)**, **unpredictable data**, and **cache-augmented SQL**. **[P]**

---

## Section B — Metrics & SLA

**Q7. Define SoAR and Socialites.**  
**A:** **SoAR (Social Action Rating)** is the **maximum actions per second** while the SLA holds. **Socialites** is the **maximum number of concurrent threads** (each a simulated user) while the SLA holds, under a **closed** model. **[P]**

**Q8. Name the three parts of a typical BG SLA.**  
**A:** (1) **Latency percentile** — e.g. **95%** of actions ≤ **100 ms**. (2) Cap on **unpredictable data** — e.g. ≤ **0.01%** or **0.001%**. (3) **Duration** — e.g. **10 minutes**, so a short spike does not count. **[P]**

**Q9. Why percentiles instead of mean latency?**  
**A:** Social sites care about **tail latency**. A strong average can still fail **p95**: if 1 in 20 requests is slow, users churn, and BG’s SLA fails. **[P]**

**Q10. What is the closed workload model, and which rating uses it?**  
**A:** Each thread issues **one request at a time** and waits for the response before the next. That models an **interactive user**. **Socialites** uses this closed model. An **open** loop would inject arrivals even when the system is backed up. **[P]**

**Q11. How is SoAR found experimentally?**  
**A:** Fix the SLA and mix. Run a trial at a candidate load. If the SLA holds, **increase** load; if it breaks, **decrease**. A **binary / heuristic search** reports the maximum feasible throughput. The search itself takes many trials. **[P]**

**Q12. SoAR vs Socialites — what each optimizes.**  
**A:** **SoAR** searches **load intensity** and reports **actions/sec**. **Socialites** searches **parallelism** (thread count) under the closed model. More threads usually raise throughput **until** latency or unpredictability violates the SLA. The two optima are related but not identical. **[P]**

---

## Section C — Architecture

**Q13. What do BGCoord and BGClient do?**  
**A:** **BGCoord** reads experiment parameters, assigns each client a **fraction of the workload**, aggregates latency and throughput, and produces **SoAR / Socialites** plus visualization. **BGClient** runs **socialite threads** that issue SNAs and record timing. **[P]**

**Q14. What is a socialite?**  
**A:** A **simulated user** in the workload. Often **one thread ≈ one socialite**, so concurrent actions look like independent users. **[P]**

**Q15. What problem does logical partitioning solve?**  
**A:** The social graph is split into **fragments** (by member-id range or hash). Each BGClient mostly works on its fragment, so parallel workers do not all contend on the same members and the benchmark can **scale out** shared-nothing style. **[P]**

**Q16. What is the validation phase?**  
**A:** Separate logic compares each read response to **ground-truth read/write logs**. If the returned value is outside the set of **acceptable** values, the action counts as **unpredictable**. A **200 OK with a wrong body** still fails. **[P]**

**Q17. Is BG tied to one database vendor?**  
**A:** **No.** It is **modular**: the same logical actions, different **physical** implementations (SQL-X, CASQL, MongoDB, and others). **[P]**

---

## Section D — Data model & the eleven actions

**Q18. Name the four conceptual entities.**  
**A:** **Members** (accounts, including thumbnails), **Friends** (edges with **pending / confirmed** status), **Resources** (wall posts), **Manipulation** (audit history of resource changes, used for validation). **[P]**

**Q19. Which attribute is the main performance knob, and why?**  
**A:** **Thumbnail / image size** (paper uses **2 KB, 12 KB, 500 KB**). Actions that ship images (**VP**, **LF**) are dominated by **payload bytes**, not just query shape. **[P]**

**Q20. List the eleven SNAs and mark read vs write.**  
**A:** **Reads:** View Profile (**VP**), List Friends (**LF**), List Pending Friend Requests (**LPFR**), Read Resource (**RR**). **Writes:** Send Friend Request (**SFR**), Accept Friend Request (**AFR**), Reject Friend Request (**RFR**), Unfriend (**UF**), Write Resource (**WR**), Comment on Resource (**CR**), Share Resource (**SR**). **[P]**

**Q21. What does View Profile do, and why does 500 KB matter?**  
**A:** **VP** fetches **one member** plus thumbnail. At **500 KB**, moving the payload blows the **95% ≤ 100 ms** tail. **SQL-X** can have **SoAR → 0** on VP because it cannot meet the SLA. **[P]**

**Q22. Why is List Friends harder than View Profile?**  
**A:** **LF** reads friend ids, then **each** friend’s info and thumbnail — an **N+1** pattern. Bytes and round trips scale with degree. Bundling the whole list into one cache value can exceed **memcached’s ~1 MB** limit. **[P]**

**Q23. What does Accept Friend Request do?**  
**A:** It confirms a pending friendship. **SQL-X:** `UPDATE` the `Friends` row’s **status**. **MongoDB:** update **both** members’ embedded arrays (add confirmed, drop pending). **CASQL:** the SQL update **plus invalidation** of related cache keys (profiles, friend lists, pending lists). A bad or late invalidation makes the next **VP/LF** **unpredictable**. **[P]**

**Q24. What are the three action mixes, and what does each stress?**  
**A:** **Read-heavy** stresses **VP, LF, RR** and favors caches. **Write-heavy** stresses **SFR, AFR, WR, CR** and stresses invalidation and write throughput. **Combined** is realistic: **no single store wins every action**. Exact percentages are in the paper’s Table 3. **[P]**

---

## Section E — SQL-X, CASQL, MongoDB

**Q25. Define SQL-X, CASQL, and the MongoDB layout.**  
**A:** **SQL-X** is an industrial **RDBMS only** (normalized tables, joins, ACID writes). **CASQL** is that same RDBMS plus **memcached**; the database remains the **source of truth**. **MongoDB** often stores **one document per member** with **embedded** friend arrays. **[P]**

**Q26. View Profile on each system?**  
**A:** **SQL-X:** row fetch plus a **LOB** for the image. **MongoDB:** often **one document** read. **CASQL:** try a profile key in memcached; on miss, read SQL and fill the cache. Writes that change the profile must **invalidate** or wait for **TTL**. **[P]**

**Q27. What is the memcached 1 MB trap?**  
**A:** Classic memcached caps **value size** at about **1 MB**. **List Friends** that caches “all friends + all thumbnails” as one value can exceed it — e.g. **1000 friends × 2 KB ≈ 2 MB**. The cache is unused, memcached sits **idle**, every read hits SQL, and **SoAR collapses**. Raising the value limit would change the result. **[P]**

**Q28. Who is the source of truth in CASQL?**  
**A:** The **RDBMS (SQL-X)**. Memcached holds **copies** of hot keys. If the cache is wrong or oversized, correctness and the SLA still depend on the database and on how keys are invalidated. **[P]**

**Q29. Logical vs physical design — what does BG actually compare?**  
**A:** **Logical:** “View Profile returns these fields.” **Physical:** normalized SQL vs embedded Mongo documents vs one fat cache key vs many small keys. BG holds the logical actions fixed and compares those implementations. **[P]**

---

## Section F — Unpredictable data & caching

**Q30. What counts as unpredictable data?**  
**A:** Data the client receives that is **stale, invalid, or inconsistent** with the **true** database state, per BG’s validation rules. It is **not** the same as an error or a timeout. A fast wrong friend list still fails the SLA. **[P]**

**Q31. Give one concrete example after Accept Friend Request.**  
**A:** Alice accepts Bob; the database is updated, but `friends:Alice` in cache still lacks Bob. The next **List Friends** returns the old list → **unpredictable**. **[P]**

**Q32. How does BG detect it?**  
**A:** Log writes as **ground truth**. For each read, compute the **acceptable values** given writes up to that time. A response outside that set is an unpredictable event. **[P]**

**Q33. Invalidation vs TTL under writes (e.g. ~10%)?**  
**A:** **Invalidation:** on write, update the DB and **delete** affected memcached keys. Reported unpredictability stays **very low** (**< 0.001%**). **TTL:** entries expire after **30, 60, or 120 seconds** with **no** immediate delete. Higher TTL lets stale keys live longer; at high thread counts unpredictability can reach about **80–90%**. **[P]**

**Q34. What is freshness confidence?**  
**A:** A characterization of how often reads stay fresh under a **weak consistency** policy such as TTL. It compares TTL policies; it is not only a binary pass/fail. **[P]**

**Q35. What is Gumball?**  
**A:** A USC technique (Ghandeharizadeh & Yap, DBSocial 2012) that prevents **races** between DB commit and cache invalidation — for example a read refilling a key in the window after commit and before delete. Cited as a way to push unpredictability toward **zero**. **[P]**

---

## Section G — Evaluation, related work & limitations

**Q36. What is the main experimental conclusion?**  
**A:** **No system wins every action.** Leadership rotates with the action, image size, and write fraction (**0.1%, 1%, 10%**). On **combined / mixed** workloads, **CASQL with invalidation** often has the **highest SoAR**. Cache key design and the **1 MB** limit can erase that win. **[P]**

**Q37. What do the image-size results say about View Profile?**  
**A:** At **2 KB**, all three systems can meet an aggressive SLA at meaningful SoAR. At **12 KB**, tradeoffs show up. At **500 KB**, **SQL-X** may fail the **100 ms p95** entirely. **Payload size is the outcome**, not a footnote. **[P]**

**Q38. What happens as BGClients or threads increase?**  
**A:** Throughput rises (**1, 2, 4, 8, 16, …** clients) until a **cliff**: tail latency or unpredictability violates the SLA. Past that point the rating stops increasing. **[P]**

**Q39. What can BG say about CAP / CP vs AP?**  
**A:** Under a **network partition**, **CP** sacrifices availability to avoid wrong answers; **AP** stays available and may return **stale** state. BG is built so failure behavior can be scored. **Unpredictable data** also covers everyday **cache staleness with no partition** — the common CASQL case. **[P]**

**Q40. Limitations you should state out loud.**  
**A:** Results depend on the **implementation** (schema, embedding, cache keys). **Eleven actions** are not all of Facebook (no search, ads, or DMs). SLA numbers (**100 ms, 95%**) are **parameters** — change them and the winner can change. The dataset is **synthetic**, for privacy and reproducibility. **[P]**

---

## Section H — Short “define in one line” (rapid fire)

| Term | Answer |
|------|--------|
| SNA | One logical user operation (e.g. View Profile) |
| SoAR | Max **actions/sec** while the SLA holds |
| Socialites | Max **concurrent threads** while the SLA holds (closed model) |
| Unpredictable data | Stale, invalid, or inconsistent read vs ground truth |
| SQL-X | RDBMS-only implementation of the schema and actions |
| CASQL | SQL-X + memcached; DB is still the source of truth |
| BGCoord | Coordinator: split work, aggregate metrics, report ratings |
| BGClient | Worker process whose threads act as socialites |
| Invalidation | Delete affected cache keys on write |
| TTL | Expire cache entries after N seconds; no immediate delete |
| D-Zipfian | Skewed access so a few members are hot |
| Gumball | Race prevention between commit and cache invalidation |
| Closed model | One outstanding request per thread; wait, then next |
| Freshness confidence | How trustworthy cached reads are under weak consistency |

---

## Section I — “Explain to the class” (2–3 minute prompts)

**I1.** Draw **BGCoord → BGClients → SQL-X / CASQL / MongoDB** and say what the **validation** step checks.

**I2.** State an example **SLA** (95% / 100 ms / unpredictability cap / 10 min) and define **SoAR** vs **Socialites**.

**I3.** Walk **Accept Friend Request** on **SQL-X**, **MongoDB**, and **CASQL**, and give one way the next **List Friends** becomes **unpredictable**.

**I4.** Explain **invalidation vs TTL** under a **~10% write** mix: freshness, write cost, and the unpredictability numbers.

**I5.** Explain the **List Friends + 1 MB** failure: 1000 friends × 2 KB, idle memcached, SoAR collapse — and why **500 KB** thumbnails can zero out **View Profile** on SQL-X.

**I6.** Close with the headline and the limits: **no system wins every action**; **CASQL + invalidation** often wins **mixed** workloads; winners move if you change image size, cache keys, or the SLA.
