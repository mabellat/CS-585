# Entity-Relationship Model — Example Exam Q&A

**Sources:** Chen 1976 **[P]**, [`PPT_Lecture.pdf`](PPT_Lecture.pdf) **[L]**, [`Class_PPT.pdf`](Class_PPT.pdf) **[C]**.  
**Narrative answers:** **[ER_Model_Full_Primer.md](ER_Model_Full_Primer.md)**.

---

## Short answer

**1. What is a database management system and why use one?** **[C]**  
Software that manages large integrated data stores. Benefits include data independence, shared access with less redundancy, enforced integrity rules, security, concurrent access without corruption, and recovery after failures.

**2. State Chen’s two primitives.** **[P][L]**  
Entities (things distinguishable in the application) and relationships (associations among entities).

**3. What is total participation?** **[L]**  
Every entity in the given entity set must participate in at least one instance of the relationship. Shown with a double line from entity set to relationship.

**4. Difference between strong and weak entity set?** **[L][P]**  
Strong: primary key from its own attributes. Weak: cannot form a primary key alone; must link to an owner entity through an identifying relationship with total participation of weak entities.

**5. Primary key of a many-to-many relationship with no attributes?** **[L]**  
The combination of the primary keys of all participating entity sets.

**6. What is a functional dependency X → Y?** **[L]**  
Two rows cannot have the same X and different Y; Y is determined by X.

**7. Second normal form violation in one sentence?** **[L]**  
A non-key column depends only on part of a composite primary key, causing repetition and update problems.

**8. Third normal form violation in one sentence?** **[L]**  
A non-key column depends on another non-key column (transitive dependency), creating update ambiguity.

**9. What level does the network model emphasize?** **[P]**  
Level 4 — access-path-dependent structures (owner–member record sets).

**10. Does a network arrow always mean an entity relationship?** **[P]**  
No. It often means one-to-many between record types and an access path; many-to-many needs an extra record type.

---

## Medium answer

**11. Explain attribute on a relationship vs on an entity.** **[P][L]**  
An entity attribute describes one object (employee name). A relationship attribute describes the link and needs all participants (percentage of time for a specific employee on a specific project). Putting it on only employee or only project loses meaning.

**12. Why does Chen separate entity information from relationship information at level 1?** **[P]**  
It clarifies which attributes depend on which objects and exposes functional dependencies before turning the design into tables.

**13. Walk through mapping a many-to-many enrollment relationship to tables.** **[L]**  
Tables for students and classes with their own primary keys; a third enrollment table with student key, class key, and any relationship attributes (for example semester). Primary key of enrollment is typically both keys together.

**14. Part–warehouse example: what is wrong and how do you fix it?** **[L]**  
Table (part, warehouse, quantity, warehouse-address) with key (part, warehouse) repeats warehouse address and violates second normal form. Split into (part, warehouse, quantity) and (warehouse, warehouse-address).

**15. Employee–department–location: what ambiguity violates third normal form?** **[L]**  
Location describes department, not employee. Updating one employee’s location could mean the person changed departments or the department moved — hard to keep consistent. Split into employee–department and department–location.

**16. Two ways to map specialization to tables.** **[L]**  
(1) Superclass table plus subclass tables with superclass key; (2) no superclass table — each subclass table includes superclass attributes.

**17. How does entity-relationship design relate to third normal form?** **[P]**  
Entity and relationship relations from a good diagram are often already like third normal form, but with explicit roles and meaning; normalization instead starts from arbitrary tables and decomposes bottom-up.

**18. Compare entity-relationship and entity set model on values.** **[P]**  
Entity-relationship: “25” is a value in a value set. Entity set model: “NO-OF-YEARS/25” is an entity linked to others — can blur semantics.

---

## Long / design questions

**19. Design sketch: authors, books (one author per book), libraries carrying books by subject.** **[L]**  
Entity sets: author, book, subject, library. Relationships: wrote (author–book), indexed or subject link (book–subject), carry (library–book) possibly ternary with subject. Keys: author social security or author identifier, book ISBN, library address. Many-to-many between library and book if libraries hold many books.

**20. Recursive “reports to” among employees.** **[L]**  
One entity set employee; binary relationship reports_to linking employee (subordinate) to employee (manager); cardinality many-to-one from subordinate to manager; chief executive has no manager (participation partial on manager side for top role).

**21. Derive network structure from entity-relationship for department–employee one-to-many.** **[P][L]**  
Department record type as owner; employee as member; arrow from department to employees. If relationship becomes many-to-many, add department–employee link record type with arrows from both.

**22. Insert/delete semantics for employee with dependents.** **[P]**  
Deleting employee should delete dependent entities (existence dependency) and relationship tuples involving that employee; updating employee primary key must update foreign keys in dependent and relationship tables.

**23. Four levels of views — place relational and entity-relationship models.** **[P]**  
Entity-relationship: strong at levels 1–2. Relational: 2–3. Network implementations: 4. Entity set: 1–2 with different value philosophy.

---

## Oral “one minute” pitches

| Topic | Pitch |
|-------|--------|
| Paper contribution | Unified semantic model + diagrams + path to relational and network implementations. **[P]** |
| Lecture emphasis | Keys, weak entities, normal forms as level-2 issues, table generation rules. **[L]** |
| Course intro | Why database management systems matter for real applications at scale. **[C]** |
