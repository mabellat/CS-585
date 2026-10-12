# FoundationDB — Example Exam Q&A

Tags: **[P]** paper · **[CC]** concurrency deck · **[G]** eBay lecture.

---

## Short answer

**1. What problem does FoundationDB solve?** **[P]**  
Scalable distributed storage with **ACID** transactions and **strict serializability**, minimal core features, and **layers** for higher data models.

**2. Name four data-plane roles.** **[P][G]**  
Sequencer, proxy, resolver, log server, storage server (any four of these five).

**3. Why separate transaction system from storage system?** **[P]**  
Scale reads and writes independently; reads hit storage servers directly.

**4. What is optimistic concurrency control in FoundationDB?** **[P][CC]**  
Transactions read and buffer writes without locking; at commit **resolvers** check read-write conflicts; abort if conflict.

**5. What does the sequencer provide?** **[P]**  
Read versions and commit versions that define serial order (log sequence numbers).

**6. Serializable schedule in one sentence?** **[CC]**  
Interleaved execution with the same final state as some serial execution of the transactions.

**7. Read-only transaction commit?** **[G][P]**  
If there are no writes, client **skips** the full commit protocol.

**8. False-positive conflict after abort?** **[G][P]**  
One resolver may have recorded state before another aborted; later transactions might conflict spuriously within MVCC window; still correct.

---

## Medium answer

**9. Walk through a write transaction.** **[P][G]**  
Get read version from proxy/sequencer; read keys at version from storage servers; buffer writes locally; on commit send read/write sets to proxy; get commit version; resolvers validate; log durable; storage servers apply asynchronously.

**10. Why FoundationDB uses simulation?** **[P]**  
Distributed bugs are rare and hard to reproduce; deterministic simulation with fault injection finds corruption risks before production; bugs replay exactly.

**11. Recovery version (RV) and previous epoch end (PEV)?** **[G][P]**  
PEV = max committed replicated log sequence from old epoch; RV = min durable version across log servers; redo/trim between them; storage drops versions after RV.

**12. Two-phase locking vs FoundationDB.** **[CC][P]**  
Two-phase locking holds locks during execution; FoundationDB avoids lock coupling between transaction and storage via **optimistic validation** at commit (low conflict assumption).

**13. JanusGraph mapping for `g.V(1).outE('route')`.** **[G]**  
Vertex id 1; **getRange** on key prefix for outgoing `route` edges; deserialize edge records.

**14. Per-hop cache vs full-query cache.** **[G]**  
Per-hop: cache expensive traversal segments; cheaper invalidation. Full-query: best for batch-loaded read-only graphs; invalidation costly on writes.

**15. eBay strong consistency with cache.** **[G]**  
Write-around: update graph keys and **invalidate** cache keys in the **same** FoundationDB transaction.

---

## Long / design

**16. Compare JanusGraph+FoundationDB to Grasper.** **[G]** (+ course Grasper notes)  
JanusGraph: durable transactional KV, Gremlin OLTP/online analytics, stateless graph processes possible. Grasper: in-memory sharded OLAP, RDMA, Experts — not the same durability model.

**17. Why f+1 replicas instead of 2f+1 quorums?** **[P]**  
Failures trigger **reconfiguration** and recovery rather than only masking with quorum reads; fewer replicas for same fault tolerance in their design.

**18. Optimistic validation rule (lecture) for transaction Tj.** **[CC]**  
For all older Ti: either Ti finished before Tj started, or write set of Ti disjoint from read set of Tj and Ti finished write before Tj validates.

**19. LAX supernode query optimization story.** **[G]**  
Without cache: index lookup + huge edge getRange + filter. With template cache: lookup + cache hit for neighbor ids after first hop.

**20. Layers on FoundationDB — why?** **[P]**  
One transactional ordered store; relational (Record Layer), document, graph (JanusGraph) share same correctness and operations backbone.

---

## Oral one-minute pitches

| Topic | Pitch |
|-------|--------|
| FoundationDB | Unbundled transactional KV; sequencer versions; optimistic + MVCC; simulation-tested |
| Concurrency link | Serializable illusion; FDB picks optimistic at commit + versions on read |
| eBay graph | JanusGraph on FDB; Gremlin waves; per-hop transactional cache |
