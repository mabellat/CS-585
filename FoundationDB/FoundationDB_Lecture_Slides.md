# FoundationDB — Slide maps (three sources)

---

## DiscussionXactCC-1.pdf (48 slides) **[CC]**

| Slides | Topic |
|--------|--------|
| 2–5 | Moore’s law; multicore; waste if one transaction at a time |
| 6–11 | Serializable vs **strict serializable** schedules; n! serial orders |
| 13–17 | Independent writes on X,Y,Z — only some serial orders matter when conflicts exist |
| 18–22 | Bank transfer interleaving; lost update; conflict rules read/write |
| 23–30 | **Lock-based:** S/X locks; deadlock / wait-for graph; **two-phase locking**; isolation levels 0–3 |
| 31 | Multi-granularity locks (IS, IX, S, X, SIX) |
| 32–37 | **Timestamp** ordering; W-timestamp / R-timestamp; rollback rules |
| 38–45 | **Optimistic concurrency control:** read, validate, write; validation vs older transactions |
| 46–48 | **Multi-version** schemes; read version selection; write creates new version |

**Link to FoundationDB:** resolvers = optimistic validation; storage versions = multi-version reads. **[P]**

---

## FoundationDB.pdf (SIGMOD paper) **[P]**

| Section | Topic |
|---------|--------|
| §1 Intro | NoSQL vs ACID; layers; simulation-first culture |
| §2 Design principles | Divide-and-conquer; failure as normal; simulation |
| §2.1 Architecture | Control plane vs data plane; Figure 1 transaction flow |
| §2.1.3 | Read/write scaling |
| §2.1.4–5 | Bootstrapping; reconfiguration; **epochs** |
| §2.2.1 | End-to-end transaction (steps 1, 2, 3.x) |
| §2.2.2 | Strict serializability; resolvers; false positives |
| §2.2.3–4 | Logging; recovery without ARIES undo |
| §2.3 | Replication: coordinators, logs, storage teams |
| §3 | Deterministic simulator; Flow; swarm; fault injection |

---

## 2026_FoundationDB_and_Its_Use_for_A_Graph_Database_v3.pdf (69 slides) **[G]**

| Slides | Topic |
|--------|--------|
| 3–6 | Disk vs DRAM; shared-nothing; Kubernetes context |
| 7–8 | Multi-DC deployment; FDB properties (MVCC, OCC, layers) |
| 9 | Components divide-and-conquer table |
| 10 | Bootstrapping / reconfiguration |
| 11–14 | Transaction: read version, reads, commit, resolvers |
| 15–18 | Logging; storage server versions; lag / future version |
| 19–21 | Recovery PEV/RV/KCV/DV; simulation; fault injection |
| 22 | Quiz: read-only commit; multi-resolver; false positive |
| 23–28 | FDB API; employee/dept example; getRange queries |
| 29 | Layers (Record, document, JanusGraph) |
| 30–44 | Graph model; JanusGraph; Gremlin; key layout; indices |
| 45–53 | Multi-hop waves; LAX supernode; per-hop cache templates |
| 54–61 | Read/write path with cache; invalidation cases; FDB as cache store |
| 62–63 | Full-query cache; batch ML pipeline |
| 64–69 | Summary; consistency models; graph DB list; JanusGraph quiz |

---

## Suggested 90-minute study session

1. **20 min** — Concurrency deck optimistic + multi-version (slides 38–48). **[CC]**
2. **35 min** — Paper architecture + transaction Figure 1. **[P]**
3. **25 min** — Guest lecture transaction + JanusGraph + cache. **[G]**
4. **10 min** — [`FoundationDB_Exam_QA.md`](FoundationDB_Exam_QA.md) oral answers.
