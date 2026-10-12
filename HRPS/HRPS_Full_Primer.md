# HRPS — Complete Primer (zero background assumed)

**Paper:** *Hybrid-Range Partitioning Strategy: A New Declustering Strategy for Multiprocessor Database Machines*  
**Authors:** Shahram Ghandeharizadeh, David J. DeWitt (Wisconsin) · **VLDB 1990**  
**Lecture:** CSCI 585, `[HRPS.pdf](HRPS.pdf)`

**How to use this doc:**


| If you want…               | Read…                          |
| -------------------------- | ------------------------------ |
| Full teaching from scratch | This file, Parts 0–12 in order |
| Shorter reference          | `HRPS_Study_Guide.md`          |
| Drill                      | `HRPS_Flashcards.md`           |
| Mock exam / oral answers   | `HRPS_Exam_QA.md`              |
| Slide-by-slide pass        | `HRPS_Lecture_Slides.md`       |


Tags: **[P]** = paper, **[L]** = lecture.

---



# Part 0 — What is this paper?



### Q: What problem is it solving?

**A:** You have a database spread across many computers, each with its own disk. A query runs on the computers that hold the rows it needs. Use too many computers for a tiny query and the messages cost more than the query. Use too few for a huge query and one computer becomes a traffic jam. **HRPS** decides **how finely to slice the table** so each query gets about the right number of computers. **[P]**

### Q: What is HRPS not?

- Not a new SQL language.
- Not a new index (it assumes you already have a clustered index for the experiments).
- Not “always use every machine.”

It is a **physical design** choice: how to **partition** a table. Logical design is “what tables, what columns.” Physical design is “how to split the rows, and which indexes.” **[L]**

### Q: One sentence for an oral exam?

**A:** HRPS cuts a relation into many small ranges whose size comes from the workload, so a cheap query runs on one node and an expensive query runs on about M nodes, while the pieces still cover the whole machine for concurrent users. **[P]**

---



# Part 1 — The machine



### Q: What is shared-nothing?

**A:** Each node has private CPU, memory, and disk. Nodes talk by messages. There is no shared disk farm and no shared RAM. Stonebraker’s argument, cited in the paper: this is how you scale a database machine. **[P]**

Picture N boxes. Node i stores some of the rows and runs the operators for those rows.

### Q: What is declustering?

**A:** Horizontal partitioning. The rows of one table are divided across nodes. The lecture also says **sharding**. Same idea. **[L]**

If there is **no** partitioning column, you can place rows **randomly** or **round-robin** (row 1 on node 0, row 2 on node 1, …). If there **is** a column, the two classic choices are **hash** and **range**. The DBA picks the column. **[L][P]**

### Q: What is Gamma?

**A:** The Wisconsin database machine the authors already had. It runs on a **32-node Intel hypercube**. Each node is a 386 with 8 MB of RAM and a 330 MB disk. The experiments use **24** nodes for the database and **8** nodes to pretend to be users. **[P]**

Gamma already implemented round-robin, hash, and range. HRPS is added and compared with hash and range. **[P]**

---



# Part 2 — Three old strategies, with a salary example

Table `Emp(salary, …)`. Partitioning attribute = `salary`. **[L]**

### Round-robin

Rows are dealt like cards. A query cannot know which node holds `salary = 60K`, so **every** query goes to **every** node. **[P]**

### Hash

`node = hash(salary) mod N`.  

- `WHERE salary = 60K` → **one** node. Good.
- `WHERE salary > 61K AND salary < 61.1K` → **every** node. The hash mixes neighboring salaries. A B+-tree would have touched one page, but you still wake up N−1 nodes that find nothing. **[L][P]**

Also bad the other way: if **many** people earn exactly 60K, the exact-match still hits **one** node, and that node does a huge amount of work. Hotspot. **[L]**

### Range

Give each node a span: node 0 holds 0–20K, node 1 holds 20K–40K, and so on.  

- A small range and an equality hit **one or two** nodes. Good for tiny queries.
- A query that should use many disks often still hits **one or two** fat ranges. Too little parallelism. Response time stays high, and that node becomes the bottleneck. **[P][L]**



### Q: What is the “tail” of range?

**A:** Ranges do not wrap. Grades split as 0–19, 20–39, 40–59, 60–79, 80–100. **[L]**

- `0 < grade < 45` uses the first three buckets.
- `45 < grade < 90` uses the middle three.

The middle bucket (40–59) is in **both** queries. It runs at 100% while the end buckets run at 33%. The middle node caps throughput. **[L]**

