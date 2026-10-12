# Entity-Relationship Model — Complete Primer (read straight through)

**Paper:** Peter Chen, 1976. **[P]**  
**Lecture:** CSCI 585 `[PPT_Lecture.pdf](PPT_Lecture.pdf)`. **[L]**  
**Course intro:** `[Class_PPT.pdf](Class_PPT.pdf)`. **[C]**


| If you want…                             | Read…                                                |
| ---------------------------------------- | ---------------------------------------------------- |
| One continuous story (recommended first) | This file, Parts 0–12 in order                       |
| Shorter lookup sections                  | `[ER_Model_Study_Guide.md](ER_Model_Study_Guide.md)` |
| Drill                                    | `[ER_Model_Flashcards.md](ER_Model_Flashcards.md)`   |
| Mock exam                                | `[ER_Model_Exam_QA.md](ER_Model_Exam_QA.md)`         |


---



# Part 0 — Why databases and database management systems matter **[C]**

A **database** is an organized collection of data, usually kept on **secondary storage** (hard disks or solid-state drives), that describes the activities of one or more related organizations. Before computers, people kept ledgers and filing cabinets; the **database** was the information and the **database management system** was the set of rules and clerks who maintained it.

A **database management system** is software that helps you store, find, update, and protect large amounts of data. Modern applications (for example large social networks) have enormous **volume** (billions of users and connections) and **variety** (structured tables, text, images, and video). A database management system is what makes that manageable.

**Why use a database management system instead of ad hoc files?**

1. **Faster application development** — common tasks (search, update, concurrent access) are built in. **[C]**
2. **Data independence** — application programs do not need to know exactly how data is laid out on disk; the system hides those details. **[C]**
3. **Data sharing** — one coherent store for many users; redundant copies can be reduced. **[C]**
4. **Integrity and consistency** — you can enforce rules (for example: seats sold cannot exceed seats on the plane by more than a small tolerance). **[C]**
5. **Centralized control** — a **database administrator** can tune performance and policies for everyone. **[C]**
6. **Security** — access can depend on *what* the data is, not only which file it lives in. **[C]**
7. **Concurrency control** — many users at once without corrupting each other’s updates. **[C]**
8. **Crash recovery** — after a failure, data can be restored to a consistent state. **[C]**

When you **design** a database, you move through **levels of data models**:

- **Conceptual** — what exists in the real world (people, projects, who works on what).
- **Logical** — how that becomes tables, keys, and constraints (what Chen calls **level 2**).
- **Physical** — indexes, disk layout, access paths (Chen’s **levels 3 and 4**).

Chen’s entity-relationship model is mainly a **conceptual** tool (level 1) with a clear path to **logical tables** (level 2). **[P][L]**

---



# Part 1 — The big idea of the entity-relationship model **[P][L]**

Peter Chen’s paper (1976) argues that the real world is naturally described as **entities** (things you can tell apart) and **relationships** (connections among those things). Examples: a person named Joe, a football team, and “Joe is a fan of that team.”

The model was proposed to:

- Capture **meaning** (semantics) that pure tables sometimes hide.
- Support **database design** with diagrams.
- **Unify** three older approaches: the **network model**, the **relational model**, and the **entity set model**. **[P]**

You are not required to memorize every historical detail, but you should know: the entity-relationship view is often **top-down** (start from meaning, then build tables), while classical **normalization** in the relational world is often **bottom-up** (start from one big table and split it). **[P][L]**

---



# Part 2 — Level 1: entities, relationships, attributes, values **[P][L]**



## Entities and entity sets

An **entity** is something that exists and can be distinguished from other things. It can be concrete (a person) or abstract (a sports team). **[L]**

An **entity set** is a collection of entities of the same type (for example all employees, all books). Entity sets **do not have to be disjoint**: someone can be both a student and an employee; both sets are subsets of “people.” **[P][L]**

## Relationships and relationship sets

A **relationship** is an association among two or more entities (Joe fans the 49ers). **[L]**

A **relationship set** is the set of all relationships of the same type. Formally, if entity sets E_1, E_2, \ldots, E_n are involved, a relationship set R is a subset of combinations [e_1, e_2, \ldots, e_n] where each e_i comes from the matching entity set. **[P][L]**

- **Binary** — two entity sets (person – team).
- **Ternary** — three (person – team – date of becoming a fan).
- **N-ary** — n entity sets.

