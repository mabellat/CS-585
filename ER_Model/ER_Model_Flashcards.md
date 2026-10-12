# Entity-Relationship Model — Flashcards

Read once: **[ER_Model_Full_Primer.md](ER_Model_Full_Primer.md)**. Cover the right column and answer aloud.

---

## Deck A — Big picture

| # | Question | Answer |
|---|----------|--------|
| A1 | What is Chen’s entity-relationship model for? | Describe real-world entities and relationships with clear semantics; design databases; unify network, relational, and entity set views. **[P]** |
| A2 | What are the two primitives? | Entity sets and relationship sets. **[P][L]** |
| A3 | What is an attribute mathematically? | A function from an entity or relationship to a value set (one value per entity). **[P]** |
| A4 | Name the four logical levels (short). | (1) conceptual entities/relationships, (2) information structure/tables, (3) access-independent relations, (4) access-dependent storage. **[P]** |

---

## Deck B — Diagrams and constraints

| # | Question | Answer |
|---|----------|--------|
| B1 | Rectangle vs diamond? | Entity set vs relationship set. **[L]** |
| B2 | Double line on entity side? | Total participation — every entity in that set must appear in the relationship. **[L][P]** |
| B3 | Many-to-many example? | Students enrolled in classes. **[L]** |
| B4 | Weak entity set? | No primary key from own attributes; depends on owner; identifying relationship; total participation on weak side. **[L][P]** |
| B5 | Disjoint vs overlap specialization? | **d:** at most one subclass; **o:** can be in multiple subclasses. **[L]** |

---

## Deck C — Keys and tables

| # | Question | Answer |
|---|----------|--------|
| C1 | Superkey vs candidate key? | Superkey uniquely identifies; candidate key is minimal superkey. **[L]** |
| C2 | Many-to-many relationship table key? | Usually all participating primary keys combined. **[L]** |
| C3 | Many-to-one from A to B — where is foreign key? | On the many side (A). **[L]** |
| C4 | Foreign key in weak entity? | Owner’s primary key columns in weak table. **[L]** |
| C5 | Arity vs cardinality of a relation? | Arity = column count; cardinality = row count. **[L]** |

---

## Deck D — Normal forms

| # | Question | Answer |
|---|----------|--------|
| D1 | Why no multi-valued attribute in one column? | Violates first normal form. **[L]** |
| D2 | Second normal form violation? | Non-key attribute depends on only part of a composite key. **[L]** |
| D3 | Third normal form violation? | Non-key attribute depends on another non-key attribute. **[L]** |
| D4 | Cost of normalization? | More joins; possible slower queries. **[L]** |
| D5 | Entity-relationship vs normalization approach? | Top-down from semantics vs bottom-up split of wide tables. **[P]** |

---

## Deck E — Other models

| # | Question | Answer |
|---|----------|--------|
| E1 | Network model arrow meaning? | One-to-many between **record types** plus access path; not always an entity relationship. **[P]** |
| E2 | Many-to-many in network? | Introduce a relationship record type. **[P][L]** |
| E3 | Entity set model quirk? | Values like “22” treated as entities; binary links only. **[P]** |
| E4 | Relationship attribute example? | Percentage of time on project (not on employee or project alone). **[P]** |
| E5 | Why database management system? (any three) | Independence, integrity, concurrency, recovery, security, sharing. **[C]** |
