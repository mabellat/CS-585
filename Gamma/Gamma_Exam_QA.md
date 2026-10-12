# Gamma — Example questions and answers

Use for **closed-book practice** and **oral explanations**. Expand with numbers and diagrams from [`Gamma_Study_Guide.md`](Gamma_Study_Guide.md).

**Paper:** DeWitt et al., *The Gamma Database Machine Project*, **IEEE TKDE** 1990. **[P]**  
**Lecture:** CSCI 585 [`GammaLecture.pdf`](GammaLecture.pdf). **[L]**

---

## Section A — Motivation and legacy

**Q1. Why build Gamma after DIRECT?**  
**A:** DIRECT showed parallel DBMS ideas could work but hit **shared-memory** limits, **centralized control**, and used parallelism where **indexes and I/O** mattered more. Gamma adopts **shared-nothing**, **hash-parallel** algorithms, and **dataflow** with minimal control traffic. **[P]**

**Q2. State Gamma’s three key technical ideas.**  
**A:** (1) **Horizontal declustering** across disks, (2) **hash-based** parallel join and aggregate, (3) **dataflow** execution with decentralized tuple routing. **[P]**

**Q3. How is Gamma relevant to modern warehouses?**  
**A:** Users write **SQL**; the system parallelizes transparently—same user model as **Redshift** / **BigQuery** (lecture). Influential enough for **ACM Software System Award 2008**. **[L]**

**Q4. What is shared-nothing and why use it?**  
**A:** Each node has private CPU, memory, and disk; coordination is by **messages**. Argued to scale to **hundreds/thousands** of processors without shared-disk bottlenecks. **[P][R3]**

**Q5. Speedup vs scaleup—give an example of each from the lecture.**  
**A:** **Speedup:** “give all employees 10% raise” faster with more nodes. **Scaleup:** keep average-salary-style query time as DB grows 10× by growing nodes ~10×. **[L]**

---

## Section B — Declustering and catalog

**Q6. Name the three declustering strategies and who picks the partition attribute.**  
**A:** **Round-robin**, **hash**, **range**. The **administrator** selects the partitioning attribute (lecture). Strategy is in the **catalog** for optimizer **node pruning**. **[P][L]**

**Q7. Hash declustering: `salary = 60K` vs `salary < 50K` when salary is the partition key.**  
**A:** **Equality** → **one** node (`hash(salary) mod N`). **Range** → **all** nodes because hash destroys order/locality. **[P][L]**

**Q8. Why store query results round-robin?**  
**A:** So each fragment of a **result relation** has approximately the **same number of tuples**, keeping later parallel operators balanced. **[P]**

**Q9. Where are indexes built in Gamma?**  
**A:** **Per fragment** on each node (clustered or non-clustered). Clustered index **does not have to** be on the partitioning attribute. **[P]**

**Q10. What placement mistake do the authors admit?**  
**A:** Declustering **every** relation across **all** disk nodes regardless of “heat”; **partial** declustering (e.g. Bubba) is better for some relations. **[P][R6]**

---

## Section C — Processes and query path

**Q11. List the main server processes and one sentence each.**  
**A:** **Catalog Manager** — schemas; **Query Manager** — per-user compile/optimize; **Scheduler** — per multi-site query, activates operators; **operators** on nodes; **deadlock detector**; **recovery/log** managers. **[P]**

**Q12. When does a query bypass the scheduler?**  
**A:** **Single-site** plans: QM sends work **directly** to the one participating processor. **[P][L]**

**Q13. Why primed operator processes?**  
**A:** Process creation per query was too expensive; a pool is started at **initialization** and receives assignments on a **new-task** port. **[P]**

**Q14. What is gdl?**  
**A:** Gamma’s **ad-hoc query language** (Quel variant); also **embedded SQL** in C. **[P]**

---

## Section D — Dataflow, split table, control traffic

**Q15. Explain the split table.**  
**A:** Each operator reads an input **tuple stream**, processes tuples, and uses a **split table** (demux) to send each output tuple to the correct **destination processor and port** without scheduler involvement. **[P][L]**

**Q16. What happens at end-of-stream for an operator?**  
**A:** Close output streams (sends **end-of-stream** downstream), then send **control** to scheduler that the operator finished. **[P][L]**

**Q17. Compare Gamma control traffic to DIRECT.**  
**A:** DIRECT needed control per **data page**. Gamma needs about **three control messages per operator per participating processor** for the query. **[P][R1]**

**Q18. Does the scheduler route tuples?**  
**A:** **No.** Scheduler exchanges **control** only; **data** flows operator-to-operator per compiled plan. **[L]**

