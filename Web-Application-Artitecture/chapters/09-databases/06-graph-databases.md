# Graph Databases (Neo4j, Amazon Neptune)

> **What you'll learn**: How graph databases model data as nodes and relationships, why they're exponentially faster than SQL for connected data, how traversal algorithms work internally, and when to use them for social networks, recommendations, and fraud detection.

---

## Real-Life Analogy

Imagine a **corkboard with photos and red strings** — like a detective's investigation board:

- **Photos** (nodes) = people, places, events, bank accounts
- **Red strings** (relationships) = "knows", "visited", "transferred money to"
- A detective asks: "Who are all the people connected to this suspect within 3 degrees?"

In a relational database, this question requires multiple self-JOINs — exponentially slower as the depth increases. In a graph database, it's a simple **traversal** — follow the strings! The query time depends only on the local neighborhood, not the total data size.

This is why fraud detection teams, social networks, and recommendation engines love graph databases.

---

## Core Concept Explained Step-by-Step

### The Graph Data Model

Everything is either a **Node** (entity) or a **Relationship** (connection):

```
┌──────────┐    FRIENDS_WITH     ┌──────────┐
│  Alice   │────────────────────▶│   Bob    │
│ (Person) │                     │ (Person) │
└──────┬───┘                     └────┬─────┘
       │                              │
       │ WORKS_AT                     │ LIVES_IN
       │                              │
       ▼                              ▼
┌──────────┐                     ┌──────────┐
│  Google  │                     │  Mumbai  │
│(Company) │                     │  (City)  │
└──────────┘                     └──────────┘

Nodes have:     Labels (Person, Company, City)
                Properties ({name: "Alice", age: 28})

Relationships:  Type (FRIENDS_WITH, WORKS_AT)
                Direction (Alice → Bob)
                Properties ({since: "2020-01-15"})
```

### Why Graphs Beat SQL for Connected Data

**Question**: "Find friends of friends of Alice who work at Google"

```
SQL Approach (3 self-JOINs):
──────────────────────────────────────────────────
SELECT DISTINCT p3.name
FROM friendships f1
JOIN friendships f2 ON f1.friend_id = f2.person_id
JOIN persons p3 ON f2.friend_id = p3.id
JOIN employment e ON p3.id = e.person_id
WHERE f1.person_id = (SELECT id FROM persons WHERE name = 'Alice')
AND e.company_id = (SELECT id FROM companies WHERE name = 'Google');

Performance: Scans millions of rows at each JOIN level
At depth 5: Billions of comparisons → TIMEOUT

Graph Approach (Traversal):
──────────────────────────────────────────────────
MATCH (alice:Person {name: "Alice"})-[:FRIENDS_WITH*2]->(fof:Person)
      -[:WORKS_AT]->(g:Company {name: "Google"})
RETURN DISTINCT fof.name

Performance: Only visits LOCAL connections
At depth 5: Still fast (visits thousands, not billions)
```

### Performance Comparison (Connected Data)

```
Number of          SQL (JOINs)         Graph (Traversal)
Connections        Response Time       Response Time
────────────────────────────────────────────────────────
Depth 2            ~0.1 sec            ~0.01 sec
Depth 3            ~2 sec              ~0.02 sec
Depth 4            ~30 sec             ~0.05 sec
Depth 5            ~TIMEOUT            ~0.1 sec
Depth 6            ~IMPOSSIBLE         ~0.3 sec

Why? SQL: O(n^depth) — exponential
     Graph: O(connected_nodes) — local
```

---

## How It Works Internally

### Native Graph Storage (Neo4j)

Neo4j uses **index-free adjacency** — each node physically stores pointers to its neighbors:

```
┌─────────────────────────────────────────────────────────────────┐
│                    NEO4J STORAGE ENGINE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Node Store:                                                     │
│  ┌────────────────────────────────────────────┐                 │
│  │ Node ID │ Labels │ First Relationship Ptr │ Properties Ptr │ │
│  │ 0       │ Person │ → Rel #5              │ → Props #12    │ │
│  │ 1       │ Person │ → Rel #8              │ → Props #15    │ │
│  └────────────────────────────────────────────┘                 │
│                                                                   │
│  Relationship Store:                                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Rel ID │ Type │ Start Node │ End Node │ Next Rel (Start)│   │
│  │ 5      │ KNOWS│ 0 (Alice) │ 1 (Bob)  │ → Rel #6       │   │
│  │ 6      │ WORKS│ 0 (Alice) │ 3 (Google)│ → NULL         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Traversal: Follow pointer chains (O(1) per hop!)               │
│  No index lookups needed between connected nodes                 │
└─────────────────────────────────────────────────────────────────┘
```

