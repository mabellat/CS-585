# BG Paper — Complete Primer (zero background assumed)

**Paper:** *BG: A Benchmark to Evaluate Interactive Social Networking Actions*  
**Authors:** Sumita Barahmand, Shahram Ghandeharizadeh (USC) · **CIDR 2013**  
**Site:** [bgbenchmark.org](http://bgbenchmark.org)

**You asked for:** every little detail, basic questions answered, context + examples—not just definitions.

**How to use this doc:**

| If you want… | Read… |
|--------------|--------|
| Full teaching from scratch | This file, Parts 0–18 in order |
| Shorter reference | `BG_Paper_Study_Guide.md` |
| Drill | `BG_Paper_Flashcards.md` |
| Mock exam / oral answers | `BG_Paper_Exam_QA.md` |
| Quick lecture pass | `BG_Paper_Lecture_Slides.md` |

---

# Part 0 — What is this paper, and why should you care?

### Q: What is the paper trying to solve?

**A:** Companies run social networks on **databases + caches + app servers**. They argue:

- “Use MongoDB, it’s faster.”
- “Use PostgreSQL + memcached.”
- “Use our cloud DB.”

Without a **standard test**, those claims are meaningless. **BG** is a **benchmark**: a repeatable experiment that stresses a system the way a **real social site** does and scores it with rules users actually care about (speed **and** not showing wrong friend lists).

### Q: What is BG *not*?

- **Not** a social network product (not Facebook).
- **Not** a database (not PostgreSQL).
- **Not** a graph database benchmark only—it tests **actions** (view profile, etc.) on whatever store you plug in.

### Q: Why is it called “BG”?

The title is **“BG: A Benchmark to Evaluate…”** — **BG** is the **short name** of the project (like YCSB). In USC DB Lab tradition, people often say **B** = **Barahmand**, **G** = **Ghandeharizadeh**. The paper does not always spell that out; exams usually care about **what BG measures**, not the acronym.

### Q: Who are the authors and why does CS 585 use this?

Same **USC Database Laboratory** as your course instructor’s research line. The paper is a **systems + databases** contribution: how to **measure** social workloads, including **cache staleness**—a topic lecture often skips.

---

# Part 1 — The real-world problem (context)

### Q: What is a “social networking site” in systems terms?

**A:** A service where:

1. **State** lives in a **data store** (users, friends, posts).
2. **Apps** issue **small operations** (read profile, post, accept friend).
3. **Millions of users** hit it at once → **concurrency**, **caches**, **replicas**.

Examples: Facebook, LinkedIn, X/Twitter, Instagram (each differs, but the pattern holds).

### Q: What does “interactive” mean here?

A human clicks and **waits** for the screen to update. If “View Profile” takes 3 seconds, the product feels broken. BG models **latency-sensitive** work, not overnight batch jobs.

### Q: What is “small touch of big data”?

The **database** might have **100 million users**, but **one action** touches:

- **One** profile, or
- **One** friend list (maybe hundreds of rows), or
- **One** new post.

You never scan the whole graph in one user click. BG matches that: **huge dataset**, **tiny per-action footprint**.

### Q: Why don’t TPC-C or YCSB cut it?

| Benchmark | Thinks the world is… | Social gap |
|-----------|----------------------|------------|
| **TPC-C** | Warehouses, orders, inventory | No friends, walls, thumbnails |
| **YCSB** | Generic key-value rows | No friend-request semantics; no “wrong read” in SLA |
| **OO7** | 1990s object DB | Outdated |

**BG** = domain-specific: **social actions**, **SLA**, **unpredictable reads** when you use caches.

---

# Part 2 — Concepts you need before the paper makes sense

Read this part once slowly. Everything in BG builds on it.

---

## 2.1 What is a data store?

**Data store** = software that **persists** and **retrieves** data: PostgreSQL, MySQL, MongoDB, Cassandra, etc.

**Application** sends: “Get user 42’s profile.”  
**Store** returns bytes or rows.

BG compares **different stores** running the **same logical actions**.

---

## 2.2 What is a benchmark?

A **benchmark** is:

1. A **fixed workload** (what requests to send).
2. A **fixed dataset** (how big, what shape).
3. **Rules** for measurement (what “good” means).
4. **Repeatable** runs so results are comparable.

**Example:** Car EPA mileage test—not “drive however you want,” but a defined route and fuel measurement.

BG’s “route” = **11 social actions** + **SLA**.

---

## 2.3 SQL (relational) vs document (NoSQL) — intuition

### SQL / RDBMS (what BG calls **SQL-X**)

- Data in **tables** with **rows** and **columns**.
- **Relationships** via keys: `Friends.userid1`, `Friends.userid2`.
- **Joins** combine tables: “friends of user 5” = join `Members` and `Friends`.
- **Transactions**: “accept friend” can be **all-or-nothing** (ACID).

**Example row:**

| userid | firstname | thumbnail |
|--------|-----------|-----------|
| 5 | Alice | (2 KB blob) |

### Document store (**MongoDB** in the paper)

- Data often stored as **JSON-like documents**.
- Can **embed** lists inside a user document:

```json
{
  "userid": 5,
  "firstname": "Alice",
  "confirmedFriends": [7, 12, 99],
  "pendingFriends": [3]
}
```

- **One read** can get profile + friend ids; **List Friends** might need **many reads** (one per friend’s doc).

**Tradeoff:** fewer joins, more duplication, different update patterns.

---

## 2.4 ACID (what SQL people mean by “strong”)

| Letter | Meaning | Friend-request example |
|--------|---------|------------------------|
| **A**tomicity | All steps or none | Accept friend: update status **and** both sides’ lists—or rollback |
| **C**onsistency | Rules hold (constraints) | Can’t have “confirmed” without two users |
| **I**solation | Concurrent txs don’t corrupt each other | Two accepts at once don’t break state |
| **D**urability | Committed survives crash | After accept, restart DB → still friends |

BG’s SQL-X path can use transactions for writes like **Accept Friend Request (AFR)**.

---

## 2.5 BASE (looser, scale-oriented)

**Basically Available, Soft state, Eventual consistency**

- **Available:** keep serving even if some replicas lag.
- **Soft state:** state may change without new input (replication catching up).
- **Eventual consistency:** if you stop writes, replicas **eventually** match.

**Example:** You accept a friend in LA; your friend in Tokyo still sees “pending” for 200 ms, then “friends.” That 200 ms is **staleness**—often acceptable for feeds, painful for payments.

BG cares because **memcached** + **TTL** push you toward **eventual** behavior on reads.

---

## 2.6 Replication (why CAP exists)

Big sites copy data to **multiple machines** for:

- **Fault tolerance** (one machine dies).
- **Geography** (users near a replica).

**Replicas** must **sync**. Sync needs **network**. If network **breaks** → **partition** (Part 2.8).

---

## 2.7 Caching and memcached

### Q: Why cache?

Reading PostgreSQL from disk/network every time is **slow**. **Hot** data (celebrity profile) gets read millions of times.

**Cache** = fast memory holding copies of hot data.

### Q: What is memcached?

**memcached** = simple **in-memory key-value** server.

- **Key:** e.g. `profile:userid:5`
- **Value:** serialized profile bytes (maybe thumbnail too)

App logic:

1. Read cache key.
2. **Hit** → return fast.
3. **Miss** → read DB, fill cache, return.

### Q: What is CASQL?

**Cache-Augmented SQL** = **SQL database is truth** + **memcached** for speed. The **application** (or middleware) must:

- Know **what** to cache.
- **Invalidate** or **expire** cache on writes.

BG compares **SQL-X** (no cache) vs **CASQL** (with cache)—same schema, different performance **and** different **staleness**.

### Q: What is the 1 MB trap?

Default memcached limits **value size** (~**1 MB** per key).  
**List Friends** might try to cache “all friends + all thumbnails” as **one** value.

**Example:** 1000 friends × 2 KB thumb ≈ **2 MB** → **cannot cache** → every LF hits DB → cache useless → CASQL **SoAR** can **collapse**.

---

## 2.8 Network partition (detailed)

### Q: What is a network partition?

Nodes that **should** talk **cannot** reach each other, but each side may still **run**.

**Not:** whole datacenter power off (that’s outage).  
**Yes:** fiber cut **between** East and West replicas.

```
  [East app + East DB]  ----X----  [West app + West DB]
         island A                      island B
```

### Q: Example: friend request during partition

- **East** records: Alice and Bob are **friends**.
- **West** never got the update: still **pending**.

Users on West see wrong state until replication heals → **inconsistency**.

### Q: What is CAP?

During a **partition**, you generally choose:

| Choice | During partition you… | Social example |
|--------|------------------------|----------------|
| **CP** | Sacrifice **availability** (errors/waits) to avoid **wrong** answers | “Can’t list friends right now—replica unsure” |
| **AP** | Stay **available**, accept **stale** answers until sync | Show old friend list quickly |

**P** (partition tolerance) = you admit partitions happen; you don’t pretend the network is perfect.

BG mentions **CP/AP** because you can benchmark behavior **under failures**, not only sunny-day throughput.

---

## 2.9 Latency, percentiles, and SLA

### Q: What is latency / response time?

Time from **click** (request sent) until **answer** (response received).

### Q: Why not average latency?

**Average hides disasters.**

Example: 99 requests at 10 ms, 1 request at 10,000 ms.  
Average ≈ 109 ms (looks OK).  
**One user** waited **10 seconds**.

Social apps care about **tail**: “How bad is it for the unlucky 5%?”

### Q: What does “95% under 100 ms” mean?

Sort all latencies in a run. **95%** of them are ≤ 100 ms; **5%** can be slower. If that 5% is huge, you **fail** the SLA.

### Q: What is an SLA here?

A **contract** for the benchmark run, e.g.:

1. **≥ 95%** of actions ≤ **100 ms**
2. **≤ 0.01%** (or 0.001%) **unpredictable** reads
3. Sustained for **10 minutes**

All three matter. A 30-second lucky sprint doesn’t count.

---

## 2.10 Closed vs open workload

### Closed (BG **Socialites**)

Fixed **N threads**. Each thread:

1. Send action.
2. **Wait** for response.
3. Send next.

Models **real users** staring at the screen.

### Open

New requests arrive at fixed rate **even if** the system is backed up → unrealistic pile-up for “interactive” modeling.

BG emphasizes **closed** for **Socialites**.

---

## 2.11 Throughput vs concurrency

- **Throughput** = actions completed per second (SoAR cares about max under SLA).
- **Concurrency** = how many actions **in flight** (threads/socialites).

More threads → often higher throughput until the DB/cache **saturates** and latency explodes.

---

# Part 3 — What BG is (the whole package)

### Q: One paragraph definition?

**BG** is a **benchmark framework** that:

1. Loads a **large** synthetic social graph into your data store.
2. Runs **11 user-like actions** with configurable mixes and **Zipfian** hot users.
3. Measures **latency**, **throughput**, and **correctness of reads** (**unpredictable data**).
4. Reports **SoAR** and **Socialites** = best performance **while SLA holds**.

### Q: What are the six design decisions (expanded)?

1. **SLA ratings (SoAR + Socialites)**  
   Not “we did 50k ops/sec once” but “max ops/sec **while** 95% &lt; 100 ms **and** almost no wrong reads.”

2. **Unpredictable data in the SLA**  
   Fast wrong answers are **failures**, like slow right answers.

3. **Realistic access**  
   Small reads/writes on huge graphs; skewed popularity.

4. **Store-agnostic**  
   Implement the same actions on SQL-X, CASQL, MongoDB.

5. **Logical partitioning**  
   Split data so many **BGClients** can scale on clusters.

6. **Visualization**  
   Watch the run live; find bottlenecks (CPU, disk, cache, network).

---

# Part 4 — Architecture: who does what?

### Components

| Piece | Role |
|-------|------|
| **BGCoord** | Boss: config, divide work, aggregate metrics, final SoAR/Socialites |
| **BGClient** | Worker: threads pretend to be **socialites**, issue SNAs |
| **Target DB** | Your SQL-X / CASQL / MongoDB |
| **memcached** | Only for CASQL |
| **Validation** | Checks reads against ground-truth logs |
| **Viz deck** | Charts during run |

### Story: one benchmark run (example)

1. You say: SLA = 95% / 100 ms / 0.01% unpredictable / 10 min; mix = read-heavy; store = CASQL.
2. **BGCoord** creates a 10M-member dataset (or uses prebuilt), partitions it.
3. **BGCoord** tells **4 BGClients**: “Client 1 handles partition A, 25% of traffic each,” etc.
4. Each **BGClient** spins **200 threads** (200 socialites).
5. Threads loop: pick action from mix → call store → record latency.
6. **Validation** compares read results to logs: was profile stale?
7. After 10 minutes, **BGCoord** asks: “Did we meet SLA?” If yes, try **more** threads or load (search for max SoAR).
8. You get: **SoAR = 12,442 actions/sec** (example number from paper figures—your run differs).

### Q: What is a socialite?

A **simulated user** with an identity in the dataset. Often **1 thread = 1 socialite** so actions look like independent users.

### Q: What is logical partitioning?

Split users by `userid` ranges:

- Client 1: users 1–2,500,000  
- Client 2: users 2,500,001–5,000,000  

Reduces two threads **fighting** over the same row and lets the benchmark **scale out** like real sharded systems.

---

# Part 5 — Data model (with running example)

**Characters:** **Alice (id=5)**, **Bob (id=7)**, **Carol (id=12)**.

### Members

One row/doc per person: name, email, **thumbnail**, etc.

### Friends

| userid1 | userid2 | status |
|---------|---------|--------|
| 5 | 7 | pending |
| 5 | 12 | confirmed |

Bob sent Alice a request → pending. Alice and Carol already friends → confirmed.

### Resources (wall posts)

| rid | creatorid | wallUserid | body |
|-----|-----------|------------|------|
| 900 | 12 | 5 | “Happy birthday Alice!” |

Post **on Alice’s wall**, written by Carol.

### Manipulation

History of edits to resources (who commented, when). Helps **validate** that after a comment action, reads see the new state.

### Why four tables?

- **Members + Friends** = graph + profiles  
- **Resources** = feed/wall  
- **Manipulation** = audit trail for **write** actions and **correctness**

---

# Part 6 — The eleven actions (user stories)

For each: **what the user sees**, **read or write**, **why it stresses the system**.

| # | Action | User story | Stress |
|---|--------|------------|--------|
| 1 | **VP** View Profile | Open someone’s profile page | Large **thumbnails**, cacheable |
| 2 | **LF** List Friends | See all friends with tiny avatars | **N+1** reads, huge cache values |
| 3 | **SFR** Send Friend Request | Add pending edge | Write + invalidate lists |
| 4 | **AFR** Accept Friend Request | Accept Bob | Write + **many** cache keys |
| 5 | **RFR** Reject | Decline request | Write |
| 6 | **UF** Unfriend | Remove friend | Write + invalidate |
| 7 | **LPFR** List Pending | See who invited you | Read pending set |
| 8 | **WR** Write Resource | Post on wall | Write blob/text |
| 9 | **RR** Read Resource | View wall/feed item | Read |
| 10 | **CR** Comment | Comment on post | Write + manipulation row |
| 11 | **SR** Share | Share post | Write |

**Exam favorites:** **VP**, **LF**, **AFR** — paper walks through implementations.

---

## 6.1 View Profile (VP) — step by step

1. Socialite requests profile for user 5 (Alice).
2. System returns name, fields, **thumbnail bytes**.
3. **SQL-X:** SELECT row + LOB.  
4. **MongoDB:** GET document `userid=5`.  
5. **CASQL:** try `profile:5` in memcached; on miss, SQL then populate cache.

**Experiment knob:** thumbnail **2 KB vs 500 KB**. At 500 KB, moving data dominates → **p95 latency** fails → **SoAR → 0** for SQL-X on VP.

---

## 6.2 List Friends (LF) — step by step

1. Load Alice’s confirmed friend ids: [7, 12, 99, …].
2. For **each** id, fetch name + thumbnail (paper’s logical spec).

**Why brutal:**

- 500 friends × 2 KB = **1 MB+** if bundled for cache.
- memcached **rejects** or skips → **no benefit**.
- Mongo might do **500 document reads** if not denormalized.

---

## 6.3 Accept Friend Request (AFR) — three implementations

**Story:** Alice (5) accepts Bob (7).

**SQL-X:**

```sql
UPDATE Friends SET status='confirmed' WHERE userid1=7 AND userid2=5;
-- (exact keys depend on schema)
```

**MongoDB:**

- Update Bob’s doc: add 5 to `confirmedFriends`, remove from `pendingFriends`.
- Update Alice’s doc symmetrically.

**CASQL:**

- Same SQL update.
- **Invalidate** keys like: `profile:5`, `profile:7`, `friends:5`, `friends:7`, `pending:5`, …

If invalidation is wrong or late, next **VP/LF** returns **unpredictable** data (Bob still “pending” in cache).

---

# Part 7 — Metrics (every term)

### SoAR (Social Action Rating)

**Question answered:** “How many social actions per second can this store handle **while** meeting the SLA?”

**How found:** increase load until SLA breaks; back off; report maximum.

### Socialites

**Question answered:** “How many **concurrent** socialites (threads) can we run **while** meeting the SLA?”

Uses **closed** model.

### Unpredictable data

**Question answered:** “How often did we show **wrong/stale** data?”

Must stay below SLA cap (e.g. 0.01%).

### Freshness confidence

**Question answered:** “If we use weak consistency (TTL), how confident are we reads are fresh?”

Characterizes **TTL policies**, not just pass/fail.

---

# Part 8 — Unpredictable data (full story)

### Q: Why is this not just “error rate”?

HTTP **200 OK** with **stale friend list** = success in logs, **failure** in BG.

### Q: How does validation work (conceptually)?

1. During the run, BG logs **every write** (ground truth timeline).
2. On each **read**, compute what values are **legal** given all writes up to that time.
3. If the store returns something else → **unpredictable** event.

### Q: Timeline example (cache)

| Time | Event |
|------|--------|
| T0 | Alice accepts Bob; DB updated |
| T1 | Cache still has old `friends:5` (TTL not expired, invalidation missed) |
| T2 | LF returns list **without** Bob → **unpredictable** |
| T3 | TTL expires or invalidation runs → fixed |

### Q: Tie to CAP/AP

Under partition or weak cache policy, **AP-like** behavior → more unpredictable reads. BG **counts** them in the score.

---

# Part 9 — Invalidation vs TTL (timelines)

### Invalidation (what paper’s CASQL prefers)

**Write path:**

1. Update database (commit).
2. **Delete** affected cache keys immediately.

**Read path:** miss → fresh from DB.

**Result:** unpredictability **very low** (&lt; 0.001% in reported runs).

### TTL only

**Write path:** update DB; **do not** delete cache.

**Read path:** serve cache until age &gt; TTL (30/60/120 sec in experiments).

**With 10% writes:** stale entries survive → unpredictability can hit **80–90%** at high thread counts.

### Gumball

Race: read refills cache **between** DB commit and invalidation. **Gumball** (USC) orders these safely; cited to push unpredictability toward **zero**.

---

# Part 10 — Workload mixes and Zipfian

### Three mixes (Table 3 in paper)

Exact percentages are in the PDF; conceptually:

1. **Read-heavy** — mostly VP, LF, RR → caches shine.  
2. **Write-heavy** — SFR, AFR, WR, CR → invalidation stress.  
3. **Combined** — realistic; **different winners** per action; **CASQL** often best **overall** with invalidation.

### Write fraction experiments

**0.1%, 1%, 10%** writes in mix. More writes → TTL looks worse; invalidation holds better.

### D-Zipfian / Zipfian

**80/20 rule:** few users get **most** visits (celebrities). Creates **hot keys** in cache and **lock contention** on DB—realistic, harsh test.

---

# Part 11 — Experiments in the paper (what each figure type means)

### Systems compared

- **SQL-X** — RDBMS alone  
- **CASQL** — RDBMS + memcached  
- **MongoDB** — document store  

### VP vs image size

- **2 KB:** everyone reasonable SoAR  
- **500 KB:** SQL-X may **fail SLA** entirely on VP  

**Lesson:** payload size is not a footnote—it **is** the benchmark outcome.

### SoAR vs threads (T)

X-axis: more concurrent threads.  
Y-axis: throughput or % meeting latency target.

You see a **cliff**: past some T, latency tail blows up → SLA fails.

### Mixed writes bar charts

Compare MongoDB, CASQL, SQL-X at **0.1% / 1% / 10%** writes. Leadership **changes**; no universal winner.

### TTL figures

TTL = 30, 60, 120 seconds; ~10% writes.  
Higher TTL → **more** unpredictable data as T grows.

### BGClient scaling

1, 2, 4, 8, 16 clients → throughput up until bottleneck (DB, network, coordinator).

### SoAR search cost

Paper shows **many trials** to find max SoAR (state space)—rating is **expensive** to compute precisely.

---

# Part 12 — Main conclusions (what to write on an exam)

1. **No system wins every action** — depends on read vs write, payload, cache design.  
2. **Mixed workloads:** **CASQL + invalidation** often **highest SoAR**.  
3. **Unpredictable data** must be in the SLA when caches exist.  
4. **Design details matter:** thumbnail size, cache key granularity, 1 MB limit, invalidation vs TTL.  
5. BG enables comparing **physical designs** (same logical benchmark) and **failure modes** (CP/AP).

---

# Part 13 — Related work (why each citation exists)

| Work | One line |
|------|----------|
| **YCSB** | Generic cloud KV; BG adds social + unpredictability |
| **TPC** | OLTP standard; wrong domain |
| **Feed following (DBSocial)** | Why read-heavy SN is hard |
| **Facebook memcached / TAO** | Production uses caches; BG quantifies cost |
| **Vogels eventual consistency** | Theory behind AP |
| **Gumball** | Fix cache races |
| **LDBC SNB** (later) | Graph benchmark; different era |

---

# Part 14 — Limitations (honest)

- Results depend on **how** you implement actions (schema, cache keys).  
- **11 actions** ≠ entire Facebook (no search, ads, DMs).  
- SLA numbers are **parameters**—change 100 ms to 50 ms, rankings shift.  
- Paper’s PDF encoding is poor; check **figures** for exact table %.

---

# Part 15 — Mega FAQ (questions you might not know to ask)

### Q: Is BG open source?

Project released via **bgbenchmark.org** (framework for research). Your course may use the paper, not require you to run it.

### Q: Difference between SoAR and Socialites again?

**SoAR** = max **actions/sec** under SLA.  
**Socialites** = max **parallel users (threads)** under SLA.  
Related but not identical optima.

### Q: Can MongoDB “win”?

Yes on some **read** actions or document-friendly patterns; loses on others. Paper’s point: **no monopoly**.

### Q: Why measure both latency and unpredictability?

Caches improve **latency** but worsen **correctness** unless managed. Measuring only speed **rewards lying**.

### Q: What if memcached is down in CASQL?

Implementation-dependent; often falls back to SQL (slower). BG would show **lower SoAR**, not necessarily high unpredictability if DB is correct.

### Q: What is a BGClient vs a thread?

**BGClient** = process/machine. **Threads** inside = socialites. Many clients × many threads = huge concurrency.

### Q: Does BG use real Facebook data?

**No** — synthetic graph at configurable scale (privacy + reproducibility).

### Q: CP vs unpredictable data?

**CP** is about **replica partition** behavior. **Unpredictable data** in BG also includes **cache staleness** even **without** partition—everyday CASQL issue.

### Q: What’s “think time”?

Real users pause between clicks. BG can model **sessions** (sequence of actions); closed loop captures **wait for response** between actions in a thread.

### Q: Why 10 minutes duration?

Stops you from reporting a **spike** that wouldn’t hold in production.

### Q: 0.01% vs 0.001% unpredictable?

Stricter SLA → harder to pass → lower reported SoAR. Paper uses both in different experiments—**read the figure caption**.

### Q: What is physical vs logical design?

**Logical:** VP means “return profile fields.”  
**Physical:** one big cache key vs many small keys, Mongo embedding vs normalize—that’s what BG compares.

### Q: LinkBench vs BG?

**LinkBench** (Facebook): social graph workloads for MySQL/etc.  
**BG** (USC): **SLA + SoAR/Socialites + unpredictable data** emphasis, CASQL focus.

### Q: Will professor ask CAP?

Often **yes** in systems courses: BG **claims** you can study CP/AP under failures; primer Part 2.8 is your answer.

### Q: Single sentence to open an oral exam?

“BG is an SLA-driven benchmark for eleven interactive social networking actions that reports maximum throughput and concurrency—SoAR and Socialites—while bounding stale or inconsistent reads called unpredictable data, illustrated on SQL, cache-augmented SQL, and MongoDB.”

---

# Part 16 — Study order (if overwhelmed)

1. Part 2 (concepts) — **one afternoon**  
2. Part 3–4 (what BG is, architecture)  
3. Part 5–6 (data model + 11 actions)  
4. Part 8–9 (unpredictable + cache)  
5. Part 11–12 (results + exam conclusions)  
6. Flashcards deck K  

---

*End of full primer. Pair with `BG_Paper.pdf` figures for exact Table 2/3 numbers.*
