# 3E.3 — Neo4j Data Modeling & Graph Algorithms 🔴🔥

> **"A well-modeled graph doesn't just store data — it tells a STORY. And graph algorithms don't just find answers — they reveal HIDDEN truths."**

---

## 📌 What You'll Master in This Chapter

- **Graph data modeling** from scratch — how to think in graphs
- Converting **relational schemas** to graph models
- **Common graph modeling patterns** (and anti-patterns)
- **Graph Data Science (GDS) Library** — the algorithm powerhouse
- **Centrality algorithms** — who's most important? (PageRank, Betweenness)
- **Community detection** — finding clusters (Louvain, Label Propagation)
- **Path finding** — optimal routes (Dijkstra, A*, BFS/DFS)
- **Similarity** — finding lookalikes (Jaccard, Cosine, Overlap)
- **Link Prediction** — predicting future connections
- **Graph Embeddings** — vectors from graph structure (Node2Vec, GraphSAGE)
- Real-world case studies with full working examples

---

## 🎨 PART 1: GRAPH DATA MODELING

---

## 🧠 1. How to Think in Graphs — The Mindset Shift

### The Relational Mindset vs The Graph Mindset

```
┌─────────────────────────────────────────────────────────────┐
│  RELATIONAL THINKING              GRAPH THINKING             │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  "What TABLES do I need?"    →   "What THINGS exist?"        │
│  "What COLUMNS per table?"   →   "What PROPERTIES do         │
│                                     things have?"              │
│  "What FOREIGN KEYS?"        →   "How are things             │
│                                     CONNECTED?"                │
│  "What JOIN tables?"         →   "What RELATIONSHIPS          │
│                                     exist?"                    │
│  "How do I normalize?"       →   "What QUESTIONS will         │
│                                     I ask?"                    │
│                                                               │
│  Start with: STRUCTURE       →   Start with: USE CASES       │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### The Whiteboard-to-Graph Rule

> **If you can draw it on a whiteboard, you can model it in Neo4j — almost 1:1.**

```
WHITEBOARD DRAWING:                    CYPHER MODEL:

  ┌──────┐                             (:Person {name: "Alice"})
  │Alice │───KNOWS──→┌──────┐                │
  └──────┘           │ Bob  │          [:KNOWS {since: 2020}]
                     └──┬───┘                │
                        │              (:Person {name: "Bob"})
                     WORKS_AT                │
                        │              [:WORKS_AT {role: "Engineer"}]
                        ▼                    │
                   ┌────────┐          (:Company {name: "Google"})
                   │ Google │
                   └────────┘

  Circles/boxes → Nodes
  Arrows        → Relationships
  Labels on arrows → Relationship types
  Details       → Properties

  That's it! Your whiteboard IS your data model.
```

---

## 🔄 2. Converting Relational to Graph — Step by Step

### The Mapping Rules

```
┌──────────────────────────────────────────────────────────────┐
│  RELATIONAL → GRAPH MAPPING                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Entity Table     →  NODE with a LABEL                        │
│    (employees)       (:Employee)                               │
│                                                                │
│  Row              →  NODE INSTANCE                             │
│    (1 employee)      (one :Employee node)                     │
│                                                                │
│  Column           →  PROPERTY                                  │
│    (name, age)       {name: "...", age: ...}                  │
│                                                                │
│  Foreign Key      →  RELATIONSHIP                              │
│    (dept_id)         -[:WORKS_IN]->                            │
│                                                                │
│  JOIN Table (M:N) →  RELATIONSHIP (with properties!)          │
│    (student_course)  -[:ENROLLED_IN {grade: "A"}]->           │
│                                                                │
│  Columns in JOIN  →  PROPERTIES on the RELATIONSHIP           │
│    table             (grade, enrolled_date on ENROLLED_IN)    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Full Example: University Database

**Relational Schema:**

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│    students      │     │  enrollments     │     │    courses       │
├──────────────────┤     ├──────────────────┤     ├──────────────────┤
│ id (PK)          │     │ student_id (FK)  │     │ id (PK)          │
│ name             │←────│ course_id (FK)   │────→│ title            │
│ email            │     │ grade            │     │ department       │
│ gpa              │     │ semester         │     │ credits          │
└──────────────────┘     └──────────────────┘     └──────────────────┘
         │                                                  │
         │ department_id (FK)                                │ taught_by (FK)
         ▼                                                  ▼
