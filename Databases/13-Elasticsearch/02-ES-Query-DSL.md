# 🔍 Chapter 3F.2 — Elasticsearch Querying (Query DSL)

> **"Elasticsearch's Query DSL is like having a conversation with your data — ask anything, in any way, and get ranked answers in milliseconds."**

> **Level:** 🟡 Intermediate | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 3F.1 (ES Architecture & Concepts)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Write **any type** of Elasticsearch query — from simple matches to complex booleans
- Understand **Query Context vs Filter Context** (and why it matters for performance)
- Build powerful **Bool queries** combining must, should, must_not, filter
- Use **Aggregations** to analyze data (group, count, average, histogram, etc.)
- Implement **fuzzy search, autocomplete, highlighting, and suggesters**
- Write production-grade queries that power real search bars

---

## 🧱 1. The Two Contexts — Query vs Filter

> **This is the #1 concept to understand before writing any query.**

```
┌─────────────────────────────────────────────────────────────┐
│                    QUERY CONTEXT                             │
│                                                              │
│  "How WELL does this document match?"                       │
│                                                              │
│  ✅ Calculates relevance _score                              │
│  ✅ Full-text search ("find products about running shoes")  │
│  ❌ Slower (must score every matching doc)                   │
│  ❌ Results NOT cached                                       │
│                                                              │
│  Used in: query { match, multi_match, bool.must/should }    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    FILTER CONTEXT                            │
│                                                              │
│  "Does this document match? YES or NO."                     │
│                                                              │
│  ✅ No scoring — binary yes/no                               │
│  ✅ MUCH faster (no score calculation)                       │
│  ✅ Results are CACHED (bitsets)                              │
│  ✅ Use for: dates, ranges, status, categories, boolean     │
│                                                              │
│  Used in: bool.filter, bool.must_not, constant_score        │
└─────────────────────────────────────────────────────────────┘
```

> ⭐ **Golden Rule:** Use **query context** for full-text search (where relevance matters). Use **filter context** for exact matches, ranges, and boolean conditions (where you just need yes/no).

```json
// ✅ OPTIMAL: Combine both contexts
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "gaming laptop" } }       // Query context (scored)
      ],
      "filter": [                                         // Filter context (not scored, cached!)
        { "term": { "brand": "ASUS" } },
        { "range": { "price": { "lte": 2000 } } },
        { "term": { "in_stock": true } }
      ]
    }
  }
}
// ✅ "gaming laptop" → relevance scored
// ✅ brand, price, stock → fast binary filters, cached
```

---

## 🔤 2. Full-Text Search Queries

### 2.1 — `match` — The Workhorse Query

The most commonly used query. Analyzes the search text, then looks it up in the inverted index.

```json
// Basic match — finds docs containing "quick" OR "fox"
GET /articles/_search
{
  "query": {
    "match": {
      "title": "quick fox"
    }
  }
}
// Behind the scenes:
// 1. Analyze "quick fox" → tokens: ["quick", "fox"]
// 2. Search: title contains "quick" OR "fox"
// 3. Score: docs with BOTH terms score highest

// Match with AND operator — BOTH terms required
GET /articles/_search
{
  "query": {
    "match": {
      "title": {
        "query": "quick fox",
        "operator": "and"           // Both "quick" AND "fox" must appear
      }
    }
  }
}

// Minimum should match — at least 2 out of 3 terms
GET /articles/_search
{
  "query": {
    "match": {
      "title": {
        "query": "quick brown fox",
        "minimum_should_match": "2"   // At least 2 of the 3 terms
      }
    }
  }
}
```

### 2.2 — `match_phrase` — Exact Phrase Matching

```json
// Order matters! Finds "quick brown fox" as a continuous phrase
GET /articles/_search
{
  "query": {
    "match_phrase": {
      "title": "quick brown fox"
    }
  }
}
// ✅ Matches: "The quick brown fox jumps"
// ❌ No match: "The quick red fox" (brown is missing between quick and fox)
// ❌ No match: "The fox is quick and brown" (wrong order)

// Allow "slop" — words can be N positions apart
GET /articles/_search
{
  "query": {
    "match_phrase": {
      "title": {
        "query": "quick fox",
        "slop": 1               // Allow 1 word between "quick" and "fox"
      }
    }
  }
}
// ✅ Matches: "quick brown fox" (1 word gap)
// ❌ No match: "quick large brown fox" (2 word gap)
```

