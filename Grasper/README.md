# Grasper — Paper & lecture study materials

**Paper:** Hongzhi Chen, Changji Li, Juncheng Fang, Chenghuan Huang, James Cheng, et al. *Grasper: A High Performance Distributed System for OLAP on Property Graphs.* **SoCC 2019.**  
**Course context:** CSCI 585 (Shahram Ghandeharizadeh) — lecture deck: [`../GrasperLecture.pdf`](../GrasperLecture.pdf) · Exam 1 includes Grasper among six syllabus papers.

**Primary links:** [SoCC PDF](http://www.cse.cuhk.edu.hk/systems/grasper/papers/Grasper_SoCC19.pdf) · [Project site](http://www.cse.cuhk.edu.hk/systems/grasper/) · [GitHub](https://github.com/yaobaiwei/Grasper)

---

## Suggested study order (exam prep)

| Step | Focus | File |
|------|--------|------|
| **1** | Big picture + architecture + storage split | [`Grasper_Study_Guide.md`](Grasper_Study_Guide.md) §1–7 |
| **2** | Gremlin + query engine + Expert Model | [`Grasper_Study_Guide.md`](Grasper_Study_Guide.md) §8–12 |
| **3** | Teach-yourself Q&A | [`Grasper_Full_Primer.md`](Grasper_Full_Primer.md) |
| **4** | Drill + mock exam | [`Grasper_Flashcards.md`](Grasper_Flashcards.md) + [`Grasper_Exam_QA.md`](Grasper_Exam_QA.md) |
| **5** | Present like the lecture (45 min) | [`Grasper_Lecture_Slides.md`](Grasper_Lecture_Slides.md) |

**Optional (15 min):** Load the air-routes graph in Gremlin Console (steps in lecture slide 21) so Gremlin examples feel concrete.

---

## Files in this folder

| File | Purpose |
|------|---------|
| **Grasper_Study_Guide.md** | Structured reference: PG model, Gremlin, RDMA, storage, query engine, experiments, limitations |
| **Grasper_Full_Primer.md** | Concepts from zero (Q&A, analogies, walkthrough of lecture + paper) |
| **Grasper_Lecture_Slides.md** | Short slide bullets mapped to `GrasperLecture.pdf` + timed outline |
| **Grasper_Flashcards.md** | Terms and one-liners for closed-book drill |
| **Grasper_Exam_QA.md** | **Example exam/oral questions with model answers** (short + long form) |
| **Grasper_References.md** | Citations and claim index |
