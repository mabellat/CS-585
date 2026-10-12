# FoundationDB — Flashcards

Cover the answer column; reply aloud. Primer: **[FoundationDB_Full_Primer.md](FoundationDB_Full_Primer.md)**.

---

## Deck A — Concurrency control **[CC]**

| # | Question | Answer |
|---|----------|--------|
| A1 | Goal of concurrency control? | Parallel execution with serializable (or strict serializable) outcome |
| A2 | Serializable vs strict serializable? | Strict also respects real-time order of commits |
| A3 | Two-phase locking phases? | Growing (acquire locks); shrinking (release only) |
| A4 | Optimistic phases? | Read, validate, write |
| A5 | Why optimistic for read-heavy workloads? | Lock overhead often not worth it if conflicts rare |

---

## Deck B — FoundationDB core **[P]**

| # | Question | Answer |
|---|----------|--------|
| B1 | What is FoundationDB? | Ordered transactional distributed key-value store |
| B2 | Unbundled three parts? | Transaction system, log system, storage system |
| B3 | Who assigns read/commit versions? | Sequencer |
| B4 | Who checks conflicts? | Resolvers (range-partitioned) |
| B5 | Where do clients read? | Storage servers at a read version |
| B6 | Where are writes first buffered? | Client until commit |
| B7 | Default isolation strength? | Strict serializability |
| B8 | f failures, how many log replicas? | f+1 synchronous for commit |

---

## Deck C — Recovery & testing **[P][G]**

| # | Question | Answer |
|---|----------|--------|
| C1 | What happens when transaction system fails? | Sequencer ends; new epoch recruited |
| C2 | Why no ARIES-style undo? | Redo forward only; drop in-memory versions after RV |
| C3 | Deterministic simulation benefit? | Reproducible bugs |
| C4 | MVCC window (lecture)? | ~5 seconds in memory on storage server |

---

## Deck D — Graph & eBay **[G]**

| # | Question | Answer |
|---|----------|--------|
| D1 | JanusGraph storage at eBay? | FoundationDB key-value backend |
| D2 | Gremlin via? | Apache TinkerPop |
| D3 | Supernode latency issue? | One huge getRange wave |
| D4 | Per-hop cache key idea? | Query template + parameters |
| D5 | Write consistency with cache? | Graph mutation + cache invalidation in one FDB transaction |
| D6 | Read-only FDB transaction commit? | Skip commit protocol if no writes |
| D7 | Composite index range query? | No (equality/hash style) |
| D8 | Vertex-centric index range? | Yes (local to vertex edges) |
