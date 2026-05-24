# N+1 Query Problem & Batch Loading

> **What you'll learn**: How to detect and fix the most common performance killer in web applications — where your code secretly fires hundreds of database queries when it should fire just one.

---

## Real-Life Analogy

Imagine you're a **teacher** who needs to get the parent contact info for every student in your class of 30:

**N+1 approach** (what most ORMs do by default):
1. Get the class roster (1 trip to the office) → "Give me all students in Class 5A"
2. For EACH student, make a separate trip: "Give me the parents of Student #1"
3. "Give me the parents of Student #2"
4. "Give me the parents of Student #3"
5. ... (28 more trips!)
6. **Total: 31 trips to the office** 🐢

**Batch approach** (what you SHOULD do):
1. Get the class roster (1 trip) → "Give me all students in Class 5A"
2. "Give me the parents of ALL these 30 students at once" (1 trip)
3. **Total: 2 trips to the office** ⚡

```
N+1 Problem:                        Batch Loading:
┌──────┐     ┌────┐                ┌──────┐     ┌────┐
│ App  │────▶│ DB │ Query 1        │ App  │────▶│ DB │ Query 1
│      │────▶│    │ Query 2        │      │────▶│    │ Query 2
│      │────▶│    │ Query 3        │      │     │    │
│      │────▶│    │ Query 4        │      │     │    │ DONE!
│      │────▶│    │ Query 5        └──────┘     └────┘
│      │────▶│    │ ...            
│      │────▶│    │ Query N+1      2 queries total ✅
└──────┘     └────┘                (1 + 1 batch)
                                   Time: 6ms
31 queries total ❌
Time: 155ms
```

---

## Core Concept Explained Step-by-Step

### Step 1: What is the N+1 Problem?

You have a list of N items (e.g., 100 blog posts). For each item, your code fires a SEPARATE query to get related data (e.g., the author of each post).

```
-- Query 1: Get all posts (the "1" in N+1)
SELECT * FROM posts LIMIT 100;

-- Returns 100 posts. Then for EACH post:
-- Query 2: Get author of post #1 (the "N" part begins)
SELECT * FROM authors WHERE id = 7;
-- Query 3: Get author of post #2
SELECT * FROM authors WHERE id = 12;
-- Query 4: Get author of post #3
SELECT * FROM authors WHERE id = 7;   ← Same author, queried AGAIN!
-- ...
-- Query 101: Get author of post #100
SELECT * FROM authors WHERE id = 45;

Total: 1 + 100 = 101 queries! ❌
```

### Step 2: Why It's So Dangerous

```
Cost of N+1 with N = 100 posts:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Each query overhead:                                   │
│    • Network round-trip to DB: ~1ms                    │
│    • Query parsing + planning: ~0.2ms                  │
│    • Execution: ~0.3ms                                 │
│    • Result transfer: ~0.1ms                           │
│    ────────────────────────────                        │
│    Total per query: ~1.6ms                             │
│                                                         │
│  N+1 cost: 101 × 1.6ms = ~162ms  ❌                   │
│  Batch cost: 2 × 2ms = ~4ms      ✅                   │
│                                                         │
│  That's a 40x difference!                              │
│                                                         │
│  At scale (1000 posts): 1601ms vs 4ms = 400x slower!  │
└─────────────────────────────────────────────────────────┘
```

### Step 3: Where Does N+1 Come From?

Almost always from **ORMs with lazy loading**:

```python
# Django ORM — LAZY loading (default)
posts = Post.objects.all()[:100]  # 1 query

for post in posts:
    print(post.author.name)  # ← Triggers 1 query PER post!
    # Django says: "Oh, you want the author? Let me fetch it..."
    # This happens 100 times = 100 extra queries!
```

```java
// JPA/Hibernate — LAZY loading (default for @ManyToOne)
List<Post> posts = entityManager.createQuery("FROM Post", Post.class)
    .setMaxResults(100).getResultList();  // 1 query

for (Post post : posts) {
    System.out.println(post.getAuthor().getName());  // N queries!
    // Hibernate fires: SELECT * FROM authors WHERE id = ?
    // ...for EACH post
}
```

### Step 4: Solutions Overview