┌──────────────────┐                             ┌──────────────────┐
│   departments    │                             │   professors     │
├──────────────────┤                             ├──────────────────┤
│ id (PK)          │                             │ id (PK)          │
│ name             │                             │ name             │
│ building         │                             │ title            │
└──────────────────┘                             └──────────────────┘
```

**Graph Model:**

```
   (:Student)                    (:Course)                  (:Professor)
  ┌───────────┐               ┌───────────┐              ┌───────────┐
  │ name      │──ENROLLED_IN─→│ title     │←─TEACHES────│ name      │
  │ email     │  grade: "A"   │ credits   │              │ title     │
  │ gpa       │  semester:"F26"│           │              │           │
  └─────┬─────┘               └───────────┘              └───────────┘
        │                           │
   BELONGS_TO               OFFERED_BY
        │                           │
        ▼                           ▼
   (:Department)              (:Department)
  ┌───────────┐              (same node!)
  │ name      │
  │ building  │
  └───────────┘
```

**Cypher to create this:**

```cypher
// Create nodes
CREATE (s:Student {name: "Ritesh", email: "r@uni.edu", gpa: 3.8})
CREATE (c:Course {title: "Database Systems", credits: 4})
CREATE (p:Professor {name: "Dr. Codd", title: "Professor"})
CREATE (d:Department {name: "Computer Science", building: "Engineering Hall"})

// Create relationships
CREATE (s)-[:ENROLLED_IN {grade: "A", semester: "Fall 2026"}]->(c)
CREATE (p)-[:TEACHES {since: 2020}]->(c)
CREATE (s)-[:BELONGS_TO]->(d)
CREATE (c)-[:OFFERED_BY]->(d)
```

> 💡 **Key Insight**: The JOIN table `enrollments` **disappears** — it becomes the `ENROLLED_IN` relationship with properties `grade` and `semester`. This is one of graph modeling's biggest wins: join tables become first-class relationships.

---

## 📐 3. Graph Modeling Patterns — Best Practices

### Pattern 1: Intermediate Nodes (Reifying Relationships)

When a relationship has **complex data** or **connects to other things**, promote it to a node.

```
❌ BEFORE (relationship with too many properties):

  (Alice)-[:REVIEWED {stars: 5, text: "Amazing!", date: "2026-06-02",
                       helpful_votes: 42, verified: true}]->(Product)

✅ AFTER (intermediate node):

  (Alice)-[:WROTE]->(review:Review {stars: 5, text: "Amazing!",
                                     date: "2026-06-02",
                                     helpful_votes: 42,
                                     verified: true})-[:ABOUT]->(Product)

Why? Now you can:
  • Query reviews independently: MATCH (r:Review) WHERE r.stars = 5
  • Connect reviews to other things: (r)-[:FLAGGED_BY]->(moderator)
  • Add comments to reviews: (r)<-[:COMMENTED_ON]-(comment)
```

### Pattern 2: Linked List (Temporal Chains)

For **ordered sequences** like employment history, event logs, or versioning:

```
  (:Person {name: "Ritesh"})
       │
    [:CURRENT_JOB]
       │
       ▼
  (:Job {title: "Architect", company: "Neo4j"})
       │
    [:PREVIOUS]
       │
       ▼
  (:Job {title: "Senior Dev", company: "Oracle"})
       │
    [:PREVIOUS]
       │
       ▼
  (:Job {title: "Developer", company: "TCS"})

Cypher:
  MATCH (p:Person {name:"Ritesh"})-[:CURRENT_JOB]->(current:Job)
        -[:PREVIOUS*]->(past:Job)
  RETURN current, collect(past) AS history
```

### Pattern 3: Timeline Tree (Temporal Indexing)

For **time-based data** that needs efficient range queries:

```
                    (:Year {value: 2026})
                   /                     \
            [:HAS_MONTH]            [:HAS_MONTH]
             /                           \
    (:Month {value: 6})          (:Month {value: 7})
       /          \
  [:HAS_DAY]  [:HAS_DAY]
    /              \
(:Day {value: 1}) (:Day {value: 2})
    |                |
[:HAS_EVENT]    [:HAS_EVENT]
    |                |
(:Event {...})  (:Event {...})

Query all events in June 2026:
  MATCH (:Year {value:2026})-[:HAS_MONTH]->(:Month {value:6})
        -[:HAS_DAY]->(:Day)-[:HAS_EVENT]->(e:Event)
  RETURN e
```

### Pattern 4: Multi-Labeled Nodes (Role-Based)

A single entity can play **multiple roles**:

```cypher
// A person can be an Actor AND Director AND Producer
CREATE (p:Person:Actor:Director {name: "Clint Eastwood"})

// Query by any role
MATCH (d:Director)-[:DIRECTED]->(m:Movie)
WHERE d:Actor  // directors who are also actors
RETURN d.name, m.title
```

### Pattern 5: Hyperedges via Intermediate Nodes

When **one event connects 3+ entities** simultaneously:

```
  ❌ Can't do: (Alice)-[:MEETING {with: Bob, about: Project}]->(Calendar)
     (Relationships connect exactly 2 nodes!)

  ✅ Solution: Create an event node

  (Alice)-[:ATTENDED]->(:Meeting {date: "2026-06-02",
                                   topic: "Sprint Review"})
                                       ↑               ↑
                          [:ATTENDED]──┘    [:ABOUT]───┘
                              ↑                        ↑
                          (Bob)               (:Project {name: "Atlas"})
