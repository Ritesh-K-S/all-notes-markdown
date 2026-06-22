# 3E.2 — Cypher Query Language — Complete Guide 🟡⭐🔥

> **"SQL talks to tables. Cypher draws pictures. And those pictures ARE the query."**

---

## 📌 What You'll Master in This Chapter

- **Cypher syntax** from zero to advanced — the entire language
- **MATCH** — reading/traversing graph data with pattern matching
- **CREATE, MERGE** — writing and upserting nodes & relationships
- **SET, REMOVE, DELETE, DETACH DELETE** — updating and removing data
- **WHERE** — filtering with powerful predicate expressions
- **Aggregations** — COUNT, SUM, AVG, COLLECT and more
- **Path finding** — variable-length patterns, shortest path
- **WITH** — query chaining (Cypher's pipeline operator)
- **UNWIND, FOREACH** — list processing
- **Indexes & Constraints** — schema operations in Cypher
- **APOC** — the essential procedures library
- Real-world query patterns with full explanations

---

## 🌍 Cypher — The Backstory (30 Seconds)

```
Created:    2011 by Andrés Taylor at Neo4j
Inspired By: SQL (declarative), SPARQL (pattern matching), Haskell (expression-based)
Standardized: openCypher project (open standard, adopted by others)
Also Used In: Amazon Neptune (partial), SAP HANA Graph, Memgraph, Redis Graph
Key Idea:    ASCII art patterns that LOOK like the graph they describe
```

---

## 🧠 1. The Core Idea — ASCII Art Pattern Matching

Cypher's genius is that your query **looks like the graph** you're searching for:

```
THE PATTERN IN YOUR QUERY          THE GRAPH ON DISK

    (a)-[:KNOWS]->(b)        →       (Alice)───KNOWS──→(Bob)
         │                                 │
    Round brackets = Nodes                 Node
    Square brackets = Relationships        Relationship
    Arrow = Direction                      Direction

More patterns:

  (a)                          A single node
  (a:Person)                   A node with label Person
  (a)-[r]->(b)                 a connected to b (any relationship)
  (a)-[:KNOWS]->(b)            a KNOWS b
  (a)-[:KNOWS {since: 2020}]->(b)   a KNOWS b since 2020
  (a)-[:KNOWS]->(b)-[:LIVES_IN]->(c)   Chain: a knows b who lives in c
  (a)-[:KNOWS*2]->(d)          a knows someone who knows d (2 hops)
  (a)-[:KNOWS*1..5]->(e)       1 to 5 hops away
  p = (a)-[:KNOWS*]->(b)       Capture the entire path as variable p
```

> 💡 **Mental Model**: When writing Cypher, **draw the graph pattern you want**, then translate it character by character into syntax. The query IS the drawing.

---

## 🔥 2. MATCH — Reading Data (The Heart of Cypher)

`MATCH` finds patterns in the graph. It's like `SELECT` in SQL — but for graph patterns.

### 2.1 Finding Nodes

```cypher
-- Find ALL nodes (careful — could be millions!)
MATCH (n)
RETURN n
LIMIT 25

-- Find all Person nodes
MATCH (p:Person)
RETURN p

-- Find a specific person
MATCH (p:Person {name: "Tom Hanks"})
RETURN p

-- Return specific properties (like SELECT columns)
MATCH (p:Person {name: "Tom Hanks"})
RETURN p.name, p.born, p.city

-- Multiple labels
MATCH (p:Person:Actor)
RETURN p.name
```

### 2.2 Finding Relationships

```cypher
-- Who did Tom Hanks act with? (1 hop)
MATCH (tom:Person {name: "Tom Hanks"})-[:ACTED_IN]->(movie:Movie)
RETURN movie.title, movie.year

-- Find Tom's co-actors (2 hops: Tom → Movie ← Other Actor)
MATCH (tom:Person {name: "Tom Hanks"})-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(coActor:Person)
RETURN DISTINCT coActor.name, m.title

-- Any relationship (ignore type)
MATCH (tom:Person {name: "Tom Hanks"})-[r]->(x)
RETURN type(r), x

-- Incoming relationships
MATCH (m:Movie {title: "Forrest Gump"})<-[r]-(p:Person)
RETURN p.name, type(r)

-- Undirected (both ways)
MATCH (tom:Person {name: "Tom Hanks"})-[:KNOWS]-(friend)
RETURN friend.name
```

### 2.3 Pattern Matching — The Power Move

```cypher
-- The Triangle: Find people who know each other AND worked together
MATCH (a:Person)-[:KNOWS]->(b:Person),
      (a)-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(b)
RETURN a.name, b.name, m.title

-- Find someone who directed AND acted in the same movie
MATCH (p:Person)-[:DIRECTED]->(m:Movie)<-[:ACTED_IN]-(p)
RETURN p.name AS directorActor, m.title
-- Notice: SAME variable 'p' on both sides = same person!

-- Negative pattern: People who DON'T know each other
MATCH (a:Person), (b:Person)
WHERE a <> b
AND NOT (a)-[:KNOWS]-(b)
RETURN a.name, b.name
LIMIT 10
```

---

## ✍️ 3. CREATE — Writing Data

### 3.1 Creating Nodes

```cypher
-- Create a single node
CREATE (p:Person {name: "Ritesh", age: 28, city: "Mumbai"})
RETURN p

-- Create a node with multiple labels
CREATE (p:Person:Developer:DBA {name: "Ritesh", skills: ["Neo4j", "Oracle"]})
RETURN p

-- Create multiple nodes at once
CREATE (a:Person {name: "Alice"}),
       (b:Person {name: "Bob"}),
       (c:Person {name: "Carol"})
```

### 3.2 Creating Relationships

```cypher
-- Create a relationship between existing nodes
MATCH (a:Person {name: "Alice"}), (b:Person {name: "Bob"})
CREATE (a)-[:KNOWS {since: 2020, strength: "close"}]->(b)

-- Create nodes AND relationships in one go
CREATE (a:Person {name: "Dave"})-[:WORKS_AT {role: "Engineer", since: 2022}]->(c:Company {name: "Neo4j Inc."})

-- Create multiple relationships
MATCH (alice:Person {name: "Alice"}),
      (bob:Person {name: "Bob"}),
      (carol:Person {name: "Carol"})
CREATE (alice)-[:KNOWS]->(bob),
       (alice)-[:KNOWS]->(carol),
       (bob)-[:KNOWS]->(carol)
```

> ⚠️ **Warning**: `CREATE` always creates new data — even if identical data already exists! This can cause **duplicates**. Use `MERGE` to avoid this.

---

## 🔄 4. MERGE — Create If Not Exists (Upsert)

`MERGE` is your **idempotent** operation — it either matches existing data or creates it.

```cypher
-- MERGE a node: creates only if no match
MERGE (p:Person {name: "Ritesh"})
ON CREATE SET p.createdAt = datetime()
ON MATCH SET p.lastSeen = datetime()
RETURN p

-- MERGE a relationship: creates only if pattern doesn't exist
MATCH (a:Person {name: "Alice"}), (b:Person {name: "Bob"})
MERGE (a)-[:KNOWS]->(b)

-- Full MERGE with ON CREATE and ON MATCH
MERGE (p:Person {name: "Alice"})
ON CREATE SET p.created = datetime(), p.age = 30
ON MATCH SET p.lastLogin = datetime(), p.loginCount = coalesce(p.loginCount, 0) + 1
RETURN p
```

### CREATE vs MERGE — The Decision

```
┌─────────────────────────────────────────────────────────┐
│                  CREATE vs MERGE                         │
├──────────────┬──────────────────────────────────────────┤
│  CREATE      │  MERGE                                   │
├──────────────┼──────────────────────────────────────────┤
│ Always       │ Only creates if pattern                  │
│ creates new  │ doesn't already exist                    │
│              │                                          │
│ Can cause    │ Prevents duplicates                      │
│ duplicates   │ (idempotent)                             │
│              │                                          │
│ Faster       │ Slightly slower (must check first)       │
│              │                                          │
│ Use for:     │ Use for:                                 │
│ Bulk imports │ Ongoing data ingestion                   │
│ (trusted     │ Upsert patterns                          │
│  source)     │ Constraint-safe operations               │
└──────────────┴──────────────────────────────────────────┘
```

---

## 🔧 5. SET, REMOVE — Updating Properties & Labels

### SET — Add or Update Properties

```cypher
-- Set a single property
MATCH (p:Person {name: "Ritesh"})
SET p.age = 29

-- Set multiple properties
MATCH (p:Person {name: "Ritesh"})
SET p.age = 29, p.city = "Bangalore", p.role = "Architect"

-- Copy all properties from a map
MATCH (p:Person {name: "Ritesh"})
SET p += {age: 29, city: "Bangalore"}
-- += merges (keeps existing props), = replaces ALL props

-- Add a label
MATCH (p:Person {name: "Ritesh"})
SET p:Developer:GraphExpert
```

### REMOVE — Remove Properties & Labels

```cypher
-- Remove a property
MATCH (p:Person {name: "Ritesh"})
REMOVE p.age

-- Remove a label
MATCH (p:Person {name: "Ritesh"})
REMOVE p:GraphExpert

-- Set property to null (same as REMOVE)
MATCH (p:Person {name: "Ritesh"})
SET p.age = null
```

---

## 🗑️ 6. DELETE — Removing Nodes & Relationships

```cypher
-- Delete a relationship
MATCH (a:Person {name: "Alice"})-[r:KNOWS]->(b:Person {name: "Bob"})
DELETE r

-- Delete a node (ONLY if it has no relationships)
MATCH (p:Person {name: "TestUser"})
DELETE p
-- ❌ ERROR if the node has any relationships!

-- DETACH DELETE: Delete a node AND all its relationships
MATCH (p:Person {name: "TestUser"})
DETACH DELETE p
-- ✅ Removes the node + all connected relationships

-- Delete ALL data (nuclear option — be careful!)
MATCH (n)
DETACH DELETE n
```

> ⚠️ **Safety Rule**: Neo4j prevents deleting nodes that have relationships (referential integrity). Use `DETACH DELETE` when you intentionally want to remove everything connected.

---

## 🔍 7. WHERE — Filtering Results

### Basic Comparisons

```cypher
MATCH (p:Person)
WHERE p.age > 25
AND p.city = "Mumbai"
RETURN p.name, p.age

-- OR conditions
MATCH (p:Person)
WHERE p.city = "Mumbai" OR p.city = "Delhi"
RETURN p.name

-- IN list
MATCH (p:Person)
WHERE p.city IN ["Mumbai", "Delhi", "Bangalore"]
RETURN p.name

-- NOT
MATCH (p:Person)
WHERE NOT p.city = "Mumbai"
RETURN p.name
```

### String Operations

```cypher
-- STARTS WITH, ENDS WITH, CONTAINS
MATCH (p:Person)
WHERE p.name STARTS WITH "Tom"
RETURN p.name

MATCH (p:Person)
WHERE p.name CONTAINS "ank"
RETURN p.name

-- Regular expressions
MATCH (p:Person)
WHERE p.name =~ "Tom.*"
RETURN p.name

-- Case-insensitive regex
MATCH (p:Person)
WHERE p.name =~ "(?i)tom.*"
RETURN p.name
```

### Existence & Null Checks

```cypher
-- Property exists
MATCH (p:Person)
WHERE p.email IS NOT NULL
RETURN p.name

-- Property doesn't exist
MATCH (p:Person)
WHERE p.email IS NULL
RETURN p.name

-- Pattern existence: does this person KNOW anyone?
MATCH (p:Person)
WHERE (p)-[:KNOWS]->()
RETURN p.name

-- Pattern non-existence: loners with no friends
MATCH (p:Person)
WHERE NOT (p)-[:KNOWS]->()
RETURN p.name AS loner
```

### List Predicates

```cypher
-- ANY: at least one element matches
MATCH (p:Person)
WHERE ANY(skill IN p.skills WHERE skill = "Neo4j")
RETURN p.name

-- ALL: every element matches
MATCH (p:Person)
WHERE ALL(skill IN p.skills WHERE skill STARTS WITH "J")
RETURN p.name

-- NONE: no element matches
MATCH (p:Person)
WHERE NONE(skill IN p.skills WHERE skill = "COBOL")
RETURN p.name

-- SINGLE: exactly one element matches
MATCH (p:Person)
WHERE SINGLE(skill IN p.skills WHERE skill = "Neo4j")
RETURN p.name
```

---

## 📊 8. Aggregations — Crunching Numbers

```cypher
-- COUNT
MATCH (p:Person)
RETURN count(p) AS totalPeople

-- COUNT per group (like GROUP BY in SQL)
MATCH (p:Person)
RETURN p.city, count(p) AS peoplePerCity
ORDER BY peoplePerCity DESC

-- SUM, AVG, MIN, MAX
MATCH (p:Person)
RETURN avg(p.age) AS avgAge,
       min(p.age) AS youngest,
       max(p.age) AS oldest,
       sum(p.age) AS totalAge

-- COLLECT: Gather values into a list (extremely powerful!)
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN p.name, collect(m.title) AS movies
-- Returns: "Tom Hanks" | ["Forrest Gump", "Cast Away", "Toy Story"]

-- COUNT DISTINCT
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN count(DISTINCT m) AS uniqueMovies

-- percentileDisc, percentileCont, stDev
MATCH (p:Person)
WHERE p.age IS NOT NULL
RETURN percentileDisc(p.age, 0.5) AS medianAge,
       stDev(p.age) AS standardDeviation
```

> 💡 **Pro Tip**: `COLLECT()` is one of Cypher's superpowers. It turns multiple rows into a single list. Use it to avoid the "Cartesian explosion" problem when combining results.

---

## 📐 9. ORDER BY, LIMIT, SKIP — Result Control

```cypher
-- Sort results
MATCH (p:Person)
RETURN p.name, p.age
ORDER BY p.age DESC

-- Pagination
MATCH (p:Person)
RETURN p.name
ORDER BY p.name
SKIP 20         -- Skip first 20 results
LIMIT 10        -- Return next 10

-- Top N pattern
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN p.name, count(m) AS movieCount
ORDER BY movieCount DESC
LIMIT 5
```

---

## 🔗 10. WITH — Query Chaining (Piping)

`WITH` is Cypher's **pipeline operator**. It passes results from one part of the query to the next — like Unix pipes.

```cypher
-- Find actors with more than 5 movies, then find their co-actors
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
WITH p, count(m) AS movieCount        -- ← Pipeline: filter first
WHERE movieCount > 5                   -- ← Filter on aggregation
MATCH (p)-[:ACTED_IN]->(m2:Movie)<-[:ACTED_IN]-(coActor:Person)
RETURN p.name, movieCount, collect(DISTINCT coActor.name) AS coActors
ORDER BY movieCount DESC

-- Use WITH to control scope and eliminate duplicates
MATCH (p:Person)-[:KNOWS]->(f:Person)
WITH p, collect(f.name) AS friends, count(f) AS friendCount
WHERE friendCount >= 3
RETURN p.name, friends, friendCount

-- WITH + LIMIT for "top N then expand" patterns
MATCH (m:Movie)
WITH m
ORDER BY m.year DESC
LIMIT 5
MATCH (m)<-[:ACTED_IN]-(actor:Person)
RETURN m.title, m.year, collect(actor.name) AS cast
```

> ⭐ **Key Insight**: Without `WITH`, you can't filter on aggregated values. `WITH` acts like a "checkpoint" — everything before it is computed, then passed forward.

---

## 🔄 11. Variable-Length Paths & Shortest Path

### Variable-Length Pattern Matching

```cypher
-- Exactly 2 hops
MATCH (a:Person {name: "Alice"})-[:KNOWS*2]->(c:Person)
RETURN c.name AS twoHopsAway

-- 1 to 3 hops (friends, friends-of-friends, f-o-f-o-f)
MATCH (a:Person {name: "Alice"})-[:KNOWS*1..3]->(c:Person)
RETURN DISTINCT c.name, length(shortestPath((a)-[:KNOWS*]-(c))) AS distance

-- Any depth (careful — can be expensive!)
MATCH (a:Person {name: "Alice"})-[:KNOWS*]->(c:Person)
RETURN DISTINCT c.name
-- ⚠️ Without a depth limit, this could traverse the ENTIRE graph!

-- Capture the path
MATCH p = (a:Person {name: "Alice"})-[:KNOWS*1..4]->(b:Person {name: "Dave"})
RETURN p, length(p) AS pathLength,
       [n IN nodes(p) | n.name] AS nodeNames
```

### Shortest Path

```cypher
-- Find THE shortest path between two people
MATCH p = shortestPath(
  (a:Person {name: "Alice"})-[:KNOWS*]-(b:Person {name: "Dave"})
)
RETURN p, length(p) AS hops,
       [n IN nodes(p) | n.name] AS route

-- ALL shortest paths (there might be multiple with same length)
MATCH p = allShortestPaths(
  (a:Person {name: "Alice"})-[:KNOWS*]-(b:Person {name: "Dave"})
)
RETURN p, length(p) AS hops

-- Shortest path with conditions
MATCH p = shortestPath(
  (a:Person {name: "Alice"})-[:KNOWS*]-(b:Person {name: "Dave"})
)
WHERE ALL(n IN nodes(p) WHERE n.age > 21)  -- All nodes must be 21+
RETURN p
```

> 💡 **Pro Tip**: `shortestPath` uses breadth-first search internally. For weighted shortest paths (like Dijkstra), use the **GDS library** (covered in Chapter 3E.3).

---

## 📋 12. UNWIND — Turning Lists Into Rows

`UNWIND` is the inverse of `COLLECT`. It takes a list and produces one row per element.

```cypher
-- Basic UNWIND
UNWIND [1, 2, 3, 4, 5] AS num
RETURN num

-- Create multiple nodes from a list
UNWIND ["Alice", "Bob", "Carol", "Dave"] AS name
CREATE (p:Person {name: name})

-- Bulk relationship creation
WITH [
  {from: "Alice", to: "Bob"},
  {from: "Alice", to: "Carol"},
  {from: "Bob", to: "Dave"}
] AS connections
UNWIND connections AS conn
MATCH (a:Person {name: conn.from}), (b:Person {name: conn.to})
MERGE (a)-[:KNOWS]->(b)

-- Explode a property list
MATCH (p:Person)
WHERE p.skills IS NOT NULL
UNWIND p.skills AS skill
RETURN skill, collect(p.name) AS peopleWithSkill
```

---

## 🛡️ 13. Indexes & Constraints — Schema in Neo4j

Neo4j is **schema-optional**: you don't need to define a schema, but you SHOULD for performance and data integrity.

### 13.1 Indexes

```cypher
-- Create a B-Tree index on a property (most common)
CREATE INDEX person_name_idx FOR (p:Person) ON (p.name)

-- Composite index (multiple properties)
CREATE INDEX person_name_city_idx FOR (p:Person) ON (p.name, p.city)

-- Full-text index (for text search)
CREATE FULLTEXT INDEX person_search FOR (p:Person) ON EACH [p.name, p.bio]

-- Relationship index
CREATE INDEX knows_since_idx FOR ()-[r:KNOWS]-() ON (r.since)

-- Show all indexes
SHOW INDEXES

-- Drop an index
DROP INDEX person_name_idx
```

### Index Type Reference

```
┌──────────────────┬───────────────────────────────────────────┐
│  Index Type      │  Best For                                │
├──────────────────┼───────────────────────────────────────────┤
│  Range (B-Tree)  │  Equality, range queries, ORDER BY       │
│  Text            │  String prefix, CONTAINS, regex          │
│  Full-Text       │  Natural language search (Lucene-based)  │
│  Point           │  Geospatial distance queries             │
│  Token Lookup    │  Fast label/type lookups (auto-created)  │
│  Composite       │  Multi-property queries                  │
└──────────────────┴───────────────────────────────────────────┘
```

### 13.2 Constraints

```cypher
-- Unique constraint (also auto-creates an index!)
CREATE CONSTRAINT person_name_unique FOR (p:Person) REQUIRE p.name IS UNIQUE

-- Node key constraint (composite uniqueness) — Enterprise only
CREATE CONSTRAINT person_key FOR (p:Person) REQUIRE (p.firstName, p.lastName) IS NODE KEY

-- Existence constraint — Enterprise only
CREATE CONSTRAINT person_email_exists FOR (p:Person) REQUIRE p.email IS NOT NULL

-- Property type constraint (Neo4j 5.9+)
CREATE CONSTRAINT person_age_type FOR (p:Person) REQUIRE p.age IS :: INTEGER

-- Show all constraints
SHOW CONSTRAINTS

-- Drop a constraint
DROP CONSTRAINT person_name_unique
```

### Constraints Quick Reference

```
┌────────────────────┬────────────────────┬────────────────────────┐
│  Constraint        │  Community Edition │  Enterprise Edition    │
├────────────────────┼────────────────────┼────────────────────────┤
│  UNIQUE            │  ✅               │  ✅                    │
│  NODE KEY          │  ❌               │  ✅                    │
│  EXISTENCE         │  ❌               │  ✅                    │
│  PROPERTY TYPE     │  ✅ (5.9+)        │  ✅ (5.9+)            │
└────────────────────┴────────────────────┴────────────────────────┘
```

---

## 🧰 14. Essential Functions Reference

### String Functions

```cypher
RETURN
  toUpper("hello")           AS upper,          -- "HELLO"
  toLower("HELLO")           AS lower,          -- "hello"
  trim("  hello  ")          AS trimmed,        -- "hello"
  replace("hello", "l", "r") AS replaced,       -- "herro"
  substring("hello", 1, 3)   AS sub,            -- "ell"
  size("hello")              AS length,          -- 5
  left("hello", 3)           AS leftPart,        -- "hel"
  right("hello", 3)          AS rightPart,       -- "llo"
  split("a,b,c", ",")        AS parts,           -- ["a","b","c"]
  reverse("hello")           AS reversed         -- "olleh"
```

### Numeric Functions

```cypher
RETURN
  abs(-42)                   AS absolute,       -- 42
  ceil(3.2)                  AS ceiling,         -- 4.0
  floor(3.8)                 AS floored,         -- 3.0
  round(3.567, 2)            AS rounded,         -- 3.57
  rand()                     AS random,          -- 0.0 to 1.0
  sign(-5)                   AS signOf,          -- -1
  toInteger("42")            AS toInt,           -- 42
  toFloat("3.14")            AS toFl             -- 3.14
```

### List Functions

```cypher
RETURN
  size([1,2,3])              AS len,             -- 3
  head([1,2,3])              AS first,           -- 1
  last([1,2,3])              AS lastEl,          -- 3
  tail([1,2,3])              AS rest,            -- [2,3]
  range(1, 5)                AS nums,            -- [1,2,3,4,5]
  reverse([1,2,3])           AS rev,             -- [3,2,1]
  [x IN [1,2,3,4,5] WHERE x > 3]  AS filtered,  -- [4,5]
  [x IN [1,2,3] | x * 2]    AS doubled          -- [2,4,6]
```

### Date & Time Functions

```cypher
RETURN
  date()                     AS today,           -- 2026-06-02
  datetime()                 AS now,             -- full timestamp
  time()                     AS currentTime,
  date("2026-06-02")         AS specific,
  date().year                AS year,            -- 2026
  date().month               AS month,           -- 6
  duration.between(date("2025-01-01"), date()) AS diff,
  date() + duration("P30D")  AS thirtyDaysLater
```

### Graph-Specific Functions

```cypher
-- Node/Relationship info
MATCH (p:Person)-[r:KNOWS]->(f:Person)
RETURN
  id(p)                      AS internalId,      -- Neo4j's internal ID
  elementId(p)               AS elementId,        -- Recommended over id()
  labels(p)                  AS nodeLabels,       -- ["Person"]
  type(r)                    AS relType,          -- "KNOWS"
  properties(p)              AS allProps,          -- {name:"...", age:...}
  keys(p)                    AS propKeys           -- ["name", "age"]

-- Path functions
MATCH p = shortestPath((a:Person)-[:KNOWS*]-(b:Person))
WHERE a.name = "Alice" AND b.name = "Dave"
RETURN
  nodes(p)                   AS allNodes,
  relationships(p)           AS allRels,
  length(p)                  AS pathLength
```

---

## 🏗️ 15. CASE Expressions (Conditional Logic)

```cypher
-- Simple CASE
MATCH (p:Person)
RETURN p.name,
  CASE p.age
    WHEN null THEN "Unknown"
    WHEN 0    THEN "Newborn"
    ELSE toString(p.age) + " years old"
  END AS ageDescription

-- Generic CASE (more flexible)
MATCH (p:Person)
RETURN p.name,
  CASE
    WHEN p.age < 18 THEN "Minor"
    WHEN p.age < 30 THEN "Young Adult"
    WHEN p.age < 60 THEN "Adult"
    ELSE "Senior"
  END AS ageGroup

-- CASE in SET (conditional updates)
MATCH (p:Person)
SET p.category = CASE
    WHEN p.age < 18 THEN "Junior"
    WHEN p.age < 60 THEN "Professional"
    ELSE "Retired"
  END
```

---

## 🌟 16. CALL — Subqueries & Procedures

### Subqueries (Neo4j 4.1+)

```cypher
-- CALL {} subquery: run a query within a query
MATCH (p:Person)
CALL {
  WITH p
  MATCH (p)-[:ACTED_IN]->(m:Movie)
  RETURN count(m) AS movieCount
}
RETURN p.name, movieCount
ORDER BY movieCount DESC

-- EXISTS subquery (for filtering)
MATCH (p:Person)
WHERE EXISTS {
  MATCH (p)-[:ACTED_IN]->(m:Movie)
  WHERE m.year > 2020
}
RETURN p.name AS recentActor

-- COUNT subquery
MATCH (p:Person)
WHERE COUNT {
  (p)-[:ACTED_IN]->(:Movie)
} > 5
RETURN p.name AS prolificActor
```

### Built-in Procedures

```cypher
-- Show all databases
SHOW DATABASES

-- Show database schema (labels, relationship types, properties)
CALL db.schema.visualization()

-- List all labels
CALL db.labels()

-- List all relationship types
CALL db.relationshipTypes()

-- List all property keys
CALL db.propertyKeys()
```

---

## 🔌 17. APOC — The Swiss Army Knife of Neo4j

**APOC** (Awesome Procedures On Cypher) is Neo4j's essential utility library with 450+ procedures and functions.

### Installing APOC

```
# Neo4j Desktop: Click "Add Plugin" → APOC → Install
# Docker: Set environment variable NEO4J_PLUGINS=["apoc"]
# Manual: Download .jar → place in /plugins/ → restart
```

### Essential APOC Procedures

```cypher
-- Batch processing (for large datasets)
CALL apoc.periodic.iterate(
  "MATCH (p:Person) WHERE p.age IS NULL RETURN p",
  "SET p.age = 0",
  {batchSize: 1000, parallel: true}
)

-- Load JSON from URL
CALL apoc.load.json("https://api.example.com/data")
YIELD value
CREATE (p:Person {name: value.name, age: value.age})

-- Load CSV with more control than LOAD CSV
CALL apoc.load.csv("file:///data.csv")
YIELD map
CREATE (p:Person {name: map.name})

-- Export graph as JSON
CALL apoc.export.json.all("export.json", {})

-- Create virtual relationships (for projection/visualization)
MATCH (a:Person)-[:KNOWS]->(b:Person)
WITH a, b, count(*) AS strength
CALL apoc.create.vRelationship(a, "KNOWS_VIRTUAL", {strength: strength}, b)
YIELD rel
RETURN a, rel, b

-- Merge nodes (deduplicate)
MATCH (p:Person)
WITH p.email AS email, collect(p) AS nodes
WHERE size(nodes) > 1
CALL apoc.refactor.mergeNodes(nodes, {properties: "combine"})
YIELD node
RETURN node

-- Graph refactoring: rename labels
CALL apoc.refactor.rename.label("OldLabel", "NewLabel")

-- Generate UUID
RETURN apoc.create.uuid() AS uuid

-- Date formatting
RETURN apoc.date.format(timestamp(), 'ms', 'yyyy-MM-dd HH:mm:ss') AS formatted
```

---

## 📦 18. LOAD CSV — Importing Data

```cypher
-- Basic CSV import
LOAD CSV WITH HEADERS FROM "file:///persons.csv" AS row
CREATE (p:Person {
  name: row.name,
  age: toInteger(row.age),
  city: row.city
})

-- Import with MERGE (prevents duplicates)
LOAD CSV WITH HEADERS FROM "file:///persons.csv" AS row
MERGE (p:Person {name: row.name})
SET p.age = toInteger(row.age), p.city = row.city

-- Import relationships from CSV
// relationships.csv: from,to,type,since
LOAD CSV WITH HEADERS FROM "file:///relationships.csv" AS row
MATCH (a:Person {name: row.from})
MATCH (b:Person {name: row.to})
MERGE (a)-[:KNOWS {since: toInteger(row.since)}]->(b)

-- Import with periodic commit (large files)
:auto LOAD CSV WITH HEADERS FROM "file:///big_data.csv" AS row
CALL {
  WITH row
  MERGE (p:Person {id: row.id})
  SET p.name = row.name
} IN TRANSACTIONS OF 1000 ROWS
```

### CSV Import Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│  CSV IMPORT BEST PRACTICES                                   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. CREATE INDEXES FIRST (before importing)                  │
│     → Create indexes on properties you'll MERGE/MATCH on    │
│                                                               │
│  2. IMPORT NODES FIRST, THEN RELATIONSHIPS                   │
│     → Nodes must exist before you can connect them           │
│                                                               │
│  3. USE MERGE (not CREATE) for idempotent imports            │
│     → Re-running won't create duplicates                     │
│                                                               │
│  4. BATCH LARGE FILES with IN TRANSACTIONS OF N ROWS         │
│     → Prevents OutOfMemory on millions of rows              │
│                                                               │
│  5. CONVERT TYPES explicitly                                  │
│     → CSV = all strings! Use toInteger(), toFloat(), etc.    │
│                                                               │
│  6. USE neo4j-admin import FOR HUGE DATASETS                 │
│     → Bypasses transaction overhead (initial load only)      │
│     → Can import billions of nodes in minutes               │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🎯 19. Real-World Query Patterns

### Pattern 1: Recommendation Engine

```cypher
-- "People who bought X also bought..."
MATCH (customer:Person {name: "Alice"})-[:BOUGHT]->(product:Product)
      <-[:BOUGHT]-(other:Person)-[:BOUGHT]->(recommendation:Product)
WHERE NOT (customer)-[:BOUGHT]->(recommendation)
RETURN recommendation.name, count(other) AS score
ORDER BY score DESC
LIMIT 5
```

### Pattern 2: Fraud Detection — Ring Discovery

```cypher
-- Find circular money flows (potential fraud rings)
MATCH path = (a:Account)-[:TRANSFERRED_TO*3..6]->(a)
WHERE ALL(r IN relationships(path) WHERE r.amount > 10000)
RETURN path, [n IN nodes(path) | n.accountId] AS accounts,
       reduce(total = 0, r IN relationships(path) | total + r.amount) AS totalFlow
```

### Pattern 3: Organizational Hierarchy

```cypher
-- Full reporting chain from employee to CEO
MATCH path = (emp:Employee {name: "Ritesh"})-[:REPORTS_TO*]->(boss:Employee)
WHERE NOT (boss)-[:REPORTS_TO]->()  -- boss has no boss = CEO
RETURN [n IN nodes(path) | n.name] AS chain, length(path) AS levels
```

### Pattern 4: Impact Analysis (IT Operations)

```cypher
-- If Server X goes down, what's affected?
MATCH (server:Server {name: "web-prod-01"})<-[:DEPENDS_ON*]-(affected)
RETURN labels(affected)[0] AS type,
       affected.name AS name,
       length(shortestPath((server)<-[:DEPENDS_ON*]-(affected))) AS depth
ORDER BY depth
```

### Pattern 5: Knowledge Graph — Connecting Facts

```cypher
-- Connect related topics across a knowledge base
MATCH (concept:Concept {name: "Graph Database"})
      -[:RELATED_TO*1..2]-(related:Concept)
OPTIONAL MATCH (related)<-[:EXPLAINS]-(article:Article)
RETURN related.name, collect(article.title) AS articles
ORDER BY size(collect(article.title)) DESC
```

---

## ⚡ 20. Performance Tips for Cypher

```
┌─────────────────────────────────────────────────────────────┐
│  CYPHER PERFORMANCE COMMANDMENTS                             │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. USE INDEXES on properties you filter/match on            │
│     → Without an index, Neo4j does a LABEL SCAN (full scan) │
│                                                               │
│  2. ALWAYS USE LABELS in MATCH patterns                      │
│     → MATCH (p:Person) ✅   MATCH (p) ❌                    │
│     → Labels narrow the search space dramatically            │
│                                                               │
│  3. PROFILE YOUR QUERIES                                     │
│     → PROFILE MATCH (p:Person)... shows actual DB hits       │
│     → Look for high "Estimated Rows" vs "Actual Rows"       │
│                                                               │
│  4. LIMIT VARIABLE-LENGTH PATHS                              │
│     → [:KNOWS*] ❌ (unbounded!)                              │
│     → [:KNOWS*1..5] ✅ (bounded)                             │
│                                                               │
│  5. USE WITH TO REDUCE CARDINALITY EARLY                     │
│     → Filter and aggregate early, then expand                │
│                                                               │
│  6. AVOID CARTESIAN PRODUCTS                                 │
│     → Two disconnected MATCH patterns = cross join!          │
│     → Neo4j Browser warns about this                        │
│                                                               │
│  7. USE PARAMETERS (not string concatenation)                │
│     → Enables query plan caching + prevents injection        │
│     → MATCH (p:Person {name: $name}) RETURN p               │
│                                                               │
│  8. BATCH WRITES with IN TRANSACTIONS OF N ROWS              │
│     → Large writes in single tx = memory explosion           │
│                                                               │
│  9. PREFER MERGE OVER CREATE + MATCH-CHECK                   │
│     → One operation vs two = less overhead                   │
│                                                               │
│  10. USE APOC.PERIODIC.ITERATE for bulk operations           │
│      → Parallel batches for millions of updates              │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🧠 Quick Recall — Cypher Cheat Sheet

```
┌──────────────────────────────────────────────────────────────┐
│  CYPHER QUICK REFERENCE                                       │
├────────────────┬─────────────────────────────────────────────┤
│  READ          │                                              │
│  MATCH         │  Find patterns in the graph                 │
│  WHERE         │  Filter results                             │
│  RETURN        │  What to output                             │
│  ORDER BY      │  Sort results                               │
│  LIMIT/SKIP    │  Pagination                                 │
│  WITH          │  Pipeline intermediate results              │
├────────────────┼─────────────────────────────────────────────┤
│  WRITE         │                                              │
│  CREATE        │  Create nodes/relationships (always new)    │
│  MERGE         │  Create if not exists (upsert)              │
│  SET           │  Update properties / add labels             │
│  REMOVE        │  Remove properties / labels                 │
│  DELETE        │  Remove nodes/relationships                 │
│  DETACH DELETE │  Remove node + all its relationships        │
├────────────────┼─────────────────────────────────────────────┤
│  LISTS         │                                              │
│  UNWIND        │  List → rows (explode)                      │
│  COLLECT       │  Rows → list (aggregate)                    │
│  FOREACH       │  Iterate and perform side effects           │
├────────────────┼─────────────────────────────────────────────┤
│  PATHS         │                                              │
│  *N            │  Exactly N hops                             │
│  *1..N         │  1 to N hops                                │
│  shortestPath  │  BFS shortest path                          │
│  allShortestPaths│ All paths of minimum length               │
├────────────────┼─────────────────────────────────────────────┤
│  SCHEMA        │                                              │
│  CREATE INDEX  │  Create property index                      │
│  CREATE CONSTRAINT│ Uniqueness, existence, type constraints  │
│  SHOW INDEXES  │  List all indexes                           │
│  SHOW CONSTRAINTS│ List all constraints                      │
├────────────────┼─────────────────────────────────────────────┤
│  DEBUG         │                                              │
│  EXPLAIN       │  Show execution plan (no execution)         │
│  PROFILE       │  Show plan with actual metrics              │
└────────────────┴─────────────────────────────────────────────┘
```

---

## ❓ Self-Check Questions

1. Write a Cypher query to find all movies released after 2000 where Tom Hanks acted.
2. What's the difference between `CREATE` and `MERGE`? When would you use each?
3. How do you find the shortest path between two nodes in Cypher?
4. Write a query that finds "friends of friends" but excludes direct friends.
5. What happens if you try to `DELETE` a node that has relationships?
6. How does `WITH` differ from `RETURN`? Why is `WITH` essential for complex queries?
7. Write a recommendation query: "People who liked the same movies as me also liked..."
8. What's the purpose of `UNWIND`? Give an example.
9. Why should you always use labels in your `MATCH` patterns?
10. How do you import a CSV with 10 million rows without running out of memory?

---

## 💡 Quick Practice — Try These Now

```cypher
-- If you have the Neo4j Movies dataset loaded (:play movies in Browser):

-- 1. Find all actors who acted in "The Matrix"
MATCH (p:Person)-[:ACTED_IN]->(m:Movie {title: "The Matrix"})
RETURN p.name

-- 2. Find Tom Hanks' co-actors across all movies
MATCH (tom:Person {name: "Tom Hanks"})-[:ACTED_IN]->(m)<-[:ACTED_IN]-(co:Person)
RETURN DISTINCT co.name, count(m) AS sharedMovies
ORDER BY sharedMovies DESC

-- 3. Find the shortest path between any two actors
MATCH p = shortestPath(
  (a:Person {name: "Tom Hanks"})-[*]-(b:Person {name: "Keanu Reeves"})
)
RETURN [n IN nodes(p) | coalesce(n.name, n.title)] AS path

-- 4. Top 5 most connected actors
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN p.name, count(m) AS movies
ORDER BY movies DESC
LIMIT 5

-- 5. Find directors who also acted in their own movies
MATCH (p:Person)-[:DIRECTED]->(m:Movie)<-[:ACTED_IN]-(p)
RETURN p.name, m.title
```

---

> **Next Chapter** → [3E.3 — Neo4j Data Modeling & Graph Algorithms](./03-Neo4j-Modeling-Algorithms.md)