### 2.3 — `multi_match` — Search Across Multiple Fields

```json
// Search "laptop" in name, description, and tags
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "gaming laptop",
      "fields": ["name^3", "description", "tags^2"],    // ^3 = name is 3x more important
      "type": "best_fields"                                // Use score from best matching field
    }
  }
}
// ^3 boost on "name" means a match in name scores 3x higher than description
```

**Multi-match types:**

| Type | Behavior | Best For |
|------|----------|----------|
| `best_fields` | Score from the single best matching field | Default. Most queries |
| `most_fields` | Sum scores from ALL matching fields | Fields with same text in different analyzers |
| `cross_fields` | Treats all fields as one big field | Person name split across first_name, last_name |
| `phrase` | Runs match_phrase on each field | Phrase search across multiple fields |
| `phrase_prefix` | Like phrase but allows prefix on last term | Autocomplete across fields |

### 2.4 — `match_phrase_prefix` — Search-As-You-Type (Autocomplete)

```json
// User is typing: "macb" → show suggestions
GET /products/_search
{
  "query": {
    "match_phrase_prefix": {
      "name": {
        "query": "macb",
        "max_expansions": 10        // Limit prefix expansions (performance)
      }
    }
  }
}
// ✅ Matches: "MacBook Pro", "MacBook Air", "Macbeth Returns"
```

---

## 🎯 3. Term-Level Queries (Exact Match — No Analysis)

> **Term queries work on the EXACT value stored in the inverted index. No analysis is applied to your search term.**

### 3.1 — `term` — Exact Single Value

```json
// Find all products by brand (keyword field — exact match)
GET /products/_search
{
  "query": {
    "term": {
      "brand": "Apple"              // Must match EXACTLY (case-sensitive for keywords!)
    }
  }
}

// ⚠️ COMMON MISTAKE: Using term query on text fields
GET /products/_search
{
  "query": {
    "term": {
      "name": "MacBook Pro"         // ❌ WON'T WORK! 
      // "name" is a text field → stored as ["macbook", "pro"]
      // term query searches for EXACT "MacBook Pro" → not found!
    }
  }
}

// ✅ CORRECT: Use name.keyword for exact match on text multi-fields
GET /products/_search
{
  "query": {
    "term": {
      "name.keyword": "MacBook Pro"    // ✅ Matches the keyword sub-field
    }
  }
}
```

### 3.2 — `terms` — Match Any of Multiple Values (SQL IN)

```json
// Like SQL: WHERE brand IN ('Apple', 'Samsung', 'Google')
GET /products/_search
{
  "query": {
    "terms": {
      "brand": ["Apple", "Samsung", "Google"]
    }
  }
}
```

### 3.3 — `range` — Numeric/Date Ranges

```json
// Price between 500 and 1500
GET /products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 500,              // Greater than or equal
        "lte": 1500              // Less than or equal
      }
    }
  }
}

// Date range — last 30 days
GET /logs/_search
{
  "query": {
    "range": {
      "timestamp": {
        "gte": "now-30d/d",      // 30 days ago (rounded to day)
        "lte": "now/d"           // Today
      }
    }
  }
}

// Date math examples:
// "now-1h"     → 1 hour ago
// "now-7d"     → 7 days ago
// "now-1M"     → 1 month ago
// "now/d"      → Start of today
// "now/M"      → Start of this month
// "2024-01-01||+1M"  → February 1, 2024
```

### 3.4 — `exists` — Field Exists Check

```json
// Find products that HAVE a "discount" field
GET /products/_search
{
  "query": {
    "exists": {
      "field": "discount"
    }
  }
}

// Find products WITHOUT a "discount" field
GET /products/_search
{
  "query": {
    "bool": {
      "must_not": [
        { "exists": { "field": "discount" } }
      ]
    }
  }
}
```