```

---

## ❌ 4. Graph Modeling Anti-Patterns — What NOT to Do

```
┌─────────────────────────────────────────────────────────────┐
│  ANTI-PATTERN 1: Rich Properties Instead of Relationships   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ❌ WRONG:                                                    │
│  (:Person {name: "Alice", employer: "Google",                │
│            manager: "Bob", department: "Engineering"})        │
│                                                               │
│  ✅ RIGHT:                                                    │
│  (Alice)-[:WORKS_AT]->(Google)                               │
│  (Alice)-[:REPORTS_TO]->(Bob)                                │
│  (Alice)-[:MEMBER_OF]->(Engineering)                         │
│                                                               │
│  Rule: If a property value is an ENTITY, make it a NODE.    │
│                                                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ANTI-PATTERN 2: Generic Relationship Types                  │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ❌ WRONG:                                                    │
│  (a)-[:RELATES_TO {type: "friend"}]->(b)                    │
│  (a)-[:RELATES_TO {type: "colleague"}]->(c)                 │
│                                                               │
│  ✅ RIGHT:                                                    │
│  (a)-[:FRIEND_OF]->(b)                                      │
│  (a)-[:COLLEAGUE_OF]->(c)                                   │
│                                                               │
│  Rule: Use SPECIFIC relationship types.                      │
│  Neo4j filters by type before traversal = way faster.       │
│                                                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ANTI-PATTERN 3: Dense Nodes (Super Nodes)                   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Problem: A node with MILLIONS of relationships             │
│  Example: (:Country {name: "India"}) with 1.4 billion       │
│           LIVES_IN relationships                             │
│                                                               │
│  Why it's bad: Traversing ALL relationships of a super      │
│  node is slow (must iterate through the linked list)        │
│                                                               │
│  Solutions:                                                  │
│  1. Add intermediate nodes:                                  │
│     (Person)-[:LIVES_IN]->(:State)-[:IN]->(:Country)       │
│  2. Use relationship properties to filter early             │
│  3. Use fanout nodes (bucketing)                            │
│                                                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ANTI-PATTERN 4: Using Neo4j as a Relational Database       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ❌ WRONG:                                                    │
│  Creating "table-like" structures:                           │
│  (:Row {col1: "a", col2: "b", col3: "c"})                  │
│  (:Row {col1: "d", col2: "e", col3: "f"})                  │
│  No relationships at all!                                    │
│                                                               │
│  If your data has NO meaningful relationships,              │
│  you probably don't need a graph database.                  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 5. Data Modeling — Decision Framework

```
              "What should this be?"

         ┌───────────────────────────────┐
         │ Is it an entity/thing that    │
         │ exists independently?         │
         └──────────────┬────────────────┘
                        │
              ┌────Yes──┴──No────┐
              ▼                   ▼
        ┌──────────┐     ┌───────────────────┐
        │  NODE    │     │ Is it a connection │
        │ (:Label) │     │ between 2 things? │
        └──────────┘     └────────┬──────────┘
                                  │
                        ┌───Yes───┴──No────┐
                        ▼                   ▼
                  ┌──────────┐     ┌──────────────┐
                  │RELATIONSHIP│    │  PROPERTY    │
                  │ [:TYPE]   │    │  {key: val}  │
                  └─────┬─────┘    └──────────────┘
                        │
                ┌───────┴────────────────┐
                │ Does this connection   │
                │ have complex data OR   │
                │ connect to other things?│
                └───────┬────────────────┘
                        │
              ┌───Yes───┴──No────┐
              ▼                   ▼
     ┌──────────────┐    ┌──────────────┐
     │ INTERMEDIATE │    │ Keep as      │
     │    NODE      │    │ RELATIONSHIP │
     │ (reify it)   │    │ with props   │
     └──────────────┘    └──────────────┘
```

---

## 🔬 PART 2: GRAPH ALGORITHMS (Graph Data Science)

---

## 🧪 6. The GDS Library — Overview

The **Graph Data Science (GDS)** library is Neo4j's algorithm powerhouse — 60+ algorithms for analyzing graph structures.

### Installing GDS

```
Neo4j Desktop:  Click "Add Plugin" → Graph Data Science → Install
Docker:         NEO4J_PLUGINS=["graph-data-science"]
AuraDB:         GDS is available on AuraDB Professional/Enterprise
```