You can always replace a non-binary relationship with several binary ones, but one ternary relationship can carry meaning that several binaries do not (classic example: which supplier supplies which part to which project). **[P][L]**

## Roles

In a relationship, each entity plays a **role** (husband, wife, worker, project). If roles are named, the order of entities in the relationship does not have to be fixed. **[P]**

## Attributes, value sets, and values

Information about an entity or relationship is expressed as **attributes**. An attribute is a **function**: each entity maps to **one** value (or one tuple of values) in a **value set** (also called a **domain**). Joe’s social security number maps Joe to a number; his name might map to a pair (first name, last name). **[P][L]**

**Values** are things like `3`, `red`, `Peter`. **Value sets** group values (colors, years, names). The same value in different value sets can be equivalent (12 inches = 1 foot). **[P]**

Important: an attribute on a **relationship** belongs to that relationship, not to a single entity. Example: **percentage of time** an employee spends on a project is an attribute of the **assignment** relationship, not of employee alone or project alone. **[P]**

At level 1, **multi-valued attributes** are allowed in principle (one person, many phone numbers). **[L]**

---



# Part 3 — Diagrams: how to draw the model **[P][L]**

**Entity-relationship diagrams** use:

- **Rectangles** for entity sets.
- **Diamonds** for relationship sets.
- **Lines** connecting relationships to the entity sets they involve.
- **Ovals** (in many textbooks; the lecture stresses attributes on entities) for attributes.

**Participation constraints:**

- **Partial participation** — some entities in a set need not take part in the relationship (not every person must fan a team). Draw a **single line**. **[L]**
- **Total participation** — every entity in the set must participate (every dependent must belong to some employee). Draw a **double line** on that side. **[P][L]**

**Cardinality (mapping constraints)** on a binary relationship between set A and set B: **[L]**


| Pattern          | Meaning                                     | Example                         |
| ---------------- | ------------------------------------------- | ------------------------------- |
| **One-to-one**   | Each A maps to at most one B and vice versa | Women marry men (as a function) |
| **Many-to-one**  | Many A to one B                             | Children → one mother           |
| **One-to-many**  | One A to many B                             | Mother → many children          |
| **Many-to-many** | Many on both sides                          | Students enrolled in classes    |


On diagrams, **1**, **n**, or **m** label how many entities from each side can appear in one relationship instance. **[P][L]**

**Existence dependency** — if entity type B cannot exist without A (dependents without employees), an **arrow** toward the owner or special notation for **weak** entities is used. **[P]**

---



# Part 4 — Level 2: keys and representing entities as tables **[P][L]**

Level 2 replaces “entities in our minds” with **representations** — mainly **tables (relations)**.

## Keys (vocabulary you must know)

- **Key** — one or more attributes that identify an entity or relationship instance.
- **Superkey** — any set of attributes that uniquely identifies rows (may include extra columns).
- **Candidate key** — a **minimal** superkey (no proper subset is still a superkey).
- **Primary key** — the candidate key the designer chooses; should be **semantically meaningful** when possible. **[L]**

For a **relationship set** with no attributes of its own:

- **Many-to-many:** primary key is usually the **union** of the primary keys of all participating entity sets.
- **Many-to-one from E_1 to E_2:** primary key of the relationship is often just the primary key of the **“many”** side (E_1). **[L]**



## Weak and strong entity sets

- **Strong (regular) entity set** — has enough attributes to form its own primary key (author with social security number).
- **Weak entity set** — cannot form a primary key from its own attributes alone (book **edition** might depend on which book). It has an **owner** (identifying) entity set, an **identifying relationship** (often many weak to one owner), and **total participation** of weak entities in that relationship. **[P][L]**

The weak entity’s primary key = **partial key of weak entity** + **primary key of owner** (foreign key). Example: book identified by (author social security number, title, edition). **[L]**

**Foreign key** — in the relational table, the columns that reference the owner’s primary key. **[L]**

Chen also uses **regular** versus **weak entity relation** and **regular** versus **weak relationship relation** at level 2; weak forms appear when identification requires another entity’s key. **[P]**

---



# Part 5 — Worked design intuition (library and manufacturing) **[L][P]**

**Library example (lecture):** authors write books on subjects; libraries carry books. Entity sets: authors, subjects, books, libraries. Relationships: wrote, indexed, carry. Keys might include international standard book number (ISBN), subject matter, library address. **[L]**