### 3.5 — `wildcard` and `regexp` — Pattern Matching

```json
// Wildcard (* = any chars, ? = single char)
GET /products/_search
{
  "query": {
    "wildcard": {
      "brand": {
        "value": "Sam*"          // Samsung, Samsonite, etc.
      }
    }
  }
}
// ⚠️ Avoid leading wildcards ("*sung") — very slow! (full index scan)

// Regex
GET /products/_search
{
  "query": {
    "regexp": {
      "product_code": {
        "value": "PROD-[0-9]{4}-[A-Z]{2}"    // PROD-1234-AB
      }
    }
  }
}
```

### 3.6 — `prefix` — Starts With

```json
GET /products/_search
{
  "query": {
    "prefix": {
      "brand": {
        "value": "Sam"           // Samsung, Samsonite...
      }
    }
  }
}
```

---

## 🧠 4. Bool Query — The Query Composer

> **The Bool query is the most powerful query in Elasticsearch. 90% of production queries use it.**

```
┌───────────────────────────────────────────────────────────────┐
│                      BOOL QUERY                                │
│                                                                │
│  ┌──────────┬────────────────────────────────────────────────┐│
│  │ must     │ MUST match (AND). Contributes to score.        ││
│  ├──────────┼────────────────────────────────────────────────┤│
│  │ should   │ SHOULD match (OR). Boosts score if matched.   ││
│  ├──────────┼────────────────────────────────────────────────┤│
│  │ must_not │ MUST NOT match (NOT). No scoring. Cached.     ││
│  ├──────────┼────────────────────────────────────────────────┤│
│  │ filter   │ MUST match but NO scoring. Cached. Fastest!   ││
│  └──────────┴────────────────────────────────────────────────┘│
│                                                                │
│  Think of it as:                                               │
│  must     = AND (scored)                                       │
│  should   = OR  (scored, boost)                                │
│  must_not = NOT (filtered, cached)                             │
│  filter   = AND (filtered, cached, fastest)                    │
└───────────────────────────────────────────────────────────────┘
```

### 4.1 — Real-World Bool Query Examples

```json
// E-commerce search: "gaming laptop" under $2000, in-stock, not refurbished
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "gaming laptop" } }          // Full-text, scored
      ],
      "should": [
        { "term": { "brand": "ASUS" } },                   // Boost ASUS
        { "range": { "ratings.average": { "gte": 4.5 } } } // Boost high-rated
      ],
      "must_not": [
        { "term": { "condition": "refurbished" } }         // Exclude refurbished
      ],
      "filter": [
        { "range": { "price": { "lte": 2000 } } },        // Under $2000 (no scoring)
        { "term": { "in_stock": true } }                   // Must be in stock (no scoring)
      ],
      "minimum_should_match": 0   // should clauses are optional boosters
    }
  }
}
```

```json
// Log search: ERROR logs from payment-service in the last 24h
GET /logs/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "ERROR" } },
        { "term": { "service": "payment-service" } },
        { "range": { "timestamp": { "gte": "now-24h" } } }
      ]
    }
  },
  "sort": [
    { "timestamp": { "order": "desc" } }
  ],
  "size": 100
}
// All filters → no scoring needed → maximum performance ⚡
```

### 4.2 — Nested Bool (Complex Logic)

```json
// (brand = "Apple" OR brand = "Samsung") AND (price < 1000) AND (has discount OR rating > 4.5)
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "bool": {                                  // Nested bool for OR
            "should": [
              { "term": { "brand": "Apple" } },
              { "term": { "brand": "Samsung" } }
            ],
            "minimum_should_match": 1               // At least one must match
          }
        }
      ],
      "filter": [
        { "range": { "price": { "lt": 1000 } } }
      ],
      "should": [
        { "exists": { "field": "discount" } },
        { "range": { "ratings.average": { "gt": 4.5 } } }
      ],
      "minimum_should_match": 1
    }
  }
}
```

---