```
┌─────────────────────────────────────────────────────────────┐
│                  N+1 SOLUTIONS                              │
├──────────────────┬──────────────────────────────────────────┤
│                  │                                          │
│  1. Eager Load   │  Load related data in the SAME query    │
│     (JOIN)       │  SELECT posts.*, authors.* FROM posts   │
│                  │  JOIN authors ON posts.author_id = ...   │
│                  │                                          │
├──────────────────┼──────────────────────────────────────────┤
│                  │                                          │
│  2. Batch Load   │  Collect all IDs, then ONE IN query     │
│     (IN clause)  │  SELECT * FROM authors                  │
│                  │  WHERE id IN (7, 12, 45, ...)           │
│                  │                                          │
├──────────────────┼──────────────────────────────────────────┤
│                  │                                          │
│  3. Subquery     │  Use a subquery to fetch related data   │
│                  │  SELECT * FROM authors                  │
│                  │  WHERE id IN (SELECT author_id           │
│                  │               FROM posts LIMIT 100)     │
│                  │                                          │
├──────────────────┼──────────────────────────────────────────┤
│                  │                                          │
│  4. DataLoader   │  Batch + deduplicate at application     │
│     Pattern      │  level (Facebook's approach for GraphQL)│
│                  │                                          │
└──────────────────┴──────────────────────────────────────────┘
```

### Step 5: The DataLoader Pattern

Facebook invented the **DataLoader** pattern for GraphQL to solve N+1 at the application layer:

```
Request: "Get 100 posts with their authors"

Without DataLoader:
  post1.author → DB query (author_id=7)
  post2.author → DB query (author_id=12)
  post3.author → DB query (author_id=7)  ← DUPLICATE!
  ... (100 queries)

With DataLoader:
  1. Collect all requests in the same "tick":
     [author_id=7, author_id=12, author_id=7, author_id=45, ...]
  
  2. Deduplicate:
     unique IDs = [7, 12, 45, ...]
  
  3. Batch into ONE query:
     SELECT * FROM authors WHERE id IN (7, 12, 45, ...)
  
  4. Map results back to each requester

  Total: 1 query instead of 100! ✅

┌──────────────────────────────────────────────────┐
│              DataLoader Flow                      │
│                                                  │
│  Resolver 1 ──┐                                  │
│  Resolver 2 ──┤                                  │
│  Resolver 3 ──┤──▶ DataLoader ──▶ Batch Query   │
│  Resolver 4 ──┤    (collects,     (1 SQL query) │
│  ...         ──┤     dedupes,                    │
│  Resolver N ──┘     batches)                     │
└──────────────────────────────────────────────────┘
```

---

## How It Works Internally

### How ORMs Create N+1 (Lazy Proxy Pattern)

```
When you load a Post, the ORM doesn't load the Author immediately.
Instead, it creates a PROXY object:

┌─────────────────────────────────┐
│  Post Object                    │
│  ├── id: 42                     │
│  ├── title: "Hello World"       │
│  ├── author: LazyProxy(id=7)   │ ← Not loaded yet!
│  │           ├── loaded: false  │
│  │           └── load(): →      │──── fires SQL on first access
│  └── content: "..."             │
└─────────────────────────────────┘

When you access post.author.name:
  1. Proxy detects it hasn't loaded the real author
  2. Proxy fires: SELECT * FROM authors WHERE id = 7
  3. Proxy replaces itself with the real Author object
  4. Returns author.name

This is "transparent" — you don't see the query happening!
That's why N+1 is so insidious.
```

### How Batch Loading Works Internally

```
Django select_related (JOIN):
─────────────────────────────
  SELECT posts.*, authors.*
  FROM posts
  INNER JOIN authors ON posts.author_id = authors.id
  LIMIT 100;

  Result: 1 query, all data in one result set
  Downside: Duplicates author data if same author wrote multiple posts

Django prefetch_related (separate batch query):
──────────────────────────────────────────────
  Query 1: SELECT * FROM posts LIMIT 100;
  → Extract all author_ids: [7, 12, 7, 45, ...]
  → Deduplicate: [7, 12, 45, ...]
  
  Query 2: SELECT * FROM authors WHERE id IN (7, 12, 45, ...);
  → Map each author back to their posts in memory

  Result: 2 queries total, no duplicate data
```