**Manufacturing firm (Chen Figure 11):** departments, employees, dependents, projects, suppliers, parts; relationships such as works-on, manages, supplies, and **component** (a part built from sub-parts — relationship on **one** entity set). The diagram shows:

- Relationships on more than two entity sets.
- Relationships on a **single** entity set.
- **Multiple** relationship sets between the same pair of entity sets (worker vs manager on employee–project).
- Cardinalities one-to-many, many-to-many, one-to-one.
- Dependents **existence-dependent** on employees. **[P]**

**Reporting hierarchy (lecture exercise):** employees report to managers who are also employees, up to the chief executive officer. This is a **recursive** relationship on the employee entity set (same entity set twice). **[L]**

**Modern motivation (lecture):** fraud detection links users through shared internet protocol addresses, devices, payments; product recommendation graphs have hundreds of millions of nodes — still entity-relationship thinking at scale. **[L]**

---



# Part 6 — Functional dependency and normal forms (level 2 only) **[P][L]**

A **functional dependency** means:  if two rows agree on attribute set **X**, they must agree on **Y**. You cannot have the same **X** with two different **Y** values. Equivalently: **Y is a function of X**. **[L]**

At level 1, every attribute is already a function on entities. At level 2, **non-key attributes depend on the primary key**. **[P][L]**

**First normal form:** table cells hold atomic values — **no multi-valued attributes** in one column. Multi-valued attributes in the conceptual model must become a separate table or relationship. **[L]**

**Second normal form:** no non-key attribute may depend on **only part** of a **composite** primary key. Classic bad table: (part, warehouse, quantity, warehouse-address) with key (part, warehouse). Warehouse address repeats for every part in that warehouse; updates are painful; empty warehouse may disappear from the database. **Fix:** split into (part, warehouse, quantity) and (warehouse, warehouse-address). **[L][P]**

**Third normal form:** no non-key attribute may depend on **another non-key attribute**. Example: (employee, department, location) where location is really a fact about **department**, not about employee. Updating one employee’s “location” is ambiguous: did the person move departments or did the department move buildings? **Fix:** split employee–department and department–location. **[L][P]**

**Normalization** = replacing bad tables with normalized ones. **Benefit:** less redundancy, fewer update anomalies. **Cost:** more tables, so some queries need **joins** (slower). Example: “address of warehouse holding part 5” needs a join after decomposition. **[L]**

Chen’s point: entity-relationship design often **already** yields structures like **third normal form** tables, but with **clearer meaning** than starting from one wide table and decomposing. If business rules change (one project, many managers), the entity-relationship diagram changes naturally; a third-normal-form table design might need a painful redesign. **[P]**

**Top-down (entity-relationship) vs bottom-up (normalization):** level 1 concepts like second and third normal form “do not apply” because you are not in flat tables yet; at level 2 they do. **[L]**

---



# Part 7 — Generalization and specialization **[L][P]**

**Specialization** — define **subclasses** below a superclass (student → undergraduate, graduate). **Generalization** — merge subclasses into a higher-level entity set (the upward view). **[P][L]**

Rules:

- Every subclass member must be a member of the superclass.
- **Disjoint (d):** an entity can be in at most one subclass (undergraduate OR graduate, not both).
- **Overlap (o):** can be in multiple subclasses (salaried AND part-time).
- **Total specialization (double lines):** every superclass member must belong to **some** subclass (every employee must be salaried or part-time). **[L]**

**Mapping to tables (two approaches):** **[L]**

1. Table for superclass + each subclass table with superclass primary key plus subclass attributes.
2. No superclass table — each subclass table includes **all** attributes (name on student and employee tables).

---



# Part 8 — From entity-relationship diagram to relational tables **[L][P]**

**Strong entity set** with attributes a_1,\ldots,a_n → one table with those columns; primary key as designed.

**Weak entity set** → table includes weak attributes **plus** foreign key columns from the owner.

**Many-to-many relationship** → new table whose key is (usually) the combination of participating primary keys; include relationship attributes.

**One-to-one, many-to-one, one-to-many (functional relationships)** → often **merge** the foreign key into the table on the **“many”** side or the side that “receives” the function. Example: author writes book with one author per book → put author’s key in the book table. **[L]**

**Arity** = number of columns; **cardinality** of a relation = number of rows (do not confuse with relationship cardinality one-to-many). **[L]**