## 🔄 5. Fuzzy Search — Typo Tolerance

> **Users make typos. Great search handles them gracefully.**

### 5.1 — How Fuzzy Works

Fuzzy matching uses **Levenshtein Distance** — the number of single-character edits needed:

```
"iphne" → "iphone"     Distance: 1 (swap h and n) ✅ Found!
"laptp" → "laptop"     Distance: 1 (insert o)      ✅ Found!
"samsg" → "samsung"    Distance: 2 (insert u, n)    ✅ Found! (if fuzziness=2)
"xyz"   → "samsung"    Distance: 5                   ❌ Too different
```

### 5.2 — Fuzzy Query

```json
// Fuzzy term search
GET /products/_search
{
  "query": {
    "fuzzy": {
      "brand": {
        "value": "Samsng",           // Typo! Missing 'u'
        "fuzziness": "AUTO"          // Auto-adjusts based on term length
      }
    }
  }
}
// AUTO fuzziness:
// 0-2 chars  → exact match only (no fuzziness)
// 3-5 chars  → fuzziness: 1 (allow 1 edit)
// 6+ chars   → fuzziness: 2 (allow 2 edits)

// Fuzzy within match query
GET /products/_search
{
  "query": {
    "match": {
      "name": {
        "query": "iphne pro",
        "fuzziness": "AUTO"          // Each TERM gets fuzzy matching
      }
    }
  }
}
// "iphne" → fuzzy matches "iphone" ✅
// "pro"   → exact match ✅
```

---

## 🌍 6. Geo Queries — Location-Based Search

```json
// Setup: field must be "geo_point" type
PUT /restaurants
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "location": { "type": "geo_point" }
    }
  }
}

// Find restaurants within 5km of a point
GET /restaurants/_search
{
  "query": {
    "geo_distance": {
      "distance": "5km",
      "location": {
        "lat": 40.7128,               // New York City
        "lon": -74.0060
      }
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "location": { "lat": 40.7128, "lon": -74.0060 },
        "order": "asc",               // Nearest first
        "unit": "km"
      }
    }
  ]
}

// Bounding box (rectangle area)
GET /restaurants/_search
{
  "query": {
    "geo_bounding_box": {
      "location": {
        "top_left":     { "lat": 40.73, "lon": -74.01 },
        "bottom_right": { "lat": 40.70, "lon": -73.98 }
      }
    }
  }
}
```

---

## 📊 7. Aggregations — Analytics Powerhouse

> **Aggregations are Elasticsearch's answer to SQL's GROUP BY — but on steroids.**

```
SQL Equivalent Mapping:
  COUNT(*)         →  value_count / cardinality
  SUM(price)       →  sum aggregation
  AVG(rating)      →  avg aggregation
  MIN/MAX          →  min / max aggregation
  GROUP BY brand   →  terms aggregation
  HAVING           →  pipeline aggregation
  HISTOGRAM        →  histogram / date_histogram
```

### 7.1 — Metric Aggregations (Single Value Results)

```json
// Average, Min, Max, Sum, Count
GET /products/_search
{
  "size": 0,                            // Don't return search hits, only aggs
  "aggs": {
    "avg_price": {
      "avg": { "field": "price" }
    },
    "max_price": {
      "max": { "field": "price" }
    },
    "min_price": {
      "min": { "field": "price" }
    },
    "total_revenue": {
      "sum": { "field": "price" }
    },
    "unique_brands": {
      "cardinality": { "field": "brand" }     // COUNT(DISTINCT brand)
    }
  }
}

// Response:
{
  "aggregations": {
    "avg_price":      { "value": 849.99 },
    "max_price":      { "value": 3499.99 },
    "min_price":      { "value": 29.99 },
    "total_revenue":  { "value": 2548500.00 },
    "unique_brands":  { "value": 47 }
  }
}

// Stats aggregation — all metrics in ONE call
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_stats": {
      "stats": { "field": "price" }
    }
  }
}
// Returns: count, min, max, avg, sum — all at once!
```

### 7.2 — Bucket Aggregations (Grouping)