### The GDS Workflow (Every Algorithm Follows This)

```
┌────────────────────────────────────────────────────────────┐
│  THE GDS WORKFLOW (3 Steps)                                 │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  STEP 1: PROJECT A GRAPH (in-memory)                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ CALL gds.graph.project(                              │   │
│  │   'myGraph',           ← name                        │   │
│  │   'Person',            ← node label(s)               │   │
│  │   'KNOWS'              ← relationship type(s)        │   │
│  │ )                                                     │   │
│  └──────────────────────────────────────────────────────┘   │
│         │                                                    │
│         ▼                                                    │
│  STEP 2: RUN ALGORITHM                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Three execution modes:                               │   │
│  │                                                       │   │
│  │ .stream()  → Returns results as rows (read-only)     │   │
│  │ .write()   → Writes results back to Neo4j nodes      │   │
│  │ .mutate()  → Writes results to in-memory graph only  │   │
│  │ .stats()   → Returns summary statistics only         │   │
│  └──────────────────────────────────────────────────────┘   │
│         │                                                    │
│         ▼                                                    │
│  STEP 3: CLEAN UP                                           │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ CALL gds.graph.drop('myGraph')                       │   │
│  │ → Free the in-memory projection                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

### GDS Algorithm Categories at a Glance

```
┌─────────────────────────────────────────────────────────────┐
│  GDS ALGORITHM CATEGORIES                                    │
├──────────────────┬──────────────────────────────────────────┤
│  CATEGORY        │  ALGORITHMS                               │
├──────────────────┼──────────────────────────────────────────┤
│  CENTRALITY      │  PageRank, Betweenness, Closeness,       │
│  "Who matters?"  │  Degree, Eigenvector, ArticleRank        │
├──────────────────┼──────────────────────────────────────────┤
│  COMMUNITY       │  Louvain, Label Propagation, WCC, SCC,   │
│  "Who belongs    │  K-1 Coloring, Modularity Optimization   │
│   together?"     │                                           │
├──────────────────┼──────────────────────────────────────────┤
│  PATH FINDING    │  Dijkstra, A*, Yen's K-Shortest Paths,  │
│  "How to get     │  BFS, DFS, Random Walk, Minimum          │
│   from A to B?"  │  Spanning Tree                           │
├──────────────────┼──────────────────────────────────────────┤
│  SIMILARITY      │  Jaccard, Cosine, Overlap, Pearson,      │
│  "Who is like    │  Euclidean, Node Similarity              │
│   whom?"         │                                           │
├──────────────────┼──────────────────────────────────────────┤
│  LINK PREDICTION │  Adamic Adar, Common Neighbors,          │
│  "What will      │  Preferential Attachment, Resource       │
│   connect next?" │  Allocation, Same Community              │
├──────────────────┼──────────────────────────────────────────┤
│  EMBEDDINGS      │  Node2Vec, FastRP, GraphSAGE,            │
│  "Graph → Vector │  HashGNN                                  │
│   for ML"        │                                           │
├──────────────────┼──────────────────────────────────────────┤
│  TOPOLOGICAL     │  Triangle Count, Local Clustering        │
│  "Graph shape?"  │  Coefficient, K-Core Decomposition       │
└──────────────────┴──────────────────────────────────────────┘
```

---

## 👑 7. Centrality Algorithms — Finding Important Nodes

### 7.1 PageRank — "Google's Algorithm"

Originally invented by Larry Page & Sergey Brin to rank web pages. Now used everywhere.

**Concept**: A node is important if it's pointed to by OTHER important nodes. Recursive definition!

```
How PageRank Works (Intuition):

  Imagine a "random surfer" clicking links randomly.
  PageRank = probability that the surfer lands on a given page.

  ┌──────┐  0.4   ┌──────┐  0.15  ┌──────┐
  │  A   │───────→│  B   │───────→│  C   │
  │ PR:  │        │ PR:  │        │ PR:  │
  │ 0.15 │←──┐    │ 0.25 │        │ 0.35 │  ← C has highest PR
  └──────┘   │    └──┬───┘        └──┬───┘    because B AND D
             │       │    0.3        │        both point to it
             │       ▼               │
             │  ┌──────┐            │
             └──│  D   │────────────┘
                │ PR:  │  0.25
                │ 0.25 │
                └──────┘
```

**Cypher + GDS:**

```cypher
// Step 1: Project the graph
CALL gds.graph.project(
  'social',                 // graph name
  'Person',                 // node labels
  'FOLLOWS'                 // relationship types
)

// Step 2: Run PageRank (stream mode — read results)
CALL gds.pageRank.stream('social')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS person, 
       round(score, 4) AS pageRank
ORDER BY pageRank DESC
LIMIT 10

