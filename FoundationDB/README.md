# FoundationDB — Paper, concurrency control & graph lecture

**Paper:** Jingyu Zhou et al. *FoundationDB: A Distributed Key-Value Store* (SIGMOD Record / SIGMOD ’21 revision). **[P]**  
**Concurrency control:** CSCI 585 discussion deck [`DiscussionXactCC-1.pdf`](DiscussionXactCC-1.pdf) — ties to FoundationDB’s optimistic + multi-version design. **[CC]**  
**Guest lecture:** Hieu Nguyen (eBay), *FoundationDB and Its Use for A Graph Database* [`2026_FoundationDB_and_Its_Use_for_A_Graph_Database_v3.pdf`](2026_FoundationDB_and_Its_Use_for_A_Graph_Database_v3.pdf). **[G]**

**Course context:** CSCI 585 (Shahram Ghandeharizadeh), USC — connects to JanusGraph, Grasper, and Gamma-style shared-nothing ideas.

---

## Suggested study order (exam prep)

| Step | Focus | File |
|------|--------|------|
| **1** | Read straight through in plain language | [`FoundationDB_Full_Primer.md`](FoundationDB_Full_Primer.md) |
| **2** | Structured reference + cheat sheet | [`FoundationDB_Study_Guide.md`](FoundationDB_Study_Guide.md) |
| **3** | Why transactions need concurrency control | Primer **Part 0** + [`DiscussionXactCC-1.pdf`](DiscussionXactCC-1.pdf) |
| **4** | Transaction path, recovery, simulation | Primer **Parts 2–4** + paper **[P]** |
| **5** | JanusGraph on FoundationDB + caching | Primer **Parts 5–6** + guest deck **[G]** |
| **6** | Drill + mock exam | [`FoundationDB_Flashcards.md`](FoundationDB_Flashcards.md) + [`FoundationDB_Exam_QA.md`](FoundationDB_Exam_QA.md) |
| **7** | Slide pass | [`FoundationDB_Lecture_Slides.md`](FoundationDB_Lecture_Slides.md) |

---

## Files in this folder

| File | Purpose |
|------|---------|
| **FoundationDB.pdf** | SIGMOD paper (distributed transactional key-value store) |
| **DiscussionXactCC-1.pdf** | Serializable schedules, locking, timestamps, optimistic validation, multi-version |
| **2026_FoundationDB_and_Its_Use_for_A_Graph_Database_v3.pdf** | Deployment, components, eBay graph + per-hop cache |
| **FoundationDB_Full_Primer.md** | Continuous narrative from zero |
| **FoundationDB_Study_Guide.md** | Sections, glossary, transaction steps |
| **FoundationDB_Lecture_Slides.md** | Bullets mapped to all three PDFs |
| **FoundationDB_Flashcards.md** | Closed-book drill |
| **FoundationDB_Exam_QA.md** | Example questions with model answers |
| **FoundationDB_References.md** | Citations and claim index |

Tags: **[P]** paper · **[CC]** concurrency deck · **[G]** eBay graph lecture.