**Index-free adjacency** means:
- Getting neighbors of a node = following fixed-size pointers
- Cost per hop = O(1), regardless of total graph size
- A graph with 1 billion nodes has the same traversal speed for local queries as one with 1,000 nodes

### Graph Query Execution

```
Query: MATCH (a:Person {name:"Alice"})-[:FRIENDS*1..3]->(f)

Execution Plan:
┌─────────────────────────────────────────────────────┐
│ 1. Index Lookup: Find "Alice" by name index         │
│    → Node ID: 42                                    │
│                                                     │
│ 2. Expand (depth 1):                               │
│    Follow all FRIENDS relationships from node 42    │
│    → Found: [Bob(15), Carol(23), Dave(31)]         │
│                                                     │
│ 3. Expand (depth 2):                               │
│    For each: follow their FRIENDS relationships     │
│    → Bob's friends: [Eve(45), Frank(52)]           │
│    → Carol's friends: [Grace(61)]                  │
│    → Dave's friends: [Henry(70), Iris(78)]         │
│                                                     │
│ 4. Expand (depth 3): ...continue...                │
│                                                     │
│ 5. Filter: Remove duplicates, apply conditions      │
│                                                     │
│ 6. Return results                                   │
└─────────────────────────────────────────────────────┘
```

### Cypher Query Language (Neo4j)

Cypher is a visual pattern-matching language:

```cypher
-- The pattern looks like the graph itself!
-- (node)-[relationship]->(node)

-- Create nodes and relationships
CREATE (alice:Person {name: "Alice", age: 28})
CREATE (bob:Person {name: "Bob", age: 35})
CREATE (alice)-[:FRIENDS_WITH {since: 2020}]->(bob)

-- Pattern matching
MATCH (p:Person)-[:WORKS_AT]->(c:Company {name: "Google"})
RETURN p.name, p.age

-- Variable-length paths (1 to 5 hops)
MATCH path = (start:Person {name: "Alice"})-[:FRIENDS_WITH*1..5]->(end:Person)
RETURN end.name, length(path) AS distance

-- Shortest path
MATCH path = shortestPath(
  (alice:Person {name: "Alice"})-[:FRIENDS_WITH*]-(bob:Person {name: "Bob"})
)
RETURN path
```

---

## Code Examples

### Python (Neo4j Driver)

```python
from neo4j import GraphDatabase

# Connect to Neo4j
driver = GraphDatabase.driver(
    "bolt://localhost:7687",
    auth=("neo4j", "password")
)

def create_social_network(tx):
    """Create a sample social network"""
    tx.run("""
        CREATE (alice:Person {name: 'Alice', age: 28, city: 'Mumbai'})
        CREATE (bob:Person {name: 'Bob', age: 35, city: 'Delhi'})
        CREATE (carol:Person {name: 'Carol', age: 30, city: 'Mumbai'})
        CREATE (dave:Person {name: 'Dave', age: 26, city: 'Pune'})
        
        CREATE (google:Company {name: 'Google', industry: 'Tech'})
        CREATE (amazon:Company {name: 'Amazon', industry: 'Tech'})
        
        CREATE (alice)-[:FRIENDS_WITH {since: 2019}]->(bob)
        CREATE (alice)-[:FRIENDS_WITH {since: 2020}]->(carol)
        CREATE (bob)-[:FRIENDS_WITH {since: 2021}]->(dave)
        CREATE (carol)-[:FRIENDS_WITH {since: 2022}]->(dave)
        
        CREATE (alice)-[:WORKS_AT {role: 'Engineer'}]->(google)
        CREATE (bob)-[:WORKS_AT {role: 'Manager'}]->(amazon)
        CREATE (dave)-[:WORKS_AT {role: 'Intern'}]->(google)
    """)

def find_friend_recommendations(tx, person_name):
    """Find friends-of-friends who aren't already friends"""
    result = tx.run("""
        MATCH (me:Person {name: $name})-[:FRIENDS_WITH]->(friend)
              -[:FRIENDS_WITH]->(recommendation)
        WHERE recommendation <> me
        AND NOT (me)-[:FRIENDS_WITH]->(recommendation)
        RETURN recommendation.name AS suggested,
               COUNT(friend) AS mutual_friends,
               COLLECT(friend.name) AS through
        ORDER BY mutual_friends DESC
    """, name=person_name)
    return [dict(record) for record in result]

def detect_fraud_ring(tx, account_id, max_depth=4):
    """Find suspicious circular money transfers"""
    result = tx.run("""
        MATCH path = (start:Account {id: $account_id})
              -[:TRANSFERRED_TO*2..%d]->(start)
        WHERE ALL(r IN relationships(path) WHERE r.amount > 10000)
        RETURN path, 
               REDUCE(total = 0, r IN relationships(path) | total + r.amount) AS total_amount
        ORDER BY total_amount DESC
        LIMIT 10
    """ % max_depth, account_id=account_id)
    return [dict(record) for record in result]

# Execute
with driver.session() as session:
    session.execute_write(create_social_network)
    
    recommendations = session.execute_read(find_friend_recommendations, "Alice")
    for rec in recommendations:
        print(f"Suggest: {rec['suggested']} ({rec['mutual_friends']} mutual via {rec['through']})")

driver.close()
```