// Step 2 (alt): Write results back to nodes
CALL gds.pageRank.write('social', {
  writeProperty: 'pageRankScore'
})

// Now you can query it directly
MATCH (p:Person)
RETURN p.name, p.pageRankScore
ORDER BY p.pageRankScore DESC

// Step 3: Clean up
CALL gds.graph.drop('social')
```

**Real-World Uses:**

| Domain | What PageRank Reveals |
|--------|----------------------|
| Social Network | Most influential people |
| Web | Most authoritative pages (Google) |
| Academic | Most cited papers/researchers |
| Finance | Most systemically important banks |
| IT Ops | Most critical servers (if they fail, cascade!) |

### 7.2 Betweenness Centrality — "The Bridge Finder"

**Concept**: Nodes that sit on **many shortest paths** between other nodes. They're the "bridges" or "gatekeepers."

```
  ┌───┐     ┌───┐     ┌───┐
  │ A │─────│ B │─────│ C │
  └───┘     └─┬─┘     └───┘
              │
         HIGH BETWEENNESS!
         B bridges two groups
              │
  ┌───┐     ┌┴──┐     ┌───┐
  │ D │─────│ B │─────│ E │
  └───┘     └───┘     └───┘

  Remove B → the graph splits into disconnected parts!
```

```cypher
// Run Betweenness Centrality
CALL gds.betweenness.stream('social')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS person,
       round(score, 4) AS betweenness
ORDER BY betweenness DESC
LIMIT 10
```

**Real-World Uses:**

| Domain | What Betweenness Reveals |
|--------|-------------------------|
| Social Network | Key connectors between groups (brokers) |
| IT Network | Single points of failure (routers/switches) |
| Supply Chain | Critical logistics hubs |
| Organization | Information bottlenecks |

### 7.3 Degree Centrality — "The Popularity Contest"

The simplest centrality — just count connections.

```cypher
// In-degree (who gets the most incoming connections)
CALL gds.degree.stream('social', {orientation: 'REVERSE'})
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS person,
       toInteger(score) AS followers
ORDER BY followers DESC
LIMIT 10

// Out-degree (who connects to the most others)
CALL gds.degree.stream('social', {orientation: 'NATURAL'})
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS person,
       toInteger(score) AS following
ORDER BY following DESC
```

### Centrality Comparison — When to Use Which

```
┌──────────────────────────────────────────────────────────────┐
│  "I want to find..."          →  "Use this algorithm"        │
├──────────────────────────────────────────────────────────────┤
│  Most popular person           →  Degree Centrality          │
│  Most influential person       →  PageRank                   │
│  Key bridge / gatekeeper       →  Betweenness Centrality     │
│  Most "central" overall        →  Closeness Centrality       │
│  Who influences influencers    →  Eigenvector Centrality     │
└──────────────────────────────────────────────────────────────┘
```

---

## 🏘️ 8. Community Detection — Finding Clusters

### 8.1 Louvain Modularity — "The Cluster Finder"

**Concept**: Groups nodes into communities where connections within the group are dense, and connections between groups are sparse.

```
BEFORE Louvain:                    AFTER Louvain:

  A───B    E───F                   ┌─────────┐   ┌─────────┐
  │\ /│    │\ /│                   │Community│   │Community│
  │ X │    │ X │                   │    1    │   │    2    │
  │/ \│    │/ \│                   │  A B    │   │  E F    │
  C───D────G───H                   │  C D    │   │  G H    │
                                   └────┬────┘   └────┬────┘
  (Who belongs together?)                │             │
                                    weak connection(D─G)
```

```cypher
// Project graph for community detection
CALL gds.graph.project('social', 'Person', {
  KNOWS: {orientation: 'UNDIRECTED'}
})

// Run Louvain
CALL gds.louvain.stream('social')
YIELD nodeId, communityId
RETURN gds.util.asNode(nodeId).name AS person,
       communityId
ORDER BY communityId, person

// Write communities back to nodes
CALL gds.louvain.write('social', {
  writeProperty: 'community'
})

// Now find the communities
MATCH (p:Person)
RETURN p.community AS community,
       count(p) AS size,
       collect(p.name) AS members
ORDER BY size DESC

// Clean up
CALL gds.graph.drop('social')
```

**Real-World Uses:**

| Domain | What Communities Reveal |
|--------|------------------------|
| Social Media | Friend groups, interest clusters |
| Fraud Detection | Fraud rings (tightly connected suspicious accounts) |
| Marketing | Customer segments for targeted campaigns |
| Biology | Protein interaction clusters |
| Research | Academic research clusters/schools of thought |

### 8.2 Label Propagation — "The Fast Cluster Finder"

Faster than Louvain but less precise. Each node adopts the label of its majority neighbors.

```cypher
CALL gds.labelPropagation.stream('social')
YIELD nodeId, communityId
RETURN gds.util.asNode(nodeId).name AS person,
       communityId
