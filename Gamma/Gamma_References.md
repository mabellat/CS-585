# Gamma — References

Numbered sources used in study docs. The IEEE TKDE survey paper is **`[P]`**. The CSCI 585 lecture deck is **`[L]`**. Earlier VLDB/SIGMOD papers are cited by the survey.

| ID | Source |
|----|--------|
| **P** | David J. DeWitt, Shahram Ghandeharizadeh, Donovan A. Schneider, Allan Bricker, Hui-I Hsiao, Rick Rasmussen. *The Gamma Database Machine Project.* **IEEE TKDE** 2(1): 44–62, **March 1990**. Local copy: [`The_Gamma_database_machine_project.pdf`](The_Gamma_database_machine_project.pdf). [Wisconsin PDF](https://pages.cs.wisc.edu/~dewitt/includes/paralleldb/ieee90.pdf) |
| **L** | CSCI 585 lecture: *Gamma*, DeWitt et al. Local copy: [`GammaLecture.pdf`](GammaLecture.pdf) (132 slides). |
| **R1** | DeWitt et al. *GAMMA - A High Performance Dataflow Database Machine.* **VLDB 1986**. Dataflow + ~3 control messages per operator per site. |
| **R2** | DeWitt et al. *A Performance Analysis of the Gamma Database Machine.* **SIGMOD 1988**. |
| **R3** | Michael Stonebraker. *The Case for Shared Nothing.* Database Engineering 9(1), 1986. |
| **R4** | Schneider & DeWitt. Parallel join algorithms (Hybrid, Grace, Sort-Merge). **SIGMOD 1989** [SCHN89]. |
| **R5** | Hsiao et al. *Chained declustering.* **IEEE TKDE** 1990 [HSIA90] — cited in survey §5.3. |
| **R6** | Copeland et al. *Data Placement in Bubba.* **SIGMOD 1988** — heat / partial declustering contrast. |
| **R7** | Bitton, DeWitt, Turbyfill. *Benchmarking Database Systems.* **VLDB 1983** — Wisconsin benchmark. |
| **R8** | Chou et al. *Wisconsin Storage System (WiSS).* Software Practice & Experience 1985. |

**Note:** The bundled `The_Gamma_database_machine_project.pdf` has no text layer in extraction tools; study content is aligned to the Wisconsin-hosted PDF text and the lecture deck.

**Pedagogical SQL/relational-algebra slides** in `GammaLecture.pdf` (Emp/Dept examples) are lecture review **`[L]`**, not separate publications.