```json
// GROUP BY brand (like SQL: SELECT brand, COUNT(*) FROM products GROUP BY brand)
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_brand": {
      "terms": {
        "field": "brand",
        "size": 10,                    // Top 10 brands
        "order": { "_count": "desc" }  // Most products first
      }
    }
  }
}

// Response:
{
  "aggregations": {
    "by_brand": {
      "buckets": [
        { "key": "Apple",    "doc_count": 156 },
        { "key": "Samsung",  "doc_count": 142 },
        { "key": "Dell",     "doc_count": 98 },
        { "key": "HP",       "doc_count": 87 },
        { "key": "Lenovo",   "doc_count": 76 }
      ]
    }
  }
}
```

### 7.3 — Nested Aggregations (GROUP BY + Metrics)

```json
// Average price BY brand (SQL: SELECT brand, AVG(price) FROM products GROUP BY brand)
GET /products/_search
{
  "size": 0,
  "aggs": {
    "by_brand": {
      "terms": { "field": "brand", "size": 10 },
      "aggs": {                                    // Sub-aggregation inside each bucket
        "avg_price": {
          "avg": { "field": "price" }
        },
        "price_range": {
          "stats": { "field": "price" }            // Full stats per brand
        }
      }
    }
  }
}

// Response:
{
  "aggregations": {
    "by_brand": {
      "buckets": [
        {
          "key": "Apple",
          "doc_count": 156,
          "avg_price": { "value": 1299.50 },
          "price_range": { "min": 429, "max": 3499, "avg": 1299.50 }
        },
        {
          "key": "Samsung",
          "doc_count": 142,
          "avg_price": { "value": 749.99 },
          "price_range": { "min": 199, "max": 1999, "avg": 749.99 }
        }
      ]
    }
  }
}
```

### 7.4 — Date Histogram (Time-Series Analysis)

```json
// Errors per hour in the last 24 hours
GET /logs/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "ERROR" } },
        { "range": { "timestamp": { "gte": "now-24h" } } }
      ]
    }
  },
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "timestamp",
        "calendar_interval": "1h",       // 1 bucket per hour
        "format": "yyyy-MM-dd HH:mm",
        "min_doc_count": 0               // Show hours with 0 errors too
      }
    }
  }
}

// Response:
{
  "aggregations": {
    "errors_over_time": {
      "buckets": [
        { "key_as_string": "2024-06-14 10:00", "doc_count": 3 },
        { "key_as_string": "2024-06-14 11:00", "doc_count": 0 },
        { "key_as_string": "2024-06-14 12:00", "doc_count": 15 },  // ← spike!
        { "key_as_string": "2024-06-14 13:00", "doc_count": 47 },  // ← 🔥 incident!
        { "key_as_string": "2024-06-14 14:00", "doc_count": 2 }
      ]
    }
  }
}
```

### 7.5 — Range Aggregation (Custom Buckets)

```json
// Price ranges: budget, mid-range, premium, luxury
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "key": "Budget (< $300)",     "to": 300 },
          { "key": "Mid-Range ($300-800)", "from": 300, "to": 800 },
          { "key": "Premium ($800-1500)",  "from": 800, "to": 1500 },
          { "key": "Luxury ($1500+)",      "from": 1500 }
        ]
      }
    }
  }
}
```

### 7.6 — Filter Aggregation (Conditional Aggs)

```json
// Compare average price: Apple vs Samsung
GET /products/_search
{
  "size": 0,
  "aggs": {
    "apple_stats": {
      "filter": { "term": { "brand": "Apple" } },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    },
    "samsung_stats": {
      "filter": { "term": { "brand": "Samsung" } },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    }
  }
}
```

### 7.7 — Aggregation Cheat Sheet