ORDER BY communityId
```

### 8.3 Weakly Connected Components (WCC)

Finds **disconnected subgraphs** — groups that have NO path between them.

```cypher
CALL gds.wcc.stream('social')
YIELD nodeId, componentId
RETURN componentId, 
       count(*) AS size,
       collect(gds.util.asNode(nodeId).name) AS members
ORDER BY size DESC
```

### Community Detection Comparison

```
┌──────────────────────────────────────────────────────────────┐
│  Algorithm              │  Speed  │  Quality  │  Best For    │
├─────────────────────────┼─────────┼───────────┼──────────────┤
│  Louvain                │  🟡     │  ⭐⭐⭐  │  Best general│
│  Label Propagation      │  🟢     │  ⭐⭐    │  Large graphs│
│  WCC                    │  🟢     │  N/A     │  Islands     │
│  SCC (Strongly Connected)│ 🟡     │  N/A     │  Directed    │
│  K-1 Coloring           │  🟢     │  ⭐      │  Scheduling  │
└──────────────────────────┴─────────┴───────────┴──────────────┘
```

---

## 🛤️ 9. Path Finding Algorithms — Optimal Routes

### 9.1 Dijkstra's Algorithm — Weighted Shortest Path

```
Find the cheapest route from A to E:

       10        5
  A ─────── B ─────── E
  │         │         ▲
  │ 3       │ 2       │ 1
  │         │         │
  C ─────── D ────────┘
       4         1

  Dijkstra finds: A → C (3) → D (4+3=7) → E (1+7=8) = cost 8
  NOT A → B (10) → E (5+10=15) = cost 15
```

```cypher
// Project with relationship weights
CALL gds.graph.project('routes', 'City', {
  ROAD: {properties: 'distance'}
})

// Run Dijkstra
CALL gds.shortestPath.dijkstra.stream('routes', {
  sourceNode: id(startNode),
  targetNode: id(endNode),
  relationshipWeightProperty: 'distance'
})
YIELD index, sourceNode, targetNode, totalCost, nodeIds, costs, path
RETURN
  [nodeId IN nodeIds | gds.util.asNode(nodeId).name] AS route,
  totalCost AS totalDistance

// Single-source to ALL targets
CALL gds.allShortestPaths.dijkstra.stream('routes', {
  sourceNode: id(startNode),
  relationshipWeightProperty: 'distance'
})
YIELD targetNode, totalCost
RETURN gds.util.asNode(targetNode).name AS destination,
       totalCost
ORDER BY totalCost

CALL gds.graph.drop('routes')
```

### 9.2 A* Algorithm — Heuristic-Based (Faster for Geo)

Uses a **heuristic** (estimated distance to target) to search smarter — doesn't explore dead ends.

```cypher
// A* with latitude/longitude heuristic
CALL gds.shortestPath.astar.stream('routes', {
  sourceNode: id(startNode),
  targetNode: id(endNode),
  relationshipWeightProperty: 'distance',
  latitudeProperty: 'latitude',
  longitudeProperty: 'longitude'
})
YIELD totalCost, nodeIds
RETURN totalCost, 
       [n IN nodeIds | gds.util.asNode(n).name] AS route
```

### 9.3 Yen's K-Shortest Paths

Find not just THE shortest, but the **top K** shortest paths (alternatives).

```cypher
CALL gds.shortestPath.yens.stream('routes', {
  sourceNode: id(startNode),
  targetNode: id(endNode),
  relationshipWeightProperty: 'distance',
  k: 3  // Find top 3 shortest paths
})
YIELD index, totalCost, nodeIds
RETURN index + 1 AS rank,
       totalCost,
       [n IN nodeIds | gds.util.asNode(n).name] AS route
```

### Path Finding Comparison

```
┌─────────────────────────────────────────────────────────────┐
│  Algorithm       │  Weighted? │  Heuristic? │  Use Case     │
├──────────────────┼────────────┼─────────────┼───────────────┤
│  BFS             │  ❌ (hops) │  ❌         │  Min hops     │
│  Dijkstra        │  ✅        │  ❌         │  General       │
│  A*              │  ✅        │  ✅         │  Geo/maps      │
│  Yen's K-Paths   │  ✅        │  ❌         │  Alternatives  │
│  Random Walk     │  Optional  │  ❌         │  Sampling/ML   │
│  Min Span Tree   │  ✅        │  ❌         │  Network design│
└──────────────────┴────────────┴─────────────┴───────────────┘
```

---

## 🤝 10. Similarity Algorithms — Finding Lookalikes

### Node Similarity (Jaccard)

**Concept**: Two nodes are similar if they share many of the same neighbors.

```
  Alice likes: [Neo4j, Java, Python, Cypher]
  Bob likes:   [Neo4j, Java, Scala, Spark]

  Jaccard Similarity = |Intersection| / |Union|
                     = |{Neo4j, Java}| / |{Neo4j, Java, Python, Cypher, Scala, Spark}|
                     = 2 / 6
                     = 0.333