### Detection: How to Spot N+1 in Production

```
Method 1: Query count logging
─────────────────────────────
  Before request: query_count = 0
  After request:  query_count = 103  ← RED FLAG! 🚨
  
  Rule of thumb: If query_count > 10 for a single API call,
  you probably have N+1

Method 2: Django Debug Toolbar / Rails Bullet gem
──────────────────────────────────────────────────
  Automatically detects and warns about N+1 queries

Method 3: Slow query log analysis
──────────────────────────────────
  Same query pattern repeated many times:
    SELECT * FROM authors WHERE id = ?  (executed 100x in 200ms)
  → N+1 detected!
```

---

## Code Examples

### Python — Django ORM (Before and After)

```python
# models.py
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField()

class Post(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    category = models.ForeignKey('Category', on_delete=models.CASCADE)

class Category(models.Model):
    name = models.CharField(max_length=50)

# ❌ BAD: N+1 query problem (103 queries!)
def get_posts_bad():
    posts = Post.objects.all()[:100]  # 1 query
    results = []
    for post in posts:
        results.append({
            "title": post.title,
            "author": post.author.name,      # +1 query per post
            "category": post.category.name,  # +1 query per post
        })
    return results  # Total: 1 + 100 + 100 = 201 queries! 💀

# ✅ GOOD: select_related (JOIN — 1 query)
def get_posts_good_join():
    posts = Post.objects.select_related('author', 'category')[:100]
    # Generates: SELECT posts.*, authors.*, categories.*
    #            FROM posts 
    #            JOIN authors ON ... JOIN categories ON ...
    results = []
    for post in posts:
        results.append({
            "title": post.title,
            "author": post.author.name,      # Already loaded!
            "category": post.category.name,  # Already loaded!
        })
    return results  # Total: 1 query! ✅

# ✅ GOOD: prefetch_related (2 separate batch queries)
# Better for ManyToMany or when you want to filter related objects
def get_posts_good_batch():
    posts = Post.objects.prefetch_related('author', 'category')[:100]
    # Query 1: SELECT * FROM posts LIMIT 100
    # Query 2: SELECT * FROM authors WHERE id IN (7, 12, 45, ...)
    # Query 3: SELECT * FROM categories WHERE id IN (1, 3, 5, ...)
    return posts  # Total: 3 queries ✅

# ✅ Detection: Count queries in tests
from django.test.utils import override_settings
from django.test import TestCase

class PostAPITest(TestCase):
    def test_no_n_plus_1(self):
        # Assert that the view uses at most 3 queries
        with self.assertNumQueries(3):
            response = self.client.get('/api/posts/')
            self.assertEqual(response.status_code, 200)
```

### Java — JPA/Hibernate (Before and After)

```java
import jakarta.persistence.*;
import java.util.List;

// Entity definitions
@Entity
public class Post {
    @Id @GeneratedValue
    private Long id;
    private String title;
    
    @ManyToOne(fetch = FetchType.LAZY) // ← Default: LAZY = N+1 trap!
    private Author author;
    
    @ManyToOne(fetch = FetchType.LAZY)
    private Category category;
}

@Entity
public class Author {
    @Id @GeneratedValue
    private Long id;
    private String name;
}

// ❌ BAD: N+1 problem
public class PostServiceBad {
    public List<PostDTO> getAllPosts(EntityManager em) {
        List<Post> posts = em.createQuery("FROM Post", Post.class)
            .setMaxResults(100)
            .getResultList();  // 1 query
        
        return posts.stream().map(p -> new PostDTO(
            p.getTitle(),
            p.getAuthor().getName(),  // ← N queries! (lazy load)
            p.getCategory().getName() // ← N more queries!
        )).toList();
        // Total: 1 + 100 + 100 = 201 queries 💀
    }
}

// ✅ GOOD: JOIN FETCH (loads in one query)
public class PostServiceGood {
    public List<PostDTO> getAllPosts(EntityManager em) {
        List<Post> posts = em.createQuery(
            "SELECT p FROM Post p " +
            "JOIN FETCH p.author " +      // ← Eagerly load author
            "JOIN FETCH p.category",       // ← Eagerly load category
            Post.class
        ).setMaxResults(100).getResultList();  // 1 query with JOINs!
        
        return posts.stream().map(p -> new PostDTO(
            p.getTitle(),
            p.getAuthor().getName(),  // Already loaded!
            p.getCategory().getName() // Already loaded!
        )).toList();
        // Total: 1 query ✅
    }
}

// ✅ GOOD: @BatchSize annotation (Hibernate batches lazy loads)
@Entity
public class PostWithBatch {
    @ManyToOne(fetch = FetchType.LAZY)
    @BatchSize(size = 25)  // Hibernate loads 25 authors at a time
    private Author author;
    // Instead of 100 queries, fires: 100/25 = 4 batch queries
}

// ✅ GOOD: EntityGraph (JPA standard way)
@Entity
@NamedEntityGraph(name = "Post.withAuthor",
    attributeNodes = @NamedAttributeNode("author"))
public class PostWithGraph {
    // ...
}

// Usage:
EntityGraph<?> graph = em.getEntityGraph("Post.withAuthor");
List<Post> posts = em.createQuery("FROM Post", Post.class)
    .setHint("jakarta.persistence.fetchgraph", graph)
    .getResultList();
```