| Aggregation | SQL Equivalent | Type | Use Case |
|------------|---------------|------|----------|
| `avg` | `AVG(field)` | Metric | Average price |
| `sum` | `SUM(field)` | Metric | Total revenue |
| `min` / `max` | `MIN(field)` / `MAX(field)` | Metric | Price range |
| `cardinality` | `COUNT(DISTINCT field)` | Metric | Unique visitors |
| `value_count` | `COUNT(field)` | Metric | Total orders |
| `stats` | Multiple stats at once | Metric | Quick overview |
| `percentiles` | `PERCENTILE_CONT(...)` | Metric | P95 latency |
| `terms` | `GROUP BY field` | Bucket | Top categories |
| `date_histogram` | `GROUP BY DATE_TRUNC(...)` | Bucket | Time series |
| `histogram` | Numeric range buckets | Bucket | Price distribution |
| `range` | `CASE WHEN...` | Bucket | Custom ranges |
| `filter` | `WHERE ... GROUP BY` | Bucket | Conditional agg |
| `nested` | N/A | Bucket | Nested object aggs |
| `top_hits` | Subquery top N | Metric | Top docs per bucket |

---

## ✨ 8. Highlighting — Show WHY a Result Matched

```json
GET /products/_search
{
  "query": {
    "match": { "description": "powerful laptop M3 chip" }
  },
  "highlight": {
    "pre_tags": ["<mark>"],            // HTML tag to wrap matches
    "post_tags": ["</mark>"],
    "fields": {
      "description": {
        "fragment_size": 150,          // Max chars per fragment
        "number_of_fragments": 3       // Max fragments to return
      }
    }
  }
}

// Response:
{
  "hits": {
    "hits": [
      {
        "_source": { "description": "Powerful laptop with M3 Max chip..." },
        "highlight": {
          "description": [
            "<mark>Powerful</mark> <mark>laptop</mark> with <mark>M3</mark> Max <mark>chip</mark>, 36GB RAM"
          ]
        }
      }
    ]
  }
}
// Frontend can render the <mark> tags to highlight matching text 🎨
```

---

## 💡 9. Suggesters — "Did You Mean?" & Autocomplete

### 9.1 — Term Suggester (Spell Correction)

```json
// User typed: "iphne" → suggest "iphone"
GET /products/_search
{
  "suggest": {
    "spell_check": {
      "text": "iphne samung",                 // User's typo-filled input
      "term": {
        "field": "name",
        "suggest_mode": "popular",            // Suggest more popular terms
        "min_word_length": 3
      }
    }
  }
}

// Response:
{
  "suggest": {
    "spell_check": [
      {
        "text": "iphne",
        "options": [
          { "text": "iphone", "score": 0.8, "freq": 156 }     // ✅ Correction!
        ]
      },
      {
        "text": "samung",
        "options": [
          { "text": "samsung", "score": 0.83, "freq": 142 }   // ✅ Correction!
        ]
      }
    ]
  }
}
```

### 9.2 — Phrase Suggester (Whole Phrase Correction)

```json
GET /products/_search
{
  "suggest": {
    "phrase_suggestion": {
      "text": "macbok pro laptp",
      "phrase": {
        "field": "name",
        "size": 3,                          // Return top 3 suggestions
        "gram_size": 2,                     // Bigram context
        "highlight": {
          "pre_tag": "<b>",
          "post_tag": "</b>"
        }
      }
    }
  }
}
// Suggests: "macbook pro laptop" ✅
```

### 9.3 — Completion Suggester (Autocomplete — Blazing Fast ⚡)

The completion suggester uses an in-memory data structure (FST) for prefix-based suggestions. It's designed for search-as-you-type.

```json
// Step 1: Setup mapping with "completion" type
PUT /products
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "name_suggest": {
        "type": "completion",              // Special type for autocomplete
        "analyzer": "simple"
      }
    }
  }
}

// Step 2: Index with suggestion input
PUT /products/_doc/1
{
  "name": "MacBook Pro 16-inch",
  "name_suggest": {
    "input": ["MacBook Pro", "MacBook", "Mac", "Apple MacBook Pro"],
    "weight": 10                            // Priority weight
  }
}

// Step 3: Query suggestions
GET /products/_search
{
  "suggest": {
    "product_suggest": {
      "prefix": "mac",                      // User typed "mac"
      "completion": {
        "field": "name_suggest",
        "size": 5,                          // Top 5 suggestions
        "skip_duplicates": true,
        "fuzzy": {
          "fuzziness": "AUTO"               // Also handle typos!
        }
      }
    }
  }
}
// Returns: "MacBook Pro", "MacBook Air", "Mac Mini" ... instantly! ⚡
```