---



# Part 3 — Parallelism has a cost



### Q: Intra-query vs inter-query?

**A:** **[L]**

- **Intra-query:** how many nodes work on **one** person’s query. “This salary report used 3 nodes.”
- **Inter-query:** how many nodes are busy because **several people** submitted queries at once. Those queries do not have to be pieces of the same SQL statement.

HRPS sets intra-query parallelism to about **M**, and still spreads fragments across all **N** nodes so inter-query parallelism can use the whole machine. **[P]**

### Q: Service time, response time, throughput?

**A:** **[L]**

- **Service time μ:** how long the query takes if it is the only one.
- **Response time τ:** μ plus the wait behind other queries.
- **Throughput λ:** completions per second when the system is busy. In a closed system **λ = 1/μ**. Queuing does not change this definition; it changes response time.

Putting a heavy query on one node raises μ and creates a hotspot, so λ falls. **[L]**

### Q: Why does a tiny query get slower if you add nodes?

**A:** The lecture’s first graph: service time **climbs** as the node count climbs. You pay startup, a message to each node, and termination. If the real work was one disk page, the messages are the whole query. **[L][P]**

### Q: Why does a huge query get faster, then stop getting faster?

**A:** Second graph: service time **drops**, bottoms out, then edges up. The bottom is **M**, the degree of parallelism you wanted. Past that, extra nodes cost more than they save. **[L]**

---



# Part 4 — The formula, slowly



### Q: What is the average query?

**A:** The workload is several query types, each with a frequency. HRPS does not pick a fragment size per query. It builds **one** average: CPU, disk, and network time, each weighted by frequency, and an average number of tuples touched (`TuplesPerQ`). **[P]**

### Q: Why add CPU + disk + network?

**A:** The model pretends one query uses only one of those at a time on a node. Gamma can overlap disk and CPU with read-ahead, so this is a simplification. The authors’ excuse: under many concurrent queries, while your query holds the CPU, someone else’s query is probably on the disk. **[P]**

### Q: Write RT(M).

**A:** **[P][L]**

```text
RT(M) = (CPU + Disk + Net) / M  +  M × CP
```

First term: the real work, split across M nodes.  
Second term: overhead grows **linearly** with M. In Gamma that is true because operators are started and stopped one after another. A tree-shaped startup would have been logarithmic. They measured **CP = 26 ms** per extra node. **[P]**

### Q: Where does M = sqrt(work / CP) come from?

**A:** Take dRT/dM and set it to zero.

```text
dRT/dM = -(CPU+Disk+Net)/M^2  +  CP  =  0
M^2 = (CPU+Disk+Net) / CP
M = sqrt( (CPU+Disk+Net) / CP )
```

The lecture says “M is the first derivative.” What they mean is: **set the derivative to zero and solve**. M is the minimizer, not the derivative itself. **[L][P]**

### Q: Why is M not “store the table on M nodes”?

**A:** Suppose the average query is tiny and the formula says M = 1. If you then store the **entire** table on 1 node, every concurrent user queues on that node. Range partitioning, with one big range per node, would have sent those users to **different** nodes. Throughput would be worse under HRPS, which is the opposite of the goal. **[P]**

So they change the meaning:

- **M** = how many nodes should **run this query**.
- The table is still spread over **all N** nodes.
- Each piece (fragment) is small enough that one query covers about M pieces.

```text
FC = TuplesPerQ / M
```

FC is “how many tuples one of those M nodes should see.” **[P]**

---



# Part 5 — A numeric toy, then the real tiny query



### Toy (paper Figure 1, lecture “Example”)

10,000 employees, ids 0 through 9,999. Average query reads **500** rows. Formula says **M = 5**. **[P][L]**

```text
FC = 500 / 5 = 100
fragments = 10,000 / 100 = 100
```

Sort by id. First fragment is 0–99, next is 100–199, and so on, through 9900–9999. Deal the 100 fragments round-robin onto the nodes so that **5 fragments in a row sit on 5 different nodes**.

A query for 500 consecutive ids covers **5 or 6** fragments, hence 5 or 6 nodes.

**M = N = 5.** HRPS uses all 5 nodes. Range (five fat chunks of 2,000 ids) usually uses **1**, sometimes 2. Hash uses all 5 for any range.

**M > N.** Only 2 nodes, but M is still 5. Each node holds many small fragments (0–99 and 200–299 on the same node, because round-robin wrapped). The 5-fragment query still touches **both** nodes. Range, with only 2 fat chunks, usually touches **one**.