### Python — DataLoader Pattern (for GraphQL)

```python
# dataloader.py — Facebook-style DataLoader
from collections import defaultdict
from typing import List, Dict, Any
import asyncio

class DataLoader:
    """Batches and deduplicates database lookups"""
    
    def __init__(self, batch_load_fn):
        self.batch_load_fn = batch_load_fn
        self._cache: Dict[Any, Any] = {}
        self._queue: List[Any] = []
        self._futures: Dict[Any, asyncio.Future] = {}
    
    async def load(self, key):
        """Load a single key — will be batched automatically"""
        if key in self._cache:
            return self._cache[key]
        
        if key not in self._futures:
            future = asyncio.get_event_loop().create_future()
            self._futures[key] = future
            self._queue.append(key)
            
            # Schedule batch execution at end of current "tick"
            asyncio.get_event_loop().call_soon(self._dispatch)
        
        return await self._futures[key]
    
    def _dispatch(self):
        """Execute the batched query"""
        if not self._queue:
            return
        keys = list(set(self._queue))  # Deduplicate!
        self._queue.clear()
        
        # Call the batch function (e.g., one SQL IN query)
        results = self.batch_load_fn(keys)
        
        for key in keys:
            result = results.get(key)
            self._cache[key] = result
            if key in self._futures:
                self._futures[key].set_result(result)

# Usage with a real database
def batch_load_authors(author_ids: List[int]) -> Dict[int, dict]:
    """Load multiple authors in ONE query"""
    # SELECT * FROM authors WHERE id IN (7, 12, 45, ...)
    rows = db.execute(
        "SELECT * FROM authors WHERE id = ANY(%s)", [author_ids]
    )
    return {row['id']: row for row in rows}

author_loader = DataLoader(batch_load_authors)

# In your GraphQL resolvers:
async def resolve_post_author(post):
    return await author_loader.load(post.author_id)
    # All resolvers in the same request batch into ONE query!
```

---

## Infrastructure Examples

### Detecting N+1 in Production with Query Logging

```yaml
# PostgreSQL: Log queries taking more than 50ms
# postgresql.conf
log_min_duration_statement = 50  # Log queries slower than 50ms
log_statement = 'none'            # Don't log all queries (too noisy)

# For development: log ALL queries and analyze patterns
# log_statement = 'all'
```

```python
# Django: Middleware to detect N+1 in development
import logging
from django.db import connection

class QueryCountMiddleware:
    """Warns when a request fires too many queries"""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        initial_queries = len(connection.queries)
        response = self.get_response(request)
        total_queries = len(connection.queries) - initial_queries
        
        if total_queries > 10:  # Threshold
            logging.warning(
                f"⚠️ N+1 ALERT: {request.path} fired {total_queries} queries!"
            )
        return response
```

### Automated N+1 Detection in CI/CD

