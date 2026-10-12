# HRPS — Example questions and answers

Use for **closed-book practice** and **oral explanations**. Answers are concise; expand with the toy example and the Gamma numbers in [`HRPS_Study_Guide.md`](HRPS_Study_Guide.md) when presenting.

**Paper:** Shahram Ghandeharizadeh and David J. DeWitt. *Hybrid-Range Partitioning Strategy.* **VLDB 1990.** **[P]**  
**Lecture:** CSCI 585 deck `HRPS.pdf`. **[L]**

---

## Section A — Motivation and the three old strategies

**Q1. What problem does HRPS solve?**  
**A:** In a **shared-nothing** database machine, more nodes usually help a heavy query and **hurt** a light one, because startup, communication, and termination dominate. Hash and range each get one of those cases right. HRPS picks the degree of **intra-query** parallelism from the **workload** and the **hardware**. **[P]**

**Q2. Hash vs range on the partitioning attribute — exact match and range predicate.**  
**A:** **Hash:** exact match → **one** node; any range predicate → **all** nodes. **Range:** both exact match and range go only to nodes whose ranges **overlap** the predicate, often one or two nodes. **[P][L]**

**Q3. Why is hash bad for `61K < salary < 61.1K`?**  
**A:** The predicate is selective (a B+-tree would touch about one page) but hash has no locality, so the query is sent to **every** node. N−1 nodes search and find nothing. Parallelism overhead dominates. **[L]**

**Q4. Why is hash also bad when many rows share `salary = 60K`?**  
**A:** Exact match still maps to **one** node. That node’s **service time** is large, it becomes a **hot spot**, and throughput drops. **[L]**

**Q5. What is the range “tail,” using the grade picture?**  
**A:** Ranges do not wrap. Partitions `0–19, 20–39, 40–59, 60–79, 80–100`. The predicate `0 < grade < 45` and the predicate `45 < grade < 90` **both** include `40–59`. That node runs at **100%** while the ends run at **33%**. It is the bottleneck. **[L]**

**Q6. Is HRPS logical design or physical design?**  
**A:** **Physical.** Logical design chooses tables and normal forms. Physical design chooses the partitioning strategy and the indexes. **[L]**

---

## Section B — M, the formula, and fragment size

**Q7. Define N, M, and FC.**  
**A:** **N** = nodes in the machine. **M** = nodes that should **participate in one average query**. **FC = TuplesPerQ / M** = tuples in one fragment. **[P]**

**Q8. Write RT(M) and the M that minimizes it.**  
**A:** `RT(M) = (CPU + Disk + Net) / M + M × CP`. Set the derivative to zero: **`M = sqrt(work / CP)`**. CP is the overhead of one extra processor (**26 ms** in Gamma), linear because Gamma starts operators sequentially. **[P][L]**

**Q9. Why is M not “decluster the relation onto M nodes”?**  
**A:** If M = 1, the whole relation would sit on one node and every concurrent user would queue there. Range would have spread those tiny queries across all N nodes and won on throughput. So M is the parallelism of **one query**. Fragments are then round-robin’d across **all N** nodes. **[P]**

**Q10. Intra-query vs inter-query parallelism?**  
**A:** **Intra-query:** nodes used inside one user’s query (HRPS aims at about M). **Inter-query:** nodes used because several independent queries run at once (fragments occupy all N, so this can use the whole machine). **[L][P]**

**Q11. Service time, response time, throughput?**  
**A:** **μ** = time with no other load. **τ** = μ plus queuing delay. **λ** = completion rate when busy. In a closed system **λ = 1/μ**, independent of the queue. **[L]**

**Q12. Walk the paper’s toy numbers.**  
**A:** 10,000 tuples, average query reads **500**, **M = 5**. **FC = 100**. **100 fragments** (0–99, 100–199, …, 9900–9999). A 500-tuple range overlaps **5 or 6** fragments. **[P][L]**

---

## Section C — Placement: M = N, M > N, M < N

**Q13. How are fragments created and assigned?**  
**A:** **Sort** on the partitioning attribute. Cut ranges of about **FC** tuples. Assign **round-robin** so **M adjacent** fragments go to different processors when N allows it. On a node, that relation’s fragments share one **physical file**. The **range table** maps range → processor. A query hits **M or M+1** processors. **[P]**

**Q14. M = N = 5. What does each strategy do with the toy query?**  
**A:** **HRPS** overlaps 5 or 6 fragments → **all 5** nodes. **Range** has 5 fat ranges, so the query usually sits in **one** (sometimes two). **Hash** sends every range query to **all N**. **[P][L]**

**Q15. M > N (only 2 nodes). What changes?**  
**A:** HRPS still places M neighboring fragments on different nodes, so the query uses **all N**. Range, with only N fat pieces, usually uses **one** node. **[P][L]**