**M < N.** 10 nodes, M = 5. HRPS still has 100 small fragments, spread over all 10 nodes, but one query only overlaps 5 or 6 of them. Hash wakes all 10. Range wakes 1 or 2.

For M < N: HRPS has the better **response time** (right number of nodes per query) and the better **throughput** (fragments occupy every node, so other users’ queries have somewhere to go, without hash’s extra messages). **[P][L]**

### The real tiny query, and why M comes out less than 1

Gamma, one million tuples, query reads **10** tuples, single-node time **80 ms**. **[P]**

Directory search matters here. Equation (4):

```text
M = sqrt( work / (CP + Cardinality × CS / TuplesPerQ) )
```

Plug in CP = 26 ms, CS = 0.243 ms per directory entry:

```text
denom = 26 + (1,000,000 × 0.243) / 10 = 24,326
M = sqrt(80 / 24,326) ≈ 0.057
```

Use **1** node. Fragment size FC = 10 / 0.057 ≈ **175** tuples. Number of fragments ≈ **5,700**, scattered round-robin. Most 10-tuple ranges sit on one fragment. **[P][L]**

If the query is as small as Debit-Credit, searching a huge directory would dominate, so the same math makes fragments **larger** and HRPS starts to look like ordinary range. **[P]**

---



# Part 6 — What happens at runtime



### Single-node query

The benchmark process sends the query **directly** to the one node. Few messages. **[P][L]**

### Multi-node query

The query goes to a **Query Manager**. The QM starts every participating node, waits for “query done” from each, then sends commit (or abort if someone deadlocks). Then it tells the user. Many more messages. That difference is why hash looks so bad on the 10-tuple query. **[P]**

The optimizer binary-searches the range table to decide the node list. They still **size** the fragments with the linear-search formula (equation 4) and the measured 0.243 ms per entry. **[P]**

---



# Part 7 — The experiments, as a story

Ten tables, a million rows each, 208 bytes per row, Wisconsin benchmark. Everyone uses a **clustered** B+-tree on the partitioning column. **[P]**

A clustered index means the rows on disk are in the same order as the index. Finding the two endpoints is random I/O. Reading the rows in between is sequential I/O. Each 8 KB page holds about **36** rows. Ten rows ≈ one page. A hundred thousand rows ≈ a long sequential read. **[P]**

The disk has a small read-ahead cache (45 KB). It helps a **single** user scanning sequentially (almost 2×). It does little when many users cause seeks, because a seek throws the cache away. **[P]**

### Only tiny queries

Hash fans out to 24 nodes and drowns in control messages.

- At 1 user, range and HRPS are about **8×** hash.
- At 160 users, about **30×**.
- Range and HRPS stay within **2.5%** of each other. A 5,700-entry directory is cheap next to the query.
- Throughput of range/HRPS bends after about **12** users, even though there are 24 nodes. Random ranges collide on the same node. The lecture’s side picture: 3 requests on 3 nodes can be assigned **27** ways, and only **6** of those are perfectly even. **[P][L]**



### Only heavy queries (10% of the table, 100,000 rows, 54.33 s on one node)

The formula wants **M = 90**. Only 24 nodes exist, so HRPS uses all 24, and each does about **3.7×** the ideal share. It builds **895** fragments. Range uses **at most 3** nodes. Hash uses 24. **[P]**

At one user, hash and HRPS are **16%** faster than range. This is the best day for hash and the worst day for range. HRPS ties hash. The SCSI cache makes the curves wobble between 1 and 3 users; the 10-row query never reads enough sequential pages for that wobble to appear. **[P]**

### Both at once

This is the point of the paper. The tiny query wants 1 node. The heavy query wants every node. **[P]**

HRPS builds **1,688** fragments and can do **both**. At one user it is **11%** above hash and **46%** above range. As you add users, hash catches up, because the slow query dominates the average.

Hold the user count at **50** and change the mix:

- Heavy query **under 5%** of the mix → range beats hash (almost everything is tiny).
- Heavy query **5% to 7%** → HRPS beats **both** by about **30%**.
- Heavy query **above ~10%** → hash and HRPS are nearly tied, both well ahead of range, until the disk saturates (around 40% heavy).

**[P]**

---



# Part 8 — Two more things HRPS gets right



### Small tables

