# Entity-Relationship Lecture — Slide map

**Deck:** [`PPT_Lecture.pdf`](PPT_Lecture.pdf) (53 slides) · **Paper:** [`TheEntity-RelationshipModel.pdf`](TheEntity-RelationshipModel.pdf)  
**Course intro (separate deck):** [`Class_PPT.pdf`](Class_PPT.pdf) — see §0 below.

**Study:** [`ER_Model_Full_Primer.md`](ER_Model_Full_Primer.md) · **Outline time:** ~75–90 minutes with paper skim.

---

## §0 — Class_PPT.pdf (18 slides) **[C]**

| Slides | Topic |
|--------|--------|
| 1–2 | CSCI 585 welcome, instructor contact |
| 3–5 | Database and database management system definitions; secondary storage |
| 6–7 | Scale of modern apps (volume, variety) |
| 8–10 | Before/after database management system (conceptual) |
| 11 | Why a database management system: development time, independence, sharing, integrity, administrator control, security, concurrency, recovery |
| 12 | Conceptual / logical / physical data models (medical records example) |
| 13–18 | Guest lectures, grading, exams on assigned papers, teaching assistants |

---

## PPT_Lecture.pdf — Chen presentation

| Slides | Topic | Notes |
|--------|--------|--------|
| 1–2 | Title, assigned reading | Chen 1976 |
| 3–5 | Entity-relationship model; level 1 vs level 2 | Entities, relationships, attributes, entity sets |
| 6–7 | (Diagrams) | Rectangles, diamonds, attributes |
| 8–11 | Relationship sets; binary, ternary, n-ary | Formal subset definition; fans example |
| 10 | Partial vs total participation | John not required to fan a team; double line = all people are fans |
| 11 | N-ary; decompose to binary | Date on ternary fan relationship |
| 12–13 | Library example | Authors, books, subjects, libraries; wrote, indexed, carry |
| 14–16 | Fraud detection, recommendations | Graph queries at scale |
| 17 | Binary cardinalities | One-to-one, many-to-one, one-to-many, many-to-many |
| 18–22 | Keys | Superkey, candidate key, primary key (semantic) |
| 23 | Relationship keys | Union of keys (many-to-many); many side only (many-to-one) |
| 24–25 | Library entity-relationship diagram + keys | ISBN, subject matter, address |
| 25–26 | Foreign key; weak vs strong entities | Book edition example |
| 27 | Exercise | Employee reports-to hierarchy (recursive) |
| 28 | Manufacturing Section 3.1 | Chen diagram features checklist |
| 29–31 | Reduction to tables; integrity constraints | Allowable vs existing values |
| 32–33 | Functional dependency | Level 1 attribute as function; level 2 key determines non-keys |
| 34 | Multi-valued attributes | Not allowed in first normal form at level 2 |
| 35 | Second/third normal form not at level 1 | Top-down vs bottom-up |
| 36–37 | Second normal form violation | Part–warehouse–address; decomposition |
| 38 | Normalization pros/cons | Join cost example |
| 39–40 | Third normal form violation | Employee–department–location ambiguity |
| 41 | Network model | Owner–member arrows |
| 42–43 | Entity set model | Entity-name-set / entity-name pairs |
| 44 | Discussion | |
| 45–48 | Generalization / specialization | Disjoint, overlap, total |
| 49 | Relational basics | Tuple, attribute, domain, instance |
| 50–53 | Table generation rules | Strong, weak, many-to-many, functional, specialization approaches |

---

## Suggested talk track (oral exam)

1. **Motivation** — semantics and unified view of network, relational, entity set. **[P]**
2. **Level 1** — entities, relationships, attributes on relationships. **[P][L]**
3. **Diagram** — cardinality, participation, weak entities. **[L]**
4. **Level 2** — keys, tables, normal forms, mapping rules. **[L][P]**
5. **Compare models** — network arrows vs entity-relationship meaning. **[P]**