**Q16. M < N (10 nodes, M = 5). Response time and throughput?**  
**A:** Each HRPS query uses about **5 or 6** nodes, not 10, so it avoids hash’s extra control messages and range’s under-parallelization. Fragments still occupy **all 10** nodes, so concurrent queries spread out. **HRPS** wins both response time and multiuser throughput in this case. **[P][L]**

**Q17. What does equation (4) add, and when does HRPS look like range?**  
**A:** Directory search. Denominator becomes `CP + Cardinality × CS / TuplesPerQ`, with **CS = 0.243 ms** per entry. If that term dominates (a tiny query, or Debit-Credit), M drops and fragments get larger — HRPS **degenerates toward range**. The optimizer binary-searches the table; the experiments **size** fragments with equation (4). **[P][L]**

---

## Section D — Gamma workload

**Q18. Cluster, pages, and tables?**  
**A:** Intel iPSC/2, **24** Gamma nodes plus **8** user nodes. **8 KB** pages, **2 MB** LRU buffer, elevator scheduler. **10** Wisconsin relations, **1,000,000** tuples of **208** bytes each. About **36** tuples per page. Selections use a **clustered** index on the partitioning attribute. **[P]**

**Q19. What are the two query shapes, and why those two?**  
**A:** **0.001%** (10 tuples, **0.08 s** on one node): range should win. **10%** (100,000 tuples, **54.33 s**): hash should win. Cheap equality is treated as a special case of the tiny query. The **mix** is where they conflict, which is where HRPS must win. **[P]**

**Q20. Why measure multiuser throughput instead of only response time?**  
**A:** On a long query, hash and HRPS can have almost the same response time, because extra messages are small next to the work. Hash still **occupies every node**. Throughput under concurrency shows that waste. **[P]**

**Q21. Why does a single-node query skip the query manager?**  
**A:** It is sent **directly** to that node. A multi-node query needs a **QM**: start each node, collect “query done,” then commit or abort. Those extra messages are why fanning a 10-tuple query out to 24 nodes is so expensive. **[P][L]**

**Q22. What does the SCSI cache do, and when does it vanish?**  
**A:** **45 KB** read-ahead. Nearly **2×** on a single-user sequential scan. In a multiuser mix the next I/O is rarely sequential, and a **seek invalidates** the cache. **[P]**

---

## Section E — Numbers you should be able to say

**Q23. Tiny query: M, fragments, and the hash gap.**  
**A:** Equation (4) gives **M ≈ 0.057**, so **one** node. **FC ≈ 175**, about **5,700** fragments. At MPL 1, range and HRPS are about **8×** hash. At MPL 160, about **30×**. Range and HRPS stay within **2.5%** of each other. **[P][L]**

**Q24. Why does tiny-query throughput stop climbing linearly around MPL 12?**  
**A:** There are 24 nodes, but random predicates are not a perfect spread. Two queries can land on the same node. (Lecture illustration: 3 requests on 3 nodes have 27 assignments and only 6 are uniform.) **[P][L]**

**Q25. Heavy query: M, fragments, nodes used.**  
**A:** **M = 90**, **895** fragments. Only 24 nodes exist, so each does about **3.7×** the ideal share, and HRPS still uses **all 24**. Range overlaps **at most 3** fragments. Hash uses all 24. At MPL 1, hash and HRPS are **16%** above range. **[P][L]**

**Q26. Why doesn’t the disk cache show up on the 10-tuple query?**  
**A:** Ten tuples fit on **one** 8 KB page most of the time (36 tuples per page), two pages sometimes. There is no long sequential run for read-ahead to help. The 10% query reads a huge sequential run, so the cache matters until concurrency destroys it. **[P]**

**Q27. Equal mix at MPL 1, and why the curves later meet.**  
**A:** HRPS builds **1,688** fragments: tiny query → **1** node, heavy query → **all** nodes. At MPL 1, HRPS is **11%** above hash and **46%** above range. Past MPL about **4**, hash and HRPS converge because the **slow 10% query dominates** the average. **[P]**

**Q28. At MPL 50, how does the winner change with the mix?**  
**A:** Heavy query **under ~5%** of the mix: **range beats hash**. **5–7%:** **HRPS beats both by about 30%**. **Above ~10%:** hash and HRPS are nearly tied and both beat range. The gain over range levels off near **40%** heavy queries because the **disk is saturated**. **[P]**

---

## Section F — Skew, small relations, limits

**Q29. How does HRPS handle a small relation?**  
**A:** Fragment count depends on the workload and the hardware, **not on N**. If there are fewer fragments than nodes, the relation occupies a **subset**. That is partial declustering. **[P][L]**

