# FoundationDB — References & claim index

## Primary sources (this folder)

| ID | Source | File |
|----|--------|------|
| **P** | Zhou et al. *FoundationDB: A Distributed Key-Value Store.* SIGMOD Record 51(1), 2022 (SIGMOD ’21 revision). DOI 10.1145/3448016.3457559 | [`FoundationDB.pdf`](FoundationDB.pdf) |
| **CC** | Ghandeharizadeh. *Concurrency Control* discussion slides (CSCI 585). | [`DiscussionXactCC-1.pdf`](DiscussionXactCC-1.pdf) |
| **G** | Nguyen, H. *FoundationDB and Its Use for A Graph Database* (eBay guest lecture, 2026 deck). | [`2026_FoundationDB_and_Its_Use_for_A_Graph_Database_v3.pdf`](2026_FoundationDB_and_Its_Use_for_A_Graph_Database_v3.pdf) |

## Related course folders

| Folder | Connection |
|--------|------------|
| **Grasper** | Contrasts in-memory OLAP Gremlin vs JanusGraph+FDB OLTP backend |
| **ER_Model / Chen** | Data modeling vs storage engine layer |
| **Gamma / HRPS** | Shared-nothing scaling vocabulary |

## External citations (from guest lecture)

- Ghandeharizadeh, Yap, Nguyen — strong consistency in cache-augmented SQL (Middleware 2014) — eBay cache invalidation model **[G]**
- Nguyen, Li, Ghandeharizadeh — one-hop subquery result caches **[G]**
- Nguyen, Li, Ghandeharizadeh — graph stores with application-level query result caches (TPCTC 2023) **[G]**
- FoundationDB Record Layer arXiv:1901.04452 **[G]**

## Claim index

| Tag | Typical exam claims |
|-----|---------------------|
| **[P]** | Architecture; transaction steps; strict serializability; logging; f+1; simulation; layers |
| **[CC]** | Serializable / strict; 2PL; OCC phases; MVCC read version |
| **[G]** | Deployment; component names; read errors; recovery acronyms; JanusGraph keys; caches; quiz answers |