Fragment count does not depend on how many nodes you own. A small table produces fewer fragments than nodes, so it occupies a **subset**. On a 1,000-node machine you do not sprinkle a 200-row table onto all 1,000 nodes. **[P][L]**

Packing many small tables onto disks is bin packing. A simple rule: next fragment goes to the disk with the most free space that does not already have a piece of this table. Hot fragments (**temperature**, from the Bubba work) should influence that choice. **[P]**

### Skew and duplicates

Fragments are cut by **row count**, after sorting, not by equal widths of the key. If 4,000 rows share the key value 2 and FC is 1,000, that value spans 4 or 5 fragments on **different** nodes. Equality on 2 uses 4 nodes. Hash and range would have piled all of those rows on **one** node. **[P][L]**

---



# Part 9 — What HRPS does not solve

1. Some work cannot be split. The formula assumes it can. **[P]**
2. Only **one** column. A predicate on a different column still visits every node that stores a fragment. **[P]**
3. Heterogeneous nodes (some fast, some slow) would need more than one curve. One M does not fit. That one is the lecture, not a measured Gamma result. **[L]**

Copeland’s heat/temperature placement is a **different** problem. HRPS decides the fragments. Bubba-style heuristics can assign those fragments to nodes and can rebalance later. The paper calls the two ideas orthogonal. **[P]**

---



# Part 10 — MAGIC, because the lecture spends 15 slides on it

HRPS is one-dimensional. **MAGIC** (SIGMOD 1992, Ghandeharizadeh, DeWitt, Qureshi) partitions on **several** columns using a **grid**. Each axis is an attribute. Each cell is a fragment and stores a node id. **[L]**

Example, 9 nodes, `Emp(age, salary)`: **[L]**

- Query A: a tight age range, ~5 rows, should use 1 node.
- Query B: a tight salary range, ~10 rows, should use 1 node.

Range-partition on age only: A uses 1 node, B uses all 9. Average = **5.5** nodes. Ideal average = **1**.

MAGIC’s grid:

- **Low** correlation between age and salary: average **3** nodes (better than 5.5, not yet 1).
- **High** correlation: average **1** node. The two predicates fall in the same cells.

**BERD** in the lecture: pick salary as the primary range partition, and keep a second little table of `(age → node)`, itself range-partitioned on age. You look up ages, then fetch those employees.

- Low correlation: BERD still averages **5.5**, same as plain range.
- High correlation: BERD averages **1**, because salary order already matches age order. The lecture asks if you can skip the auxiliary lookup; high correlation is when that lookup matters less.

On the throughput plots (low/low mix), **MAGIC > BERD > range**, and the advantage is larger when the attributes are highly correlated. **[L]**

You do not need the SIGMOD paper’s full algorithm for this course. You need: MAGIC is the multi-attribute sequel, a grid, workload-shaped, and it beats one-attribute range when queries hit different columns. **[L]**

---



# Part 11 — Closed and open, in the last three slides

**Closed:** a fixed number of clients. Each waits for an answer (and maybe a think time) before the next request. Better means **higher throughput** as you add clients. The Gamma graphs are this experiment. Hybrid-range sits above range. **[L][P]**

**Open:** new requests arrive at rate λ even if old ones are unfinished. Better means you can push λ higher **before** a queue blows up the response time. The slide shows hash’s curve turning upward sooner than hybrid-range’s. **[L]**

---



# Part 12 — If you only remember ten facts

1. Shared-nothing: the operator runs where the rows live. **[P]**
2. Hash: equality → 1 node, range → all nodes. **[P]**
3. Range: small queries stay local; big queries stay too local; the middle range is a bottleneck. **[P][L]**
4. M minimizes RT(M) = work/M + M×CP. **[P]**
5. M is nodes **per query**, not nodes that store the table. **[P]**
6. FC = TuplesPerQ / M. Sort, cut, round-robin. **[P]**
7. M = N uses all nodes; M > N still uses all N; M < N uses about M nodes per query but occupies all N. **[P]**
8. Tiny Gamma query: M ≈ 0.057, 5,700 fragments, ~8× and ~30× over hash. **[P]**
9. Heavy query: M = 90, 895 fragments, ties hash, beats range by 16% at one user. Mix at 5–7% heavy: HRPS beats both by ~30%. **[P]**
10. One-dimensional limit → MAGIC’s grid. **[P][L]**

---

*Pair with* `Hybrid-Range.PDF` *for Figures 1 and 4–9, and with* `HRPS.pdf` *for the grade diagram, the grid, and the closed/open sketches.*