**Q19. Typical network batch size for tuples?**  
**A:** About **8 KB** per batch (except last). **[P]**

---

## Section E — Joins and aggregates

**Q20. What join algorithms did Gamma implement and which became default?**  
**A:** Sort-merge, Grace, simple hash, **hybrid hash** — **hybrid hash default** after performance study. **[P][R4]**

**Q21. Describe hybrid hash-join in the parallel setting.**  
**A:** Relations partitioned by **join-key hash**; **split tables** ship tuples to join processors (may be **diskless**); build/probe phases; **first buckets** can remain in memory; results stored (e.g. round-robin) on data nodes. **[P][L]**

**Q22. Build vs probe—are they concurrent on one node?**  
**A:** On a **single** node, **sequential**: build completes, then probe (lecture). Parallelism is across **nodes** and **pipelines**. **[L]**

**Q23. If build relation does not fit in RAM (lecture numbers)?**  
**A:** Partition inner and outer into **M buckets** with the same hash; for each i, load bucket i of inner into memory and probe with bucket i of outer; optional in-memory **first bucket** without writing. **[L]**

**Q24. Parallel aggregate pattern?**  
**A:** Compute **partial aggregates** locally → **redistribute** by hash on GROUP BY columns → finalize per partition. **[P]**

**Q25. Update that changes the partitioning attribute?**  
**A:** Tuple routed through **split table** to the node that owns the new key—not updated only locally. **[P]**

---

## Section F — Transactions, availability, evaluation

**Q26. Concurrency and deadlock in Gamma?**  
**A:** **Two-phase locking** (file/page, multiple lock modes). **Local** wait-for graphs; **central** detector periodically merges and aborts transaction with **fewest locks**. **[P]**

**Q27. Chained vs interleaved declustering—one contrast.**  
**A:** **Chained:** backup on **next** disk in ring; failover without **moving data**; ~1/(M−1) load shift per survivor. **Interleaved:** backup spread; **two** failures can break availability; Teradata favors **small** clusters. **[P][R5][L]**

**Q28. Why might selection speedup be superlinear?**  
**A:** With one disk, relation occupies **most** of the drive (long **seeks**); declustered over 30 disks each holds **~2%** — seek time ∝ **√ distance**. Multi-zone disks add bandwidth. **[P][L]**

**Q29. Why were benchmark results stored in the database?**  
**A:** To avoid **host CPU** and **host–machine link** becoming the measured bottleneck. **[P]**

**Q30. Indexed 1% selection: clustered index on 1M vs 10M tuples—what stays similar?**  
**A:** When **same number of tuples** are retrieved (e.g. 100K rows), clustered-index time is **similar**—I/O/CPU track **output size**, not only table size. **[P]**

**Q31. Single-tuple select on partitioning attribute—how many nodes?**  
**A:** **One** node; time can rise when B-tree **height** increases (e.g. 1M vs 10M tuples). **[P]**

---

## Section G — Lecture scenarios (oral)

**Q32. Walk through pushing selection before join for Toy department.**  
**A:** Instead of joining full Dept with Emp then σ on `dname=Toy`, **scan Dept**, σ to Toy rows, **build** hash on `dno`, **probe** with Emp—join and network carry fewer tuples. **[L]**

**Q33. What are virtual nodes for?**  
**A:** Map one multicore machine to **multiple logical** shared-nothing nodes (split CPU, memory, disk). **[L]**

**Q34. Recite the slide-132 contribution list (abbreviated).**  
**A:** HRPS/MAGIC declustering (next course topic), **hybrid hash-join**, **chained declustering**, **dataflow**, **split operator**, disk and **diskless** nodes, **speedup/scaleup** methodology, **SQL hides parallelism**. **[L]**

**Q35. VAX vs iPSC/2—one hardware lesson each.**  
**A:** VAX: **network/bus** bottleneck, **2 KB pages** too small. iPSC/2: **32** nodes, **8 MB** RAM, hypercube messaging, **NOSE** threads, porting **0-based** vs **1-based** node IDs caused bugs. **[P]**

---

## Section H — Compare to HRPS (bridge question)

**Q36. How does HRPS relate to Gamma?**  
**A:** HRPS is an **additional declustering strategy** on the **same Gamma machine**; it tunes **fragment count** from workload so small queries use ~**M** nodes and fragments still spread across **N** for concurrency. Gamma paper covers **round-robin/hash/range** and the **execution engine**; HRPS paper focuses on **physical placement**. **[L][HRPS]**

**Q37. Same hash predicate behavior in Gamma and HRPS lectures?**  
**A:** Yes: hash + **equality** on partition key → **one** node; hash + **range** → **all** nodes. **[L]**