**Q30. Duplicate key example.**  
**A:** 100,000 tuples, FC = 1,000, and 4,000 tuples have key **2**. After the sort, value 2 spans **4 or 5 fragments on different nodes**. Equality on 2 uses those nodes. Hash and range would put every “2” on **one** node. **[P]**

**Q31. Two limitations stated in the paper?**  
**A:** (1) The model assumes the work **can** be parallelized. (2) HRPS is **one-dimensional**. A predicate on any other attribute is sent to every node that holds a fragment. **[P]**

**Q32. How is HRPS related to Copeland’s Bubba placement (heat, temperature)?**  
**A:** They are **orthogonal**. HRPS decides **how many fragments** and how big. Copeland’s heuristics decide **which node** gets a fragment (load vs locality, disk vs memory) and can reorganize when the workload changes. Cutting fragments is not the same problem as packing them, which is bin packing. **[P]**

**Q33. Why does the paper ignore full table scans?**  
**A:** For a sequential scan, range declustering was already the best of the earlier strategies. HRPS targets **index selections** (range or equality), because the selection’s parallelism caps every later operator in a dataflow plan. **[P]**

---

## Section G — MAGIC, BERD, closed vs open

**Q34. What does the lecture say HRPS still cannot do?**  
**A:** **Heterogeneous** nodes (one RT curve does not fit every machine) and queries that filter on **different attributes**. **[L]**

**Q35. What is MAGIC?**  
**A:** The SIGMOD **1992** follow-on (Ghandeharizadeh, DeWitt, Qureshi). A **grid directory**: each axis is a partitioning attribute, each **cell** is a fragment stored on a node. The grid is shaped from the workload. **[L]**

**Q36. Range vs MAGIC on the 9-node Emp example.**  
**A:** Query A (age, ~5 tuples) and query B (salary, ~10 tuples), 50% each, each ideally on **1** node. Range on age: average **5.5** nodes (A on 1, B on all 9). MAGIC: average **3** if age and salary are weakly correlated, average **1** if they are strongly correlated. **[L]**

**Q37. What is BERD on those slides?**  
**A:** Range-partition the table on a **primary** attribute (salary). Keep an **auxiliary** relation `(age, node)` range-partitioned on age, and look up which nodes to read. Low correlation: average still **5.5**. High correlation: average **1**, so the auxiliary lookup buys less. Throughput plots: **MAGIC above BERD above range**. **[L]**

**Q38. Closed vs open — which one is the Gamma figure?**  
**A:** **Closed:** a fixed set of clients, next request only after the previous reply (optional think time). Better means higher throughput as MPL grows. The paper’s plots are closed. **Open:** arrivals at rate **λ** even if the system is busy. Better means a higher λ before response time blows up. The lecture draws hash’s knee to the left of hybrid-range’s. **[L][P]**

---

## Section H — Short “define in one line” (rapid fire)

| Term | Answer |
|------|--------|
| HRPS | Workload-sized **range fragments**, round-robin across the nodes |
| M | Desired **nodes per average query** |
| N | Nodes in the machine |
| FC | **TuplesPerQ / M** |
| CP | Overhead per extra node (**26 ms** in Gamma) |
| CS | Range-table search per entry (**0.243 ms**) |
| Range table | Fragment range → processor |
| Intra-query | Nodes inside **one** query |
| Inter-query | Nodes used by **concurrent** queries |
| μ, τ, λ | Service time; response time; throughput (λ = 1/μ when closed and busy) |
| Clustered index | Heap order matches the index, so a range is sequential I/O |
| MAGIC | Multi-attribute **grid** of fragments (SIGMOD 1992) |
| BERD | Primary range partition plus an auxiliary attribute→node relation |
| Closed model | Fixed clients; score by throughput vs MPL |

---

## Section I — “Explain to the class” (2–3 minute prompts)

**I1.** Draw hash, range, and HRPS for `salary = 60K` and for a **small** salary range. Say which nodes wake up and why.

**I2.** Derive **M = sqrt(work / CP)** from RT(M). Then explain, with the M = 1 counterexample, why the relation is **not** stored on only M nodes.

**I3.** Use the 10,000-tuple toy (M = 5, FC = 100) and walk **M = N**, **M > N**, and **M < N**. End with who wins response time and who wins throughput when M < N.

**I4.** Tell the Gamma story in three beats: tiny query (**8×** and **30×** vs hash), heavy query (**16%** vs range, M = 90), and the mix (**+11% / +46%**, then the **5–7%** band where HRPS beats both).

**I5.** Explain the grade tail (node 3 at 100%) and the duplicate-key example (value 2 on 4 nodes) as two different imbalance problems.

**I6.** Close with the one-dimensional limit and MAGIC: 5.5 vs 3 vs 1 on the age/salary grid, and name BERD as the auxiliary-relation alternative.