### Java (Neo4j Driver)

```java
import org.neo4j.driver.*;
import java.util.List;
import java.util.Map;

public class GraphDBExample {
    public static void main(String[] args) {
        Driver driver = GraphDatabase.driver(
            "bolt://localhost:7687",
            AuthTokens.basic("neo4j", "password"));

        try (Session session = driver.session()) {
            // Find shortest path between two people
            Result result = session.run("""
                MATCH path = shortestPath(
                    (a:Person {name: $from})-[:FRIENDS_WITH*]-(b:Person {name: $to})
                )
                RETURN [n IN nodes(path) | n.name] AS names,
                       length(path) AS hops
            """, Map.of("from", "Alice", "to", "Dave"));

            while (result.hasNext()) {
                Record record = result.next();
                List<Object> names = record.get("names").asList();
                long hops = record.get("hops").asLong();
                System.out.printf("Path (%d hops): %s%n", hops, names);
            }

            // Page rank — find most influential nodes
            session.run("""
                CALL gds.pageRank.stream('social-graph')
                YIELD nodeId, score
                RETURN gds.util.asNode(nodeId).name AS name, score
                ORDER BY score DESC LIMIT 10
            """).forEachRemaining(record -> {
                System.out.printf("%s: %.4f influence%n",
                    record.get("name").asString(),
                    record.get("score").asDouble());
            });
        }
        driver.close();
    }
}
```

---

## Infrastructure Examples

### Neo4j Docker Setup

```yaml
version: '3.8'
services:
  neo4j:
    image: neo4j:5.15
    ports:
      - "7474:7474"  # Browser UI
      - "7687:7687"  # Bolt protocol
    environment:
      - NEO4J_AUTH=neo4j/secretpassword
      - NEO4J_PLUGINS=["apoc", "graph-data-science"]
      - NEO4J_dbms_memory_heap_initial__size=1G
      - NEO4J_dbms_memory_heap_max__size=2G
      - NEO4J_dbms_memory_pagecache_size=2G
    volumes:
      - neo4j-data:/data
      - neo4j-logs:/logs

volumes:
  neo4j-data:
  neo4j-logs:
```

### Amazon Neptune (Terraform)

```hcl
resource "aws_neptune_cluster" "main" {
  cluster_identifier  = "social-graph"
  engine              = "neptune"
  engine_version      = "1.3.0.0"
  
  backup_retention_period = 7
  preferred_backup_window = "03:00-04:00"
  storage_encrypted       = true
  
  vpc_security_group_ids = [aws_security_group.neptune.id]
  neptune_subnet_group_name = aws_neptune_subnet_group.main.name
}

resource "aws_neptune_cluster_instance" "main" {
  count              = 3
  cluster_identifier = aws_neptune_cluster.main.id
  instance_class     = "db.r6g.xlarge"
  engine             = "neptune"
}
```

---

## Real-World Example

### LinkedIn — Social Graph at Scale