```

```cypher
// Project a bipartite graph (persons → skills)
CALL gds.graph.project('skillGraph', 
  ['Person', 'Skill'], 
  {KNOWS_SKILL: {orientation: 'NATURAL'}}
)

// Run Node Similarity
CALL gds.nodeSimilarity.stream('skillGraph')
YIELD node1, node2, similarity
RETURN gds.util.asNode(node1).name AS person1,
       gds.util.asNode(node2).name AS person2,
       round(similarity, 3) AS jaccardScore
ORDER BY jaccardScore DESC
LIMIT 10

CALL gds.graph.drop('skillGraph')
```

---

## 🔮 11. Link Prediction — Predicting Future Connections

**Concept**: Based on the current graph structure, predict which connections are LIKELY to form in the future.

```
  Alice ── Bob ── Carol ── Dave

  Question: Will Alice connect to Carol?
  → They share a neighbor (Bob)
  → Common Neighbors score = 1
  → Prediction: LIKELY! 👍
```

```cypher
// Common Neighbors score
MATCH (a:Person {name: "Alice"})
MATCH (b:Person {name: "Carol"})
RETURN gds.alpha.linkprediction.commonNeighbors(a, b) AS cnScore

// Adamic Adar (weights common neighbors by their connectivity)
RETURN gds.alpha.linkprediction.adamicAdar(a, b) AS aaScore

// Preferential Attachment (popular nodes attract more connections)
RETURN gds.alpha.linkprediction.preferentialAttachment(a, b) AS paScore
```

**Real-World Uses:**

| Domain | Prediction |
|--------|-----------|
| Social Media | "People you may know" (LinkedIn, Facebook) |
| E-commerce | "You might also like" |
| Research | Predicting future collaborations |
| Finance | Predicting business relationships |

---

## 🧬 12. Graph Embeddings — Graphs → Vectors → ML

Convert graph structure into **vector representations** that can be fed into machine learning models.

### FastRP (Fast Random Projection) — Fastest

```cypher
CALL gds.graph.project('embedding', 'Person', 'KNOWS')

// Generate 128-dimensional embeddings
CALL gds.fastRP.stream('embedding', {
  embeddingDimension: 128,
  iterationWeights: [0.0, 1.0, 1.0]
})
YIELD nodeId, embedding
RETURN gds.util.asNode(nodeId).name AS person,
       embedding
LIMIT 5

CALL gds.graph.drop('embedding')
```

### Node2Vec — Most Popular

```cypher
CALL gds.node2vec.stream('embedding', {
  embeddingDimension: 64,
  walkLength: 80,
  walksPerNode: 10,
  returnFactor: 1.0,    // Controls exploration vs exploitation
  inOutFactor: 1.0
})
YIELD nodeId, embedding
RETURN gds.util.asNode(nodeId).name, embedding
```

### Embedding Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│  Graph Embeddings → ML Pipeline                              │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────┐    ┌──────────────┐    ┌──────────────────┐    │
│  │ Graph   │    │ Embeddings   │    │ ML Model         │    │
│  │ Data    │───→│ (Vectors)    │───→│                  │    │
│  │ in Neo4j│    │ [0.2, -0.5,  │    │ Classification   │    │
│  │         │    │  0.8, ...]   │    │ Clustering       │    │
│  │         │    │              │    │ Recommendations  │    │
│  └─────────┘    └──────────────┘    └──────────────────┘    │
│                                                               │
│  Use Cases:                                                  │
│  • Node classification (predict missing labels)             │
│  • Link prediction (predict future connections)             │
│  • Anomaly detection (unusual graph patterns)               │
│  • Recommendation systems (similar users/items)             │
│  • Knowledge graph completion                               │
│  • Drug discovery (molecular graph analysis)                │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🏆 13. Real-World Case Studies

### Case Study 1: Fraud Detection at a Bank

```cypher
// Model: Accounts, Transactions, Devices, IP Addresses

// Find fraud ring: accounts that share devices AND IPs
// AND transfer money in circles
MATCH (a1:Account)-[:USED_DEVICE]->(d:Device)<-[:USED_DEVICE]-(a2:Account),
      (a1)-[:USED_IP]->(ip:IP)<-[:USED_IP]-(a2)
WHERE a1 <> a2
WITH a1, a2, d, ip

