# Grasper — Flashcards

Format: **Q** in left column mentally; answer on the right.

---

## Deck 1 — Big picture

| Q | A |
|---|---|
| What is Grasper? | Distributed **in-memory OLAP** on **property graphs** with **Gremlin**, **RDMA**, **Expert Model** |
| OLTP or OLAP? | **OLAP** (interactive analytical traversals) |
| Key paper venue? | **SoCC 2019** |
| vs JanusGraph+FDB? | Grasper **co-locates** native store + engine; JanusGraph **external** transactional KV |
| Why RDMA? | Cut **network** cost for many **small** remote reads/messages |
| Load input? | **HDFS** → hash partition across engines |

---

## Deck 2 — Property graph & Gremlin

| Q | A |
|---|---|
| PG components? | **V, E**, labels, **KV properties**, directed edges, flexible schema |
| Gremlin style? | **Procedural** traversal steps |
| SQL vs Gremlin (2)? | Declarative/set vs procedural/**streaming** traversals |
| `out('knows')` step type? | **flatMap** (expand neighbors) |
| `has('age',20)`? | **Filter** |
| Simple vs complex query? | Few indexed verts vs **large** graph touch |
| Cypher used by? | **Neo4j** |
| SPARQL used for? | **RDF** triples |

---

## Deck 3 — RDMA

| Q | A |
|---|---|
| Ethernet copies? | **4×** (user/kernel/NIC both sides) |
| RDMA data path? | **Zero-copy** registered memory |
| One-sided verb? | Remote **read/write** without remote CPU |
| IPoIB problem? | Brings back **IP/socket** copies |
| Grasper verb choice? | **One-sided** read/write |
| “Order of magnitude” (latency)? | ~**10×** better than Ethernet for small msgs (paper) |

---

## Deck 4 — Storage

| Q | A |
|---|---|
| Normal memory holds? | **Topology**, adjacency, intermediates, indexes |
| RDMA memory holds? | **Labels + property values** (distributed KV) |
| Why split topology/properties? | Avoid **skew**; **balance** CPU; RDMA hides distance |
| Cost of split? | **Remote RDMA** for properties / some lookups |
| Adjacency lists? | **in** and **out** per vertex |
| String labels on wire? | Mapped to **integer IDs** |
| V-KVS / E-KVS? | Vertex vs edge property stores in RDMA mem |

---

## Deck 5 — Query engine

| Q | A |
|---|---|
| Execution model? | **Data-flow** of **messages** between steps |
| Message too big? | **Split** into multiple messages |
| Sequential flow? | Step in → step out (`in`, `out`, `has`) |
| Barrier flow? | **Collect + aggregate** (`count`, `max`) |
| Branch flow? | **Fork** sub-queries (`and`, `or`, `union`) |
| Expert? | Operator for one **step category** with shared cache |
| Adaptive parallelism metric? | **Number of messages** at step |
| Filter cache policy (example)? | **LRU** |
| Traversal cache (example)? | **FIFO** |
| Query plan opts (3)? | **Decompose, combine, reorder** |
| Straggler? | Slowest task delaying whole query |
| Work stealing order? | Same region → other region → **other nodes** |
| Thread binding why? | Avoid **context switch**, improve **locality** |

---

## Deck 6 — Evaluation & limits

| Q | A |
|---|---|
| Cluster size (lecture)? | **10** engines |
| RDMA card? | Mellanox ConnectX-3 **40 Gbps** |
| Threads per machine? | **24** |
| Benchmarks? | **LDBC**, **AMiner**, **Twitter** |
| Ablation factors? | **RDMA**, **query opt**, **work stealing** |
| Scale-up shape? | **Sublinear** (e.g. 7× on 10 nodes) |
| Twitter on 2 nodes? | **Out of memory** |
| Lecture limitations? | **Faults**, **writes/ACID**, **durability** |
