# Entity-Relationship Model — Study Guide

**Paper:** Peter Pin-Shan Chen, *The Entity-Relationship Model — Toward a Unified View of Data*, ACM TODS 1(1), 1976. **[P]**  
**Lecture:** [`PPT_Lecture.pdf`](PPT_Lecture.pdf). **[L]**  
**Course intro:** [`Class_PPT.pdf`](Class_PPT.pdf). **[C]**

Continuous narrative: **[ER_Model_Full_Primer.md](ER_Model_Full_Primer.md)**. Drill: **[ER_Model_Flashcards.md](ER_Model_Flashcards.md)** · Exam: **[ER_Model_Exam_QA.md](ER_Model_Exam_QA.md)**.

---

## Table of contents

1. [Course context: database and database management system](#1-course-context-database-and-database-management-system)
2. [Glossary](#2-glossary)
3. [Four levels of logical views](#3-four-levels-of-logical-views)
4. [Level 1: entities, relationships, attributes](#4-level-1-entities-relationships-attributes)
5. [Diagram notation and constraints](#5-diagram-notation-and-constraints)
6. [Keys, weak entities, foreign keys](#6-keys-weak-entities-foreign-keys)
7. [Mapping to relational tables](#7-mapping-to-relational-tables)
8. [Functional dependency and normal forms](#8-functional-dependency-and-normal-forms)
9. [Generalization and specialization](#9-generalization-and-specialization)
10. [Integrity and data manipulation semantics](#10-integrity-and-data-manipulation-semantics)
11. [Network, relational, and entity set models](#11-network-relational-and-entity-set-models)
12. [Exam cheat sheet](#12-exam-cheat-sheet)

---

## 1. Course context: database and database management system

| Topic | One-line answer |
|-------|-----------------|
| Database | Integrated data on secondary storage describing related organizations. **[C]** |
| Database management system | Software to maintain and use large data collections. **[C]** |
| Why a database management system | Development speed, data independence, sharing, integrity, centralized tuning, security, concurrency, recovery. **[C]** |
| Conceptual → logical → physical | Real-world concepts → tables and keys → storage and access paths. **[C][P]** |

---

## 2. Glossary

| Term | Meaning |
|------|---------|
| **Entity** | A distinguishable object (person, team, project). **[P][L]** |
| **Entity set** | Same-type entities (all employees). Sets may overlap. **[P][L]** |
| **Relationship** | Association among entities. **[P][L]** |
| **Relationship set** | All relationships of one type. **[P][L]** |
| **Role** | Function of an entity in a relationship (worker, manager). **[P]** |
| **Attribute** | Function from entity or relationship set to value set(s). **[P][L]** |
| **Value set / domain** | Allowed values for an attribute. **[P][L]** |
| **Binary / ternary / n-ary** | Relationship involves 2, 3, or n entity sets. **[P][L]** |
| **Superkey** | Attributes that uniquely identify a row (may be redundant). **[L]** |
| **Candidate key** | Minimal superkey. **[L]** |
| **Primary key** | Chosen candidate key; prefer meaningful keys. **[L][P]** |
| **Foreign key** | Owner’s key columns appearing in weak entity table. **[L]** |
| **Strong entity set** | Has its own primary key. **[L][P]** |
| **Weak entity set** | Needs owner key; identifying relationship; total participation. **[L][P]** |
| **Partial participation** | Not every entity must join the relationship (single line). **[L]** |
| **Total participation** | Every entity must join (double line). **[L][P]** |
| **Cardinality** | One-to-one, one-to-many, many-to-one, many-to-many. **[L][P]** |
| **Functional dependency** | Same X always implies same Y in a table. **[L][P]** |
| **Normalization** | Split tables to reduce redundancy and update problems. **[L][P]** |
| **Arity (of relation)** | Number of columns in a table. **[L]** |
| **Cardinality (of relation)** | Number of rows in a table. **[L]** |

---

## 3. Four levels of logical views

| Level | Name | Entity-relationship role |
|-------|------|---------------------------|
| 1 | Information about entities and relationships | Conceptual — entities, relationship sets, attributes on them **[P]** |
| 2 | Information structure | Tables, primary keys, entity/relationship relations **[P][L]** |
| 3 | Access-path-independent structure | Relational relations without storage bias **[P]** |
| 4 | Access-path-dependent structure | Network record chains, indexes **[P]** |

**Network model** — mainly level 4. **Relational model** — levels 2–3. **Entity set model** — levels 1–2 with values-as-entities. **[P]**

---

## 4. Level 1: entities, relationships, attributes

- Separate **entity information** from **relationship information** — helps find functional dependencies. **[P]**
- Relationship attributes depend on **both** (or all) participants (percentage of time on project). **[P]**
- Multi-valued attributes allowed at level 1; must be reworked for tables. **[L]**
- Enterprise administrator decides borderline cases (is marriage an entity or a relationship?). **[P]**

---

## 5. Diagram notation and constraints

| Notation | Meaning |
|----------|---------|
| Rectangle | Entity set |
| Diamond | Relationship set |
| Line | Links relationship to entity set |
| Double line on entity side | Total participation |
| 1, n, m on edges | Mapping cardinality |
| Arrow to dependent | Existence dependency (weak entity) **[P]** |
| d in circle | Disjoint specialization |
| o in circle | Overlapping specialization |

**Binary cardinality examples (lecture):** women–men marriage (one-to-one); children–mother (many-to-one); students–classes (many-to-many). **[L]**

**Non-binary:** can decompose to binaries but may lose semantics (supplier–project–part). **[P]**

---

## 6. Keys, weak entities, foreign keys

**Relationship primary key (no relationship attributes):**

| Mapping | Primary key of relationship table |
|---------|-----------------------------------|
| Many-to-many | Union of all participant primary keys **[L]** |
| Many-to-one from E1 to E2 | Primary key of E1 (many side) **[L]** |

**Weak entity:** partial key + owner primary key. Identifying relationship is many-to-one from weak to owner; weak side total participation. **[P][L]**

**Regular vs weak entity relation (Chen):** weak when identification uses another entity’s key (dependents). **[P]**

---

## 7. Mapping to relational tables

| Construct | Relational mapping |
|-----------|-------------------|
| Strong entity | One table, all attributes |
| Weak entity | Table = weak attributes + foreign key to owner |
| Many-to-many | Junction table with both keys (+ relationship attrs) |
| Many-to-one / one-to-many | Put foreign key on “many” side |
| One-to-one | Foreign key on either side (design choice) |
| Specialization | Superclass table + subclass tables, OR one table per subclass with all attrs **[L]** |

---

## 8. Functional dependency and normal forms

**Level 2 only** for second and third normal form (tables exist). **[L]**

| Normal form | Violation | Fix |
|-------------|-----------|-----|
| **First** | Multi-valued column | Separate table or relationship |
| **Second** | Non-key attribute depends on **part** of composite key | Decompose (part–warehouse vs warehouse–address) **[L]** |
| **Third** | Non-key depends on **non-key** | Decompose (employee–department vs department–location) **[L]** |

**Trade-off:** normalization improves integrity; joins may hurt performance. **[L][P]**

**Entity-relationship vs normalization:** top-down design vs bottom-up decomposition from arbitrary relations. **[P]**

---

## 9. Generalization and specialization

- Subclass ⊆ superclass; specialization down, generalization up. **[P][L]**
- Disjoint vs overlap; total specialization forces every superclass member into some subclass. **[L]**
- Two table-mapping strategies (see primer Part 7). **[L]**

---

## 10. Integrity and data manipulation semantics

**Constraint types:** allowable domain values; permitted subset per attribute; constraints on stored data (inclusion, inequalities, aggregates). **[P]**

**Retrieval (level 1):** select values, entities, relationships, or attributes — expressible with set notation; maps to structured query language at level 2. **[P]**

**Insert / update / delete:** primary key changes propagate; delete owner may delete weak entities and related tuples. **[P]**

---

## 11. Network, relational, and entity set models

| Model | Main idea | Entity-relationship comparison |
|-------|-----------|--------------------------------|
| **Relational** | Relations over domains; roles disambiguate columns | Domains ≈ value sets; join ambiguity when semantics differ **[P]** |
| **Network** | Owner–member record sets; arrows = access + one-to-many | Arrows ≠ always entity relationships; many-to-many needs link record **[P][L]** |
| **Entity set** | All values are entities; binary links only | Entity-relationship keeps values as values; supports n-ary **[P]** |

**Deriving network from entity-relationship:** one-to-many → arrow; many-to-many or n-ary → relationship record type. **[P]**

---

## 12. Exam cheat sheet

1. **Two primitives:** entities and relationships. **[P][L]**
2. **Attribute = function** → one value per entity. **[P]**
3. **Double line** = total participation; **weak entity** = no key without owner. **[L][P]**
4. **Many-to-many** → junction table with combined key. **[L]**
5. **Second normal form / third normal form** fix redundancy; entity-relationship design often aligns with third normal form with clearer meaning. **[P][L]**
6. **Four levels:** conceptual (1) → tables (2) → relational (3) → network access (4). **[P]**
7. **Network arrow** = record-type one-to-many + access path, not always conceptual relationship. **[P]**
8. **Chen’s goal:** unified view of data + semantics for integrity and manipulation. **[P]**