// Check for circular money flow
MATCH path = (a1)-[:TRANSFERRED*2..5]->(a1)
WHERE ALL(r IN relationships(path) WHERE r.amount > 5000)
RETURN DISTINCT [n IN nodes(path) | n.accountId] AS fraudRing,
       length(path) AS ringSize,
       reduce(s = 0, r IN relationships(path) | s + r.amount) AS totalFlow
```

### Case Study 2: Recommendation Engine (Netflix-style)

```cypher
// "Users similar to you watched these — you might like them too"
MATCH (me:User {id: $userId})-[:WATCHED]->(m:Movie)
      <-[:WATCHED]-(similar:User)-[:WATCHED]->(rec:Movie)
WHERE NOT (me)-[:WATCHED]->(rec)
WITH rec, count(DISTINCT similar) AS score,
     collect(DISTINCT similar.name) AS recommendedBy
RETURN rec.title, score, recommendedBy
ORDER BY score DESC
LIMIT 10
```

### Case Study 3: Knowledge Graph for GenAI (RAG)

```cypher
// Build a knowledge graph from documents
// Then use it for Retrieval-Augmented Generation

// Store document chunks with entities
CREATE (chunk:Chunk {text: "Neo4j is a graph database...", embedding: $vector})
CREATE (entity:Entity {name: "Neo4j", type: "Technology"})
CREATE (chunk)-[:MENTIONS]->(entity)

// RAG query: Find relevant context using graph + vectors
MATCH (e:Entity {name: "Neo4j"})<-[:MENTIONS]-(chunk:Chunk)
OPTIONAL MATCH (e)-[:RELATED_TO*1..2]-(related:Entity)<-[:MENTIONS]-(relChunk:Chunk)
RETURN chunk.text AS directContext,
       collect(DISTINCT relChunk.text) AS relatedContext
// Feed both into LLM for grounded, accurate answers
```

---

## 🧠 Quick Recall — Chapter Summary

### Data Modeling

| Concept | One-Line Summary |
|---------|-----------------|
| Whiteboard Rule | If you can draw it on a whiteboard, it's your graph model |
| Entity → Node | Independent things become nodes with labels |
| FK → Relationship | Foreign keys become named relationships |
| JOIN Table → Relationship | Many-to-many tables become relationships with properties |
| Intermediate Node | Promote complex relationships to nodes (reification) |
| Super Node | Avoid nodes with millions of relationships — add hierarchy |
| Specific Rel Types | Use `:FRIEND_OF` not `:RELATES_TO {type: "friend"}` |

### Graph Algorithms

| Algorithm | Question It Answers | Time Complexity |
|-----------|-------------------|-----------------|
| PageRank | Who is most influential? | O(V + E) per iteration |
| Betweenness | Who bridges groups? | O(V × E) |
| Degree | Who has the most connections? | O(V + E) |
| Louvain | What groups/communities exist? | O(V × log V) |
| Label Propagation | Fast clustering? | O(V + E) |
| WCC | Are there disconnected islands? | O(V + E) |
| Dijkstra | Cheapest/shortest weighted path? | O(E + V log V) |
| A* | Shortest geo-path with heuristic? | O(E) best case |
| Node Similarity | Who are the lookalikes? | O(V²) worst |
| Node2Vec/FastRP | Graph → vector for ML? | O(V × walks) |

---

## ❓ Self-Check Questions

1. How would you convert a relational many-to-many JOIN table into a graph model?
2. What is the "Intermediate Node" pattern? When should you use it?
3. Name 3 graph modeling anti-patterns and how to fix them.
4. Explain the 3-step GDS workflow (Project → Run → Clean up).
5. What's the difference between PageRank and Betweenness Centrality?
6. When would you use Louvain vs Label Propagation for community detection?
7. Compare Dijkstra vs A* — when is A* faster?
8. What are graph embeddings and why are they valuable for ML?
9. Design a graph model for a **hospital** system (patients, doctors, diagnoses, treatments, departments).
10. Write a Cypher + GDS query to find the top 5 most influential users in a social network.

---

## 🎓 Practice Project — Build This!

```
Build a MOVIE RECOMMENDATION GRAPH:

1. Load the Neo4j Movies dataset   → :play movies (in Neo4j Browser)
2. Run PageRank                     → Find most influential actors
3. Run Louvain                      → Find actor communities/clusters
4. Run Node Similarity              → Find similar actors
5. Build a recommender              → "If you liked X, try Y"
6. Run shortest path               → "Six Degrees of Kevin Bacon"

This single project touches EVERY major concept in this chapter!
```

---

> **Previous Chapter** → [3E.2 — Cypher Query Language — Complete Guide](./02-Cypher.md)
>
> **Next Section** → [3F.1 — Elasticsearch Architecture & Concepts](../13-Elasticsearch/01-ES-Architecture.md)