---

## 🔢 10. Sorting & Pagination

### 10.1 — Sorting

```json
// Sort by price ascending, then by rating descending
GET /products/_search
{
  "query": { "match": { "name": "laptop" } },
  "sort": [
    { "price": { "order": "asc" } },
    { "ratings.average": { "order": "desc" } },
    "_score"                                    // Then by relevance
  ]
}

// Sort by geo distance
GET /restaurants/_search
{
  "sort": [
    {
      "_geo_distance": {
        "location": { "lat": 40.71, "lon": -74.00 },
        "order": "asc",
        "unit": "km"
      }
    }
  ]
}
```

### 10.2 — Pagination

```json
// Basic pagination (from + size)
GET /products/_search
{
  "from": 0,        // Skip 0 documents (page 1)
  "size": 10,       // Return 10 results
  "query": { "match_all": {} }
}
// Page 2: from=10, size=10
// Page 3: from=20, size=10

// ⚠️ WARNING: from + size is limited to 10,000!
// from=9990, size=20 → ❌ Error!
// This is by design — deep pagination is expensive
```

### 10.3 — Deep Pagination Solutions

```json
// Solution 1: search_after (Recommended for real-time scrolling)
// First request:
GET /products/_search
{
  "size": 10,
  "sort": [
    { "price": "asc" },
    { "_id": "asc" }              // Tiebreaker (must be unique)
  ]
}
// Response includes last doc's sort values: [299.99, "prod_042"]

// Next page — use search_after with last doc's sort values:
GET /products/_search
{
  "size": 10,
  "sort": [
    { "price": "asc" },
    { "_id": "asc" }
  ],
  "search_after": [299.99, "prod_042"]     // Continue after this point
}

// Solution 2: Scroll API (For batch processing ALL results)
// Not for real-time — creates a point-in-time snapshot
POST /products/_search?scroll=5m
{
  "size": 1000,
  "query": { "match_all": {} }
}
// Returns a scroll_id — use it to get next batch
POST /_search/scroll
{
  "scroll": "5m",
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAAAA..."
}
```

---

## 🎨 11. Source Filtering & Field Selection

```json
// Return only specific fields (like SQL SELECT name, price)
GET /products/_search
{
  "_source": ["name", "price", "brand"],       // Only these fields
  "query": { "match": { "name": "laptop" } }
}

// Exclude fields
GET /products/_search
{
  "_source": {
    "includes": ["name", "price", "specs.*"],  // Include name, price, all specs
    "excludes": ["description"]                 // Exclude description
  },
  "query": { "match_all": {} }
}

// Disable _source entirely (only need _id and _score)
GET /products/_search
{
  "_source": false,
  "query": { "match": { "name": "laptop" } }
}
```

---

## 🏗️ 12. Putting It All Together — Real-World Search Page

Here's a complete production-style query for an e-commerce search page:

```json
// User searched: "gaming laptop"
// Filters: brand=ASUS, price $500-$2000, in stock, rating >= 4.0
// Show facets: brand counts, price distribution, rating distribution

GET /products/_search
{
  "size": 20,
  "from": 0,
  "_source": ["name", "brand", "price", "ratings", "image_url", "in_stock"],

  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "gaming laptop",
            "fields": ["name^3", "description", "tags^2"],
            "type": "best_fields",
            "fuzziness": "AUTO"
          }
        }
      ],
      "filter": [
        { "term": { "in_stock": true } },
        { "range": { "price": { "gte": 500, "lte": 2000 } } },
        { "range": { "ratings.average": { "gte": 4.0 } } }
      ]
    }
  },

  "highlight": {
    "pre_tags": ["<b>"],
    "post_tags": ["</b>"],
    "fields": {
      "name": {},
      "description": { "fragment_size": 120, "number_of_fragments": 2 }
    }
  },

  "sort": [
    "_score",
    { "ratings.average": { "order": "desc" } }
  ],

  "aggs": {
    "brand_facets": {
      "terms": { "field": "brand", "size": 20 }
    },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "key": "Under $500", "to": 500 },
          { "key": "$500-$1000", "from": 500, "to": 1000 },
          { "key": "$1000-$1500", "from": 1000, "to": 1500 },
          { "key": "$1500-$2000", "from": 1500, "to": 2000 },
          { "key": "Over $2000", "from": 2000 }
        ]
      }
    },
    "avg_price": {
      "avg": { "field": "price" }
    },
    "rating_histogram": {
      "histogram": {
        "field": "ratings.average",
        "interval": 0.5,
        "min_doc_count": 0
      }
    }
  },

  "suggest": {
    "spell_check": {
      "text": "gaming laptop",
      "term": {
        "field": "name",
        "suggest_mode": "popular"
      }
    }
  }
}
```

This single query gives you:
- ✅ **Search results** with relevance scoring & fuzzy matching
- ✅ **Highlighted** matching terms
- ✅ **Filtered** by stock, price, rating
- ✅ **Facets** for brand sidebar, price ranges
- ✅ **Spell-check** suggestions
- ✅ All in **one request**, under **50ms** ⚡

---

## 🔑 Key Takeaways

```
✅ Query context scores relevance; Filter context is binary yes/no (faster, cached)
✅ match = full-text search; term = exact match (use on keyword fields!)
✅ Bool query (must/should/must_not/filter) is the backbone of 90% of queries
✅ Put exact matches, ranges, and booleans in filter for max performance
✅ Fuzzy matching handles typos — use fuzziness: "AUTO" in production
✅ Aggregations = SQL GROUP BY on steroids (terms, date_histogram, range, nested)
✅ Always use "size": 0 when you only need aggregations (skip search hits)
✅ Highlighting shows users WHY a result matched — essential for search UX
✅ Completion suggester = fastest autocomplete (in-memory FST)
✅ For pagination beyond 10K, use search_after (not from+size)
✅ _source filtering reduces network payload — only return fields you need
```

---

## ❓ Self-Check Questions

1. What's the difference between `query` context and `filter` context? When should you use each?
2. Why would `term: { "name": "MacBook Pro" }` fail on a `text` field?
3. How does `multi_match` with `best_fields` differ from `cross_fields`?
4. Write a Bool query: Find ERROR logs from "auth-service" in the last 7 days, exclude health checks
5. What fuzziness value does `AUTO` set for a 7-character search term?
6. What's the max offset for `from + size` pagination? What's the alternative?
7. How would you build a "brands sidebar" with product counts for an e-commerce filter panel?
8. What's the difference between `terms` aggregation and `range` aggregation?
9. How does the completion suggester differ from the term suggester?
10. Why should you always use `"size": 0` when you only want aggregation results?

---

## 🎯 Hands-On Lab

```
Lab 1: Create a "movies" index with 20+ movies (use _bulk API)
  → Fields: title (text), genre (keyword), year (integer),
    rating (float), director (keyword), plot (text)

Lab 2: Write queries:
  a) Find movies about "space adventure" (full-text, fuzzy)
  b) Find all "Sci-Fi" movies from 2010-2024 with rating > 8.0
  c) Find movies NOT directed by "Christopher Nolan"
  d) Search "intrstller" (typo) and still find "Interstellar"

Lab 3: Write aggregations:
  a) Top 5 genres by movie count
  b) Average rating per genre
  c) Movies per decade (date_histogram with 10y interval)
  d) Rating distribution (histogram, interval 1.0)

Lab 4: Build a complete search page query combining:
  → Full-text search + filters + highlighting + facets + sorting

Lab 5: Implement autocomplete for movie titles using
  completion suggester
```

---

> **Next Chapter:** [3F.3 — ELK Stack: Elasticsearch + Logstash + Kibana](./03-ELK-Stack.md) — Build a complete observability pipeline: ingest logs, visualize data, and manage index lifecycles! 📊