---



# Part 9 — Data integrity (what the system should enforce) **[P]**

Three kinds of constraints Chen discusses:

1. **Allowable values** — domain definition (employee number between 0 and 2000).
2. **Permitted values for an attribute** — stricter than domain (age between 20 and 65 for employees).
3. **Constraints among existing values** — set inclusions (names of male persons ⊆ names of persons) or rules like tax less than salary, or budget equals sum of department budgets. **[P]**

Insert, update, and delete have **consequences**: deleting an employee may delete dependents and all relationships involving that employee; changing a primary key must ripple to foreign keys. Entity-relationship semantics make those rules easier to state than in a bare relational algebra. **[P]**

---



# Part 10 — How entity-relationship relates to the network model **[P][L]**

The **network model** (Database Task Group style) uses **record types** (boxes) and **arrows** for **owner–member sets**. An arrow is **not** always “relationship between entities”:

- It often means **one-to-many between record types** plus an **access path** from owner to members (level 4 — physical/access-oriented). **[P]**
- **Many-to-many** binary relationships become a **relationship record type** with arrows from both entity record types. **[P][L]**

**Rules to derive network diagrams from entity-relationship diagrams:**

1. One-to-many binary → arrow from “one” to “many.”
2. Many-to-many binary → new relationship record type.
3. Ternary and higher → relationship record type.

Arrows are **one-directional** in the network; entity-relationship shows **both roles**. If a department–employee relationship becomes many-to-many, the network design must add a new record type; the entity-relationship diagram only changes a label. **[P]**

**Disciplined network design:** every entity and relationship becomes a record; every relationship gets its own record type even when one-to-many — more uniform, easier integrity rules. **[P]**

---



# Part 11 — How entity-relationship relates to the relational model **[P][L]**

A **relation** is a set of tuples over domains. **Domains** correspond to **value sets**. In relational theory, **roles** disambiguate two columns with the same domain name in one table (legal first name vs alternative first name). In the entity-relationship model, **attribute** means a function from an entity set to a value set — different semantics for the same word. **[P]**

**Join ambiguity:** joining employee and ship on “number of years” is meaningless even if column names match; entity-relationship exposes **age of employee** vs **age of ship**. The system could warn users if semantics differ. **[P]**

**Third normal form relations** from normalization often match **entity and relationship relations** from a diagram, but the entity-relationship version says *why* a column is there (role, relationship, entity). **[P]**

---



# Part 12 — Entity set model (brief) **[P][L]**

In the **entity set model**, even values like `22` or `blue` are treated as **entities** (name/value pairs like `QUANTITY/22`). Everything is linked by **binary** associations. The entity-relationship model treats `22` as a **value** in a value set, not as a first-class entity — usually more natural. **[P]**

You can **derive** the entity set view from entity-relationship figures by treating each value set as an “entity-name-set” and values as entity names; roles become how entities point to each other. N-ary relationships need artificial “relationship entities” in the entity set world. **[P]**

---



# Part 13 — Four levels of logical views (exam favorite) **[P]**


| Level | What it is                             | Typical models                               |
| ----- | -------------------------------------- | -------------------------------------------- |
| **1** | Entities and relationships in the mind | Entity-relationship, entity set (conceptual) |
| **2** | Information structure — tables, keys   | Entity-relationship, relational, entity set  |
| **3** | Access-path-independent structuresf    | Relational                                   |
| **4** | Access paths, indexes, record chains   | Network (as implemented)                     |


Chen’s figure: entity-relationship spans 1–2 cleanly; relational focuses on 2–3; network on 4; entity set on 1–2 with a different philosophy. **[P]**

---



# Part 14 — Design steps (what to do on an exam) **[P][L]**

1. Find **entity sets** and **relationship sets** for the application.
2. Mark **cardinalities**, **participation**, **weak entities**, **specialization**.
3. Define **attributes** and **value sets**.
4. Choose **primary keys**; build **entity and relationship relations** (tables).
5. Check **integrity constraints** and **normal forms** at level 2.

If you can narrate this pipeline in plain English, you can answer most CSCI 585 questions on Chen and the lecture deck.

---

**Next:** `[ER_Model_Study_Guide.md](ER_Model_Study_Guide.md)` for tables and cheat sheet · `[ER_Model_Exam_QA.md](ER_Model_Exam_QA.md)` for practice questions.