```python
# pytest fixture to catch N+1 in tests
import pytest
from django.test.utils import CaptureQueriesContext
from django.db import connection

@pytest.fixture
def assert_max_queries():
    """Fixture that fails test if too many queries are fired"""
    class QueryAssertion:
        def __init__(self, max_queries):
            self.max_queries = max_queries
            self.context = CaptureQueriesContext(connection)
        
        def __enter__(self):
            self.context.__enter__()
            return self
        
        def __exit__(self, *args):
            self.context.__exit__(*args)
            actual = len(self.context.captured_queries)
            assert actual <= self.max_queries, (
                f"Expected at most {self.max_queries} queries, "
                f"but got {actual}!\n"
                f"Queries:\n" + 
                "\n".join(q['sql'] for q in self.context.captured_queries)
            )
    
    return QueryAssertion

# Usage in tests:
def test_post_list_api(client, assert_max_queries):
    with assert_max_queries(max_queries=3):
        response = client.get('/api/posts/')
        assert response.status_code == 200
```

---

## Real-World Example

### GitHub — N+1 in GraphQL

GitHub's GraphQL API had significant N+1 challenges:

```graphql
# This innocent-looking query could trigger N+1:
query {
  repository(owner: "facebook", name: "react") {
    issues(first: 100) {
      nodes {
        title
        author {        # ← N+1: loads each author separately!
          login
        }
        labels(first: 5) {  # ← N+1: loads labels for each issue!
          nodes { name }
        }
      }
    }
  }
}

# Without DataLoader: 1 + 100 + 100 = 201 queries
# With DataLoader:    1 + 1 + 1 = 3 queries (batched!)
```

### Shopify — Rails N+1 Detection

```
Shopify uses the "Bullet" gem to automatically detect N+1:
┌─────────────────────────────────────────────────────────┐
│  Shopify's approach:                                    │
│                                                         │
│  1. Bullet gem runs in development + staging            │
│  2. Detects N+1 patterns and suggests fixes             │
│  3. CI fails if N+1 is detected in new code            │
│  4. Production: query count tracking per endpoint       │
│                                                         │
│  Impact: Reduced average queries per page from          │
│  50+ down to 5-8 across their entire platform          │
└─────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why it's wrong | Fix |
|---------|---------------|-----|
| Using `FetchType.EAGER` everywhere | Loads ALL relations even when not needed (over-fetching) | Keep LAZY default, use JOIN FETCH when needed |
| Not testing query count | N+1 sneaks in with "innocent" code changes | Add query count assertions in tests |
| Batch size too large | `WHERE id IN (10000 items)` can be slow too | Batch in groups of 100-500 |
| Ignoring nested N+1 | posts → authors → addresses → cities (3 levels!) | Check EXPLAIN for multi-level relations |
| Caching individual objects | Caching author#7 still fires N cache lookups | Cache at the collection level or use DataLoader |
| Using `select_related` with ManyToMany | JOINs on M2M create cartesian products (data explosion) | Use `prefetch_related` for M2M relations |

---

## When to Use / When NOT to Use

### Fix N+1 WHEN:
- API response time is slow (>100ms for list endpoints)
- You see repeated similar queries in logs
- Query count for a single page exceeds 10
- Database CPU is high despite simple queries

### N+1 is acceptable WHEN:
- You're loading 1-3 items (overhead is negligible)
- The related data is already cached in Redis
- You're in a paginated cursor with batch_size set
- The query is in a background job (latency doesn't matter)

---

## Key Takeaways

- **N+1** means 1 query to get a list + N queries to get related data = N+1 total queries
- **ORMs with lazy loading** are the #1 cause — Hibernate, Django ORM, ActiveRecord all have this by default
- **Solutions**: JOIN FETCH (eager), `select_related`/`prefetch_related` (batch), or DataLoader (application-level batching)
- **Always test query count** in your test suite — N+1 regressions are silent killers
- **DataLoader pattern** (batch + deduplicate) is essential for GraphQL APIs
- The performance difference is dramatic: 200 queries × 1.5ms = 300ms → 2 queries × 2ms = 4ms
- **Detection tools**: Django Debug Toolbar, Bullet gem (Rails), Hibernate Statistics, query count middleware

---

## What's Next?

Now let's look at optimizing what gets sent over the wire. In **Chapter 19.6: Compression (Gzip, Brotli) & Minification**, you'll learn how to reduce response sizes by 60-80% — making your app faster without changing a single line of business logic.