```
┌─────────────────────────────────────────────────────────────────┐
│           LinkedIn's Graph Database Usage                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  900+ Million members, each with:                                │
│  • ~500 connections (avg)                                        │
│  • Skills, companies, education                                  │
│  • Interactions (views, messages, endorsements)                  │
│                                                                   │
│  ┌─────────────────────────────────────────────┐                │
│  │ Key Graph Queries:                          │                │
│  │                                              │                │
│  │ "People You May Know" (PYMK)                │                │
│  │ → 2nd/3rd degree connections                │                │
│  │ → Weighted by mutual connections            │                │
│  │                                              │                │
│  │ "Who Viewed Your Profile"                   │                │
│  │ → Traverse viewer → connections → you       │                │
│  │                                              │                │
│  │ "Jobs You Might Like"                       │                │
│  │ → Company graph + skills graph + location   │                │
│  └─────────────────────────────────────────────┘                │
│                                                                   │
│  Technology: Custom graph engine (LIquid)                        │
│  Scale: Trillions of edges, sub-50ms queries                    │
│  Storage: Distributed across thousands of servers               │
└─────────────────────────────────────────────────────────────────┘
```

### Fraud Detection at PayPal

```
Fraud Pattern: Money Laundering Ring
───────────────────────────────────────
Account A ──$10K──▶ Account B ──$9.8K──▶ Account C ──$9.5K──▶ Account A
    │                                                              ▲
    └──────────────── Circular Transfer (RED FLAG!) ───────────────┘

Graph Query:
MATCH (a:Account)-[:TRANSFER*3..6]->(a)
WHERE ALL(t IN relationships(path) WHERE t.amount > 5000)
AND ALL(t IN relationships(path) WHERE t.timestamp > datetime() - duration('P7D'))
RETURN a, path

SQL equivalent: Recursive CTEs with 6+ self-JOINs → EXTREMELY slow
Graph: Sub-second detection!
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using graphs for non-graph data | Overhead without traversal benefit | Use graph ONLY when relationships are the query focus |
| Super-nodes (nodes with millions of edges) | Traversal becomes slow at those nodes | Add relationship types, filter on properties |
| Not using indexes on starting nodes | Full scan to find starting point | Create index on frequently searched properties |
| Modeling everything as properties | Can't traverse properties | If you query by it, make it a node |
| Unbounded traversals | Exploring the entire graph | Always set max depth (`*1..5`) |
| Using graphs for analytics on all data | Aggregations are not graph's strength | Use SQL/Spark for aggregate analytics |
| Not considering data import strategy | Loading billions of nodes is slow | Use batch import tools (neo4j-admin import) |

---

## When to Use / When NOT to Use

### ✅ Use Graph Databases When:
- Queries involve **traversing relationships** (friends-of-friends, shortest path)
- Data is **naturally connected** (social networks, org charts, dependency trees)
- You need **real-time recommendations** ("people who bought X also bought Y")
- **Fraud detection** requiring pattern matching across connections
- **Knowledge graphs** (connecting entities with meaning)
- Depth of relationship queries is **variable and deep** (3+ hops)
- **Network analysis** (influence, centrality, community detection)

### ❌ Do NOT Use Graph Databases When:
- Simple CRUD operations on individual records
- Data is **tabular** with few relationships
- You need **high write throughput** (graphs are read-optimized)
- Queries are **aggregations** (SUM, COUNT, AVG across all data)
- Dataset is **small** (< 1M nodes — SQL handles this fine)
- **Full-text search** is the primary need (use Elasticsearch)
- **Time-series data** (use Cassandra/TimescaleDB)

---

## Key Takeaways

1. **Graph databases** model data as nodes and relationships — perfect for highly connected data where traversal patterns are the primary query type.
2. **Index-free adjacency** gives O(1) traversal per hop, making graph databases exponentially faster than SQL for deep relationship queries.
3. **Cypher** (Neo4j) and **Gremlin** (Neptune, TinkerPop) are the two main graph query languages — Cypher is more readable, Gremlin is more portable.
4. **Use cases**: social networks, recommendations, fraud detection, knowledge graphs, dependency analysis, access control.
5. **Graph databases are NOT general-purpose** — they excel at traversal but struggle with aggregations, bulk operations, and high write throughput.
6. **Data modeling** in graphs: if you query relationships, make it a graph. If you query properties of individual records, use SQL.
7. **At scale**, companies like LinkedIn, Facebook, and Twitter build custom graph engines because general-purpose graph DBs can't handle billions of edges with sub-ms latency.

---

## What's Next?

Next, we'll explore **Time-Series Databases (InfluxDB, TimescaleDB)** (Chapter 9.7), specialized databases designed for storing and querying timestamped data like metrics, IoT sensor readings, and financial tick data.
