# 2E.3 — PostgreSQL Advanced Features 🟡⭐🔥

> **"PostgreSQL isn't just a relational database. It's a relational database that ate MongoDB, Elasticsearch, and PostGIS — and got stronger."**

> **Level:** 🟡 Intermediate | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 2E.1-2E.2 (Architecture & Installation)

---

## 🎯 What You'll Master

```
✅ JSONB — store, query, and index JSON like a NoSQL pro
✅ Arrays — native array columns (yes, in a relational DB!)
✅ hstore — lightweight key-value store in a column
✅ Full-Text Search — built-in search engine (no Elasticsearch needed!)
✅ Range Types — date ranges, number ranges with overlap detection
✅ PostGIS — geospatial queries (GPS, distance, polygons)
✅ Table Inheritance — OOP in your database
✅ ENUM Types — type-safe status fields
✅ Generated Columns — auto-computed columns
✅ LISTEN/NOTIFY — real-time pub/sub built into PostgreSQL
✅ Composite Types & Domains — custom data modeling
```

---

## 🏗️ Sample Database for This Chapter

```sql
-- E-commerce platform with modern PostgreSQL features
CREATE TABLE products (
    id         INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name       TEXT NOT NULL,
    price      NUMERIC(10,2) NOT NULL,
    tags       TEXT[],                              -- Array!
    attributes JSONB DEFAULT '{}',                  -- JSONB!
    metadata   hstore,                              -- Key-value!
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE events (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_type TEXT NOT NULL,
    payload    JSONB NOT NULL,
    occurred_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 🔥 1. JSONB — PostgreSQL's MongoDB Killer

### JSON vs JSONB — Know the Difference

```
┌──────────────────────────────────────────────────────────────────┐
│                    JSON vs JSONB                                  │
├──────────────────┬───────────────────┬───────────────────────────┤
│  Feature         │  JSON             │  JSONB                    │
├──────────────────┼───────────────────┼───────────────────────────┤
│  Storage         │  Plain text       │  Binary (decomposed)      │
│  Parsing         │  Every read       │  Once on write            │
│  Duplicate keys  │  Preserved        │  Last value wins          │
│  Key ordering    │  Preserved        │  Not guaranteed           │
│  Indexing        │  ❌ No            │  ✅ GIN indexes           │
│  Operators       │  Limited          │  Full set (@>, ?, #>)     │
│  Speed (write)   │  Faster           │  Slightly slower          │
│  Speed (read)    │  Slower           │  MUCH faster ⭐           │
│                  │                   │                           │
│  USE THIS?       │  Almost never     │  ✅ ALWAYS use JSONB      │
└──────────────────┴───────────────────┴───────────────────────────┘
```

> 💡 **Rule:** Always use `JSONB` unless you have a specific reason to preserve key order or duplicates (you almost never do).

### Inserting JSONB Data

```sql
INSERT INTO products (name, price, attributes) VALUES
('iPhone 15 Pro', 999.99, '{
    "brand": "Apple",
    "color": "Natural Titanium",
    "storage_gb": 256,
    "features": ["Dynamic Island", "USB-C", "A17 Pro"],
    "specs": {
        "weight_grams": 187,
        "screen_inches": 6.1,
        "battery_mah": 3274
    },
    "in_stock": true
}'),
('Galaxy S24 Ultra', 1199.99, '{
    "brand": "Samsung",
    "color": "Titanium Gray",
    "storage_gb": 512,
    "features": ["S Pen", "AI Features", "200MP Camera"],
    "specs": {
        "weight_grams": 232,
        "screen_inches": 6.8,
        "battery_mah": 5000
    },
    "in_stock": true
}'),
('Pixel 8', 699.99, '{
    "brand": "Google",
    "color": "Obsidian",
    "storage_gb": 128,
    "features": ["Tensor G3", "Magic Eraser", "Call Screen"],
    "specs": {
        "weight_grams": 187,
        "screen_inches": 6.2,
        "battery_mah": 4575
    },
    "in_stock": false
}');
```

### Querying JSONB — The Complete Operator Reference

```sql
-- ═══════════════════════════════════════════════════
--  EXTRACTING VALUES
-- ═══════════════════════════════════════════════════

-- -> returns JSON (keeps quotes for strings)
SELECT attributes->'brand' FROM products;
-- Result: "Apple"  (with quotes — it's JSON)

-- ->> returns TEXT (no quotes — plain text)
SELECT attributes->>'brand' FROM products;
-- Result: Apple  (without quotes — it's text)

-- #> navigate nested path (returns JSON)
SELECT attributes#>'{specs,weight_grams}' FROM products;
-- Result: 187

-- #>> navigate nested path (returns TEXT)
SELECT attributes#>>'{specs,weight_grams}' FROM products;
-- Result: '187'  (text)

-- Access array elements (0-indexed)
SELECT attributes->'features'->0 FROM products;
-- Result: "Dynamic Island"

SELECT attributes->'features'->>0 FROM products;
-- Result: Dynamic Island
```

```sql
-- ═══════════════════════════════════════════════════
--  FILTERING / SEARCHING
-- ═══════════════════════════════════════════════════

-- @> contains (left contains right)
SELECT name FROM products 
WHERE attributes @> '{"brand": "Apple"}';
-- "Does attributes contain this JSON subset?"

-- <@ is contained by (right contains left)
SELECT name FROM products 
WHERE '{"brand": "Apple"}' <@ attributes;

-- ? key exists
SELECT name FROM products 
WHERE attributes ? 'brand';
-- "Does the key 'brand' exist?"

-- ?| any key exists
SELECT name FROM products 
WHERE attributes ?| array['brand', 'color'];
-- "Does ANY of these keys exist?"

-- ?& all keys exist
SELECT name FROM products 
WHERE attributes ?& array['brand', 'color', 'storage_gb'];
-- "Do ALL of these keys exist?"
```

```sql
-- ═══════════════════════════════════════════════════
--  REAL-WORLD QUERIES
-- ═══════════════════════════════════════════════════

-- Find all Apple products
SELECT name, price 
FROM products 
WHERE attributes->>'brand' = 'Apple';

-- Find products with 256GB or more storage
SELECT name, attributes->>'storage_gb' AS storage
FROM products 
WHERE (attributes->>'storage_gb')::int >= 256;

-- Find products with a specific feature
SELECT name 
FROM products 
WHERE attributes->'features' @> '"USB-C"';  -- Element in array

-- Find products with screen > 6.5 inches
SELECT name, attributes#>>'{specs,screen_inches}' AS screen
FROM products
WHERE (attributes#>>'{specs,screen_inches}')::float > 6.5;

-- Find in-stock products
SELECT name FROM products 
WHERE attributes @> '{"in_stock": true}';

-- Products with ANY of these features
SELECT name FROM products
WHERE attributes->'features' ?| array['S Pen', 'Dynamic Island'];
```

### Modifying JSONB

```sql
-- ═══════════════════════════════════════════════════
--  UPDATING JSONB
-- ═══════════════════════════════════════════════════

-- Set/update a key (|| merges)
UPDATE products 
SET attributes = attributes || '{"on_sale": true, "discount_pct": 10}'
WHERE name = 'Pixel 8';

-- Set nested value
UPDATE products 
SET attributes = jsonb_set(attributes, '{specs,5g}', 'true')
WHERE attributes->>'brand' = 'Apple';

-- Remove a key
UPDATE products 
SET attributes = attributes - 'on_sale'
WHERE name = 'Pixel 8';

-- Remove nested key
UPDATE products 
SET attributes = attributes #- '{specs,5g}';

-- Remove array element by index
UPDATE products 
SET attributes = attributes #- '{features,0}';  -- Remove first feature

-- Append to array inside JSONB
UPDATE products 
SET attributes = jsonb_set(
    attributes, 
    '{features}', 
    (attributes->'features') || '"Wireless Charging"'
)
WHERE attributes->>'brand' = 'Apple';
```

### JSONB Aggregation Functions

```sql
-- Build JSON from rows
SELECT jsonb_agg(
    jsonb_build_object('name', name, 'price', price)
) AS products_json
FROM products;

-- Build a single JSON object from key-value pairs
SELECT jsonb_object_agg(name, price) FROM products;
-- {"iPhone 15 Pro": 999.99, "Galaxy S24 Ultra": 1199.99, ...}

-- Expand JSON array to rows
SELECT p.name, feature
FROM products p, 
     jsonb_array_elements_text(p.attributes->'features') AS feature
WHERE p.attributes->>'brand' = 'Apple';
-- Result:
-- iPhone 15 Pro | Dynamic Island
-- iPhone 15 Pro | USB-C
-- iPhone 15 Pro | A17 Pro
```

### Indexing JSONB — Make It FAST 🚀

```sql
-- GIN index on entire JSONB column (most common)
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);

-- Now these queries use the index:
SELECT * FROM products WHERE attributes @> '{"brand": "Apple"}';  ✅
SELECT * FROM products WHERE attributes ? 'brand';                ✅
SELECT * FROM products WHERE attributes ?| array['brand','color']; ✅

-- GIN index with jsonb_path_ops (smaller, faster for @> only)
CREATE INDEX idx_products_attrs_path ON products 
  USING GIN (attributes jsonb_path_ops);
-- Only supports @> operator, but 2-3x smaller and faster

-- B-tree index on specific JSONB path (for equality/range queries)
CREATE INDEX idx_products_brand ON products ((attributes->>'brand'));

-- Now this uses a regular B-tree index:
SELECT * FROM products WHERE attributes->>'brand' = 'Apple';  ✅

-- Expression index for numeric comparisons
CREATE INDEX idx_products_storage ON products 
  (((attributes->>'storage_gb')::int));

SELECT * FROM products 
WHERE (attributes->>'storage_gb')::int >= 256;  -- Uses index! ✅
```

> 💡 **When to use JSONB vs regular columns?**
> ```
> USE JSONB when:
>   ✅ Schema varies between rows (product attributes)
>   ✅ Semi-structured data (API responses, configs)
>   ✅ Nested/complex structures
>   ✅ You query by content (@>, ?)
> 
> USE regular columns when:
>   ✅ Schema is fixed and well-known
>   ✅ You need foreign keys, constraints
>   ✅ Frequent equality/range queries on specific fields
>   ✅ You need maximum query performance
> 
> HYBRID approach (best of both worlds):
>   → Core fields as regular columns
>   → Flexible/optional data as JSONB column
> ```

---

## 📦 2. Arrays — Lists Built Into Your Columns

PostgreSQL supports **native array columns** — no join tables needed for simple lists!

### Creating and Inserting Arrays

```sql
CREATE TABLE articles (
    id     INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title  TEXT NOT NULL,
    tags   TEXT[],                    -- Array of text
    scores INT[]                     -- Array of integers
);

-- Insert with array literal syntax
INSERT INTO articles (title, tags, scores) VALUES
('PostgreSQL Guide', ARRAY['database', 'postgresql', 'sql'], ARRAY[95, 88, 92]),
('Docker Tutorial', '{docker, devops, containers}', '{87, 91, 85}'),
('React Hooks', ARRAY['react', 'javascript', 'frontend'], ARRAY[92, 89, 94]);

-- Multi-dimensional arrays
INSERT INTO articles (title, scores) 
VALUES ('Matrix', ARRAY[[1,2],[3,4]]);  -- 2D array
```

### Querying Arrays

```sql
-- Access by index (1-based in PostgreSQL!)
SELECT title, tags[1] AS first_tag FROM articles;
-- PostgreSQL: tags[1]  (NOT tags[0] like in most languages!)

-- Array slice
SELECT tags[1:2] FROM articles;   -- First two elements

-- Check if array CONTAINS a value
SELECT title FROM articles 
WHERE 'postgresql' = ANY(tags);    -- ANY — is value IN array?

-- Check if array contains ALL of these values
SELECT title FROM articles 
WHERE tags @> ARRAY['database', 'sql'];  -- @> contains

-- Check if array is contained by
SELECT title FROM articles 
WHERE tags <@ ARRAY['database', 'postgresql', 'sql', 'nosql'];

-- Array overlap (any common elements?)
SELECT title FROM articles 
WHERE tags && ARRAY['react', 'docker'];  -- && = overlap

-- Array length
SELECT title, array_length(tags, 1) AS tag_count FROM articles;
-- 2nd arg = dimension (1 for 1D arrays)

-- Unnest — expand array to rows (extremely useful!)
SELECT title, unnest(tags) AS tag FROM articles;
-- Result:
-- PostgreSQL Guide | database
-- PostgreSQL Guide | postgresql
-- PostgreSQL Guide | sql
-- Docker Tutorial  | docker
-- Docker Tutorial  | devops
-- ...
```

### Array Functions

```sql
-- Append to array
UPDATE articles 
SET tags = array_append(tags, 'tutorial')
WHERE title = 'PostgreSQL Guide';

-- Prepend to array
UPDATE articles 
SET tags = array_prepend('beginner', tags)
WHERE title = 'Docker Tutorial';

-- Remove element
UPDATE articles 
SET tags = array_remove(tags, 'tutorial')
WHERE title = 'PostgreSQL Guide';

-- Concatenate arrays
SELECT ARRAY[1,2] || ARRAY[3,4];   -- {1,2,3,4}

-- Array position (find index)
SELECT array_position(tags, 'docker') FROM articles;

-- Replace element
UPDATE articles 
SET tags = array_replace(tags, 'docker', 'Docker')
WHERE title = 'Docker Tutorial';

-- Aggregate values INTO an array
SELECT array_agg(DISTINCT unnest) 
FROM (SELECT unnest(tags) FROM articles) t;
-- All unique tags across all articles as one array

-- Convert array to string
SELECT array_to_string(tags, ', ') FROM articles;
-- "database, postgresql, sql"

-- Convert string to array
SELECT string_to_array('one,two,three', ',');
-- {one,two,three}
```

### Indexing Arrays — GIN is Your Friend

```sql
-- GIN index for array containment queries
CREATE INDEX idx_articles_tags ON articles USING GIN (tags);

-- Now these are indexed:
SELECT * FROM articles WHERE tags @> ARRAY['sql'];        ✅  -- contains
SELECT * FROM articles WHERE tags && ARRAY['react','sql']; ✅  -- overlap
SELECT * FROM articles WHERE 'sql' = ANY(tags);            ✅  -- any
```

> 💡 **Arrays vs Join Tables — When to use which?**
> ```
> USE ARRAYS when:
>   ✅ Simple lists of values (tags, labels, scores)
>   ✅ Order matters
>   ✅ No need to query/join on individual elements frequently
>   ✅ Array is small (< 100 elements)
> 
> USE JOIN TABLES when:
>   ✅ Elements are complex (have their own attributes)
>   ✅ Need foreign key constraints
>   ✅ Need to query/aggregate on individual elements frequently
>   ✅ Many-to-many relationships
> ```

---

## 🗝️ 3. hstore — Simple Key-Value Store

`hstore` is a lighter alternative to JSONB when you just need flat key-value pairs (no nesting).

```sql
CREATE EXTENSION IF NOT EXISTS hstore;

-- Create table with hstore column
CREATE TABLE server_configs (
    id       INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    hostname TEXT NOT NULL,
    settings hstore
);

-- Insert data
INSERT INTO server_configs (hostname, settings) VALUES
('web-01', 'cpu_cores => 4, ram_gb => 16, os => Ubuntu, env => production'),
('db-01',  'cpu_cores => 8, ram_gb => 64, os => CentOS, env => production'),
('dev-01', 'cpu_cores => 2, ram_gb => 8,  os => Ubuntu, env => development');

-- Query by key
SELECT hostname, settings->'ram_gb' AS ram 
FROM server_configs;

-- Check if key exists
SELECT hostname FROM server_configs 
WHERE settings ? 'env';

-- Check key-value pair
SELECT hostname FROM server_configs 
WHERE settings @> 'env => production';

-- Get all keys
SELECT hostname, akeys(settings) FROM server_configs;

-- Get all values
SELECT hostname, avals(settings) FROM server_configs;

-- Convert hstore to JSON
SELECT hostname, hstore_to_json(settings) FROM server_configs;

-- Update/add a key
UPDATE server_configs 
SET settings = settings || 'region => us-east-1'
WHERE hostname = 'web-01';

-- Delete a key
UPDATE server_configs 
SET settings = delete(settings, 'region');
```

> 💡 **hstore vs JSONB:**
> | Feature | hstore | JSONB |
> |---------|--------|-------|
> | Nesting | ❌ Flat only | ✅ Nested |
> | Arrays | ❌ No | ✅ Yes |
> | Data types | Strings only | Numbers, booleans, nulls, arrays |
> | Performance | Slightly faster for flat data | More versatile |
> | **Verdict** | Use for simple config/metadata | **Use for everything else** ⭐ |

---

## 🔍 4. Full-Text Search — Built-In Search Engine

PostgreSQL has a **powerful full-text search engine** built right in. For many applications, you don't need Elasticsearch at all!

### The Two Key Types

```
┌─────────────────────────────────────────────────────────────────┐
│  tsvector = The "document" (processed, searchable text)         │
│  tsquery  = The "search query" (what you're looking for)        │
│                                                                  │
│  tsvector:  'quick brown fox jumps over the lazy dog'           │
│           → 'brown':2 'dog':8 'fox':3 'jump':4 'lazi':7 'quick':1│
│              (stemmed, positioned, stop words removed)           │
│                                                                  │
│  tsquery:   'fox & dog'                                         │
│           → 'fox' & 'dog'  (AND search)                         │
│                                                                  │
│  Match: tsvector @@ tsquery → true/false                        │
└─────────────────────────────────────────────────────────────────┘
```

### Setting Up Full-Text Search

```sql
CREATE TABLE blog_posts (
    id           INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title        TEXT NOT NULL,
    body         TEXT NOT NULL,
    published_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- Dedicated search vector column (for performance)
    search_vector tsvector GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(body, '')), 'B')
    ) STORED
);

-- Create GIN index on the search vector
CREATE INDEX idx_blog_search ON blog_posts USING GIN (search_vector);

-- Insert sample data
INSERT INTO blog_posts (title, body) VALUES
('PostgreSQL Full-Text Search Guide', 
 'Learn how to implement powerful text search in PostgreSQL without needing external tools like Elasticsearch. PostgreSQL supports stemming, ranking, and phrase search out of the box.'),
('Database Performance Tuning', 
 'Optimize your database queries with proper indexing strategies. Learn about B-tree, GIN, GiST, and BRIN indexes. Query plans and EXPLAIN ANALYZE are essential tools.'),
('Getting Started with Docker', 
 'Docker containers revolutionize application deployment. Learn about images, containers, volumes, and networking. Docker Compose simplifies multi-container applications.');
```

### Searching

```sql
-- Basic search (AND)
SELECT title, ts_rank(search_vector, query) AS rank
FROM blog_posts, to_tsquery('english', 'postgresql & search') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- OR search
SELECT title FROM blog_posts
WHERE search_vector @@ to_tsquery('english', 'docker | postgresql');

-- NOT search
SELECT title FROM blog_posts
WHERE search_vector @@ to_tsquery('english', 'database & !docker');

-- Phrase search (words adjacent)
SELECT title FROM blog_posts
WHERE search_vector @@ phraseto_tsquery('english', 'text search');

-- Prefix search (autocomplete!)
SELECT title FROM blog_posts
WHERE search_vector @@ to_tsquery('english', 'post:*');  
-- Matches: postgresql, posts, posting...

-- Fuzzy / plain text search (handles user input safely)
SELECT title FROM blog_posts
WHERE search_vector @@ plainto_tsquery('english', 'database indexing');
-- Automatically ANDs the words
```

### Search with Highlighting

```sql
-- Highlight matching terms in results
SELECT 
    title,
    ts_headline('english', body, 
        to_tsquery('english', 'postgresql & search'),
        'StartSel=<b>, StopSel=</b>, MaxFragments=2, MaxWords=30'
    ) AS highlighted_excerpt
FROM blog_posts
WHERE search_vector @@ to_tsquery('english', 'postgresql & search');

-- Result:
-- title: "PostgreSQL Full-Text Search Guide"
-- highlighted_excerpt: "Learn how to implement powerful text <b>search</b> in <b>PostgreSQL</b> without needing..."
```

### Ranking Results

```sql
-- ts_rank: Ranks based on frequency of matching terms
SELECT 
    title,
    ts_rank(search_vector, query) AS rank,
    ts_rank_cd(search_vector, query) AS cover_density_rank
FROM blog_posts, 
     to_tsquery('english', 'database | search') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- Weighted ranking (title matches rank higher!)
-- Remember: we used setweight 'A' for title, 'B' for body
-- Weights: A=1.0, B=0.4, C=0.2, D=0.1 (defaults)
SELECT 
    title,
    ts_rank(search_vector, query, 1) AS rank  -- 1 = normalize by document length
FROM blog_posts, 
     to_tsquery('english', 'database') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### Multiple Languages & Custom Dictionaries

```sql
-- See available text search configurations
SELECT cfgname FROM pg_ts_config;
-- english, french, german, spanish, russian, simple, ...

-- Search in different languages
SELECT to_tsvector('french', 'les chats mangent des souris');
-- 'chat':2 'mang':3 'souri':5  (stemmed French!)

-- 'simple' config: no stemming, no stop words (for exact matching)
SELECT to_tsvector('simple', 'The Quick Brown Fox');
-- 'brown':3 'fox':4 'quick':2 'the':1  (keeps "the"!)

-- Use pg_trgm for fuzzy/typo-tolerant search
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_blog_title_trgm ON blog_posts USING GIN (title gin_trgm_ops);

-- Now you can do similarity search
SELECT title, similarity(title, 'postgre') AS sim
FROM blog_posts
WHERE title % 'postgre'    -- % = similarity above threshold
ORDER BY sim DESC;

-- Or ILIKE with index support!
SELECT title FROM blog_posts 
WHERE title ILIKE '%postgre%';  -- Uses GIN trgm index! ✅
```

> 💡 **PostgreSQL FTS vs Elasticsearch:**
> | Feature | PostgreSQL FTS | Elasticsearch |
> |---------|---------------|---------------|
> | Setup complexity | Zero — built in! | Separate cluster needed |
> | Transactions | Full ACID ✅ | Eventually consistent |
> | Real-time | Instant (same DB) | Near real-time (refresh interval) |
> | Scale | Good for millions | Great for billions |
> | Fuzzy matching | pg_trgm extension | Built-in |
> | Faceted search | Manual with GROUP BY | Built-in aggregations |
> | **Verdict** | **Perfect for < 10M docs** | Needed for massive scale |

---

## 📏 5. Range Types — Elegant Interval Queries

PostgreSQL has built-in **range types** for representing intervals. No more `start_date`/`end_date` column pairs!

```sql
-- Built-in range types:
-- int4range     → Range of integers
-- int8range     → Range of bigints
-- numrange      → Range of numeric
-- tsrange       → Range of timestamp (without tz)
-- tstzrange     → Range of timestamp with timezone
-- daterange     → Range of dates

-- ═══════════════════════════════════════════════════
--  EXAMPLE: Hotel Room Bookings
-- ═══════════════════════════════════════════════════

CREATE TABLE room_bookings (
    id         INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id    INT NOT NULL,
    guest_name TEXT NOT NULL,
    stay       daterange NOT NULL,    -- ← Range type!
    
    -- EXCLUSION CONSTRAINT — prevent overlapping bookings! ⭐
    EXCLUDE USING GIST (
        room_id WITH =,         -- Same room
        stay WITH &&             -- Overlapping dates
    )
);

-- Enable required extension for exclusion constraints with non-GiST types
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Insert bookings
INSERT INTO room_bookings (room_id, guest_name, stay) VALUES
(101, 'Alice', '[2024-06-01, 2024-06-05)'),   -- June 1-4 (upper exclusive)
(101, 'Bob',   '[2024-06-05, 2024-06-08)'),   -- June 5-7 (no overlap ✅)
(102, 'Carol', '[2024-06-01, 2024-06-10)');    -- Room 102, different room ✅

-- This FAILS — overlapping dates for room 101!
INSERT INTO room_bookings (room_id, guest_name, stay) VALUES
(101, 'Dave', '[2024-06-03, 2024-06-06)');     -- ERROR: conflicting ❌
```

### Range Operators

```sql
-- Contains element
SELECT * FROM room_bookings WHERE stay @> '2024-06-03'::date;
-- "Which bookings include June 3?"

-- Range contains range
SELECT * FROM room_bookings WHERE stay @> '[2024-06-02, 2024-06-04)'::daterange;

-- Overlaps
SELECT * FROM room_bookings WHERE stay && '[2024-06-04, 2024-06-07)'::daterange;
-- "Which bookings overlap with June 4-6?"

-- Adjacent (ranges touch but don't overlap)
SELECT * FROM room_bookings WHERE stay -|- '[2024-06-05, 2024-06-08)'::daterange;

-- Upper and lower bounds
SELECT guest_name, lower(stay) AS check_in, upper(stay) AS check_out
FROM room_bookings;

-- Is empty?
SELECT isempty('[5,5)'::int4range);  -- true (empty range)

-- Intersection
SELECT '[1,10)'::int4range * '[5,15)'::int4range;  -- [5,10)

-- Union
SELECT '[1,5)'::int4range + '[3,10)'::int4range;   -- [1,10)
```

> 💡 **Range Notation:**
> - `[1, 10]` — inclusive on both sides (1 ≤ x ≤ 10)
> - `[1, 10)` — inclusive lower, exclusive upper (1 ≤ x < 10) ← **Most common for dates!**
> - `(1, 10]` — exclusive lower, inclusive upper (1 < x ≤ 10)
> - `(1, 10)` — exclusive on both sides (1 < x < 10)

---

## 🌍 6. PostGIS — Geospatial Superpowers

**PostGIS** turns PostgreSQL into a full-fledged Geographic Information System. Used by OpenStreetMap, Uber, Airbnb, and Lyft.

```sql
-- Install PostGIS
CREATE EXTENSION postgis;

-- Create a table with geographic data
CREATE TABLE restaurants (
    id       INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name     TEXT NOT NULL,
    cuisine  TEXT,
    location GEOGRAPHY(POINT, 4326)  -- GPS coordinates (lat/lng)
);

-- Insert restaurants (longitude, latitude — note: longitude FIRST!)
INSERT INTO restaurants (name, cuisine, location) VALUES
('Pizza Palace', 'Italian', ST_Point(77.2090, 28.6139)::geography),    -- Delhi
('Sushi Supreme', 'Japanese', ST_Point(77.2167, 28.6353)::geography),  -- Delhi
('Burger Barn', 'American', ST_Point(72.8777, 19.0760)::geography),    -- Mumbai
('Curry House', 'Indian', ST_Point(77.5946, 12.9716)::geography);      -- Bangalore

-- Find restaurants within 5km of a point (Connaught Place, Delhi)
SELECT name, cuisine,
    ST_Distance(location, ST_Point(77.2195, 28.6328)::geography) AS distance_meters
FROM restaurants
WHERE ST_DWithin(location, ST_Point(77.2195, 28.6328)::geography, 5000)
ORDER BY distance_meters;

-- Find the nearest 3 restaurants
SELECT name, cuisine,
    round(ST_Distance(location, ST_Point(77.2195, 28.6328)::geography)::numeric) AS meters
FROM restaurants
ORDER BY location <-> ST_Point(77.2195, 28.6328)::geography
LIMIT 3;

-- Create spatial index (essential for performance!)
CREATE INDEX idx_restaurants_location ON restaurants USING GIST (location);
```

---

## 🧬 7. Table Inheritance — OOP in Your Database

PostgreSQL supports **table inheritance** — child tables inherit columns from parent tables.

```sql
-- Parent table
CREATE TABLE vehicles (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    make        TEXT NOT NULL,
    model       TEXT NOT NULL,
    year        INT NOT NULL,
    price       NUMERIC(10,2)
);

-- Child tables inherit ALL columns from parent
CREATE TABLE cars (
    doors       INT DEFAULT 4,
    trunk_size  TEXT
) INHERITS (vehicles);

CREATE TABLE trucks (
    payload_kg  NUMERIC(10,2),
    bed_length  TEXT
) INHERITS (vehicles);

-- Insert into children
INSERT INTO cars (make, model, year, price, doors) 
VALUES ('Toyota', 'Camry', 2024, 28000, 4);

INSERT INTO trucks (make, model, year, price, payload_kg) 
VALUES ('Ford', 'F-150', 2024, 35000, 1450);

-- Query PARENT → gets ALL rows (parent + children)
SELECT * FROM vehicles;        -- Returns cars AND trucks!

-- Query ONLY parent (exclude children)
SELECT * FROM ONLY vehicles;   -- Only rows directly in vehicles table

-- Query specific child
SELECT * FROM cars;            -- Only cars
SELECT * FROM trucks;          -- Only trucks
```

> ⚠️ **Table Inheritance Limitations:**
> - Indexes on parent don't cover children
> - UNIQUE constraints don't span children
> - Foreign keys don't work across inheritance hierarchy
> - **For partitioning, use Declarative Partitioning instead** (PostgreSQL 10+)

---

## 🏷️ 8. ENUM Types — Type-Safe Status Fields

```sql
-- Create an ENUM type
CREATE TYPE order_status AS ENUM (
    'pending', 'confirmed', 'shipped', 'delivered', 'cancelled'
);

-- Use it in a table
CREATE TABLE orders (
    id        INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product   TEXT NOT NULL,
    status    order_status NOT NULL DEFAULT 'pending',
    amount    NUMERIC(10,2)
);

-- Insert (only valid values allowed!)
INSERT INTO orders (product, status, amount) VALUES
('Laptop', 'confirmed', 999.99);

-- This FAILS:
INSERT INTO orders (product, status, amount) VALUES
('Phone', 'invalid_status', 499.99);
-- ERROR: invalid input value for enum order_status: "invalid_status"

-- ENUMs have a natural ordering (by definition order)
SELECT * FROM orders ORDER BY status;  
-- pending → confirmed → shipped → delivered → cancelled

-- Add new value (PostgreSQL 9.1+)
ALTER TYPE order_status ADD VALUE 'refunded' AFTER 'cancelled';

-- List all values
SELECT enum_range(NULL::order_status);
-- {pending,confirmed,shipped,delivered,cancelled,refunded}
```

> 💡 **ENUM vs CHECK constraint vs Lookup table:**
> | Approach | Pros | Cons |
> |----------|------|------|
> | `ENUM` | Type-safe, compact storage (4 bytes) | Can't remove values, hard to rename |
> | `CHECK` | Simple, flexible | No autocomplete, no type safety |
> | Lookup table | Fully flexible, FK integrity | Extra JOIN needed |
> | **Verdict** | ENUM for stable sets (status, priority) | Lookup table for dynamic sets |

---

## ⚡ 9. Generated Columns — Auto-Computed Values

```sql
CREATE TABLE products_v2 (
    id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        TEXT NOT NULL,
    price       NUMERIC(10,2) NOT NULL,
    tax_rate    NUMERIC(4,2) DEFAULT 0.18,
    
    -- Generated (computed) columns
    tax_amount  NUMERIC(10,2) GENERATED ALWAYS AS (price * tax_rate) STORED,
    total_price NUMERIC(10,2) GENERATED ALWAYS AS (price * (1 + tax_rate)) STORED,
    name_lower  TEXT GENERATED ALWAYS AS (lower(name)) STORED
);

INSERT INTO products_v2 (name, price) VALUES ('MacBook Pro', 1999.99);

SELECT name, price, tax_amount, total_price FROM products_v2;
-- MacBook Pro | 1999.99 | 360.00 | 2359.99
```

> 💡 **`STORED` vs `VIRTUAL`:** PostgreSQL currently only supports `STORED` generated columns (computed on write, stored on disk). Virtual columns (computed on read) are not yet supported.

---

## 📡 10. LISTEN/NOTIFY — Real-Time Pub/Sub

PostgreSQL has **built-in pub/sub messaging** — no Redis or RabbitMQ needed for simple event notifications!

```sql
-- Terminal 1 (Listener):
LISTEN order_events;

-- Terminal 2 (Notifier):
NOTIFY order_events, '{"order_id": 42, "status": "shipped"}';

-- Terminal 1 receives:
-- Asynchronous notification "order_events" with payload 
-- "{"order_id": 42, "status": "shipped"}" received from server process with PID 1234.
```

### Trigger-Based Notifications

```sql
-- Automatically notify when rows change
CREATE OR REPLACE FUNCTION notify_order_change()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('order_changes', json_build_object(
        'action', TG_OP,
        'order_id', NEW.id,
        'status', NEW.status
    )::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_order_notify
AFTER INSERT OR UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION notify_order_change();

-- Now any INSERT or UPDATE on orders automatically sends a notification!
-- Your application (Node.js, Python, etc.) listens and reacts in real-time.
```

> 💡 **LISTEN/NOTIFY Use Cases:**
> - Cache invalidation (notify app when data changes)
> - Real-time dashboards
> - Task queues (simple job processing)
> - Microservice event communication

---

## 🧱 11. Composite Types & Domains

### Composite Types — Custom Row Types

```sql
-- Create a custom type (like a struct)
CREATE TYPE address AS (
    street  TEXT,
    city    TEXT,
    state   TEXT,
    zip     TEXT,
    country TEXT
);

CREATE TABLE customers (
    id       INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name     TEXT NOT NULL,
    home     address,      -- Composite type column
    work     address       -- Another one!
);

INSERT INTO customers (name, home, work) VALUES
('Ritesh', 
 ROW('123 Main St', 'Delhi', 'DL', '110001', 'India'),
 ROW('456 Tech Park', 'Gurgaon', 'HR', '122001', 'India')
);

-- Access composite fields
SELECT name, (home).city AS home_city, (work).city AS work_city 
FROM customers;

-- Filter on composite fields
SELECT * FROM customers WHERE (home).country = 'India';
```

### Domains — Constrained Types

```sql
-- Create a domain (type with constraints)
CREATE DOMAIN email AS TEXT
CHECK (VALUE ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$');

CREATE DOMAIN positive_amount AS NUMERIC(10,2)
CHECK (VALUE > 0);

CREATE DOMAIN indian_phone AS TEXT
CHECK (VALUE ~ '^\+91[0-9]{10}$');

-- Use domains
CREATE TABLE contacts (
    id    INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name  TEXT NOT NULL,
    email email NOT NULL,                -- Domain type!
    phone indian_phone,                  -- Domain type!
    balance positive_amount              -- Domain type!
);

INSERT INTO contacts (name, email, phone, balance) VALUES
('Ritesh', 'ritesh@example.com', '+919876543210', 1000.00);  -- ✅

INSERT INTO contacts (name, email) VALUES
('Bad', 'not-an-email');  -- ERROR: value violates check constraint ❌
```

---

## 📝 Chapter Summary — Key Takeaways

```
┌────────────────────────────────────────────────────────────────────┐
│  🧠 REMEMBER THESE:                                               │
│                                                                     │
│  1. JSONB → Always use JSONB (not JSON). Index with GIN.           │
│     Operators: -> ->> @> ? #>                                      │
│                                                                     │
│  2. Arrays → Native list columns. Index with GIN.                  │
│     Use for tags, labels, scores. Use join tables for complex data.│
│                                                                     │
│  3. Full-Text Search → tsvector + tsquery + GIN index              │
│     Sufficient for most apps. Use pg_trgm for fuzzy/typo search.   │
│                                                                     │
│  4. Range Types → daterange, int4range, tstzrange                  │
│     Exclusion constraints prevent overlapping ranges!              │
│                                                                     │
│  5. PostGIS → Geospatial queries on GPS data                       │
│     ST_DWithin, ST_Distance, GIST index                            │
│                                                                     │
│  6. ENUMs → Type-safe for stable value sets (status, priority)     │
│                                                                     │
│  7. LISTEN/NOTIFY → Built-in pub/sub for real-time events          │
│                                                                     │
│  8. Generated Columns → Auto-computed STORED columns               │
│                                                                     │
│  9. Domains → Custom types with built-in validation                │
│                                                                     │
│  PostgreSQL isn't just SQL — it's SQL + NoSQL + Search + GIS       │
│  in one battle-tested package.                                      │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

> **Next Chapter:** [2E.4 — PL/pgSQL & Extensions](./04-PG-PLPGSQL.md) 🟡
> *Write functions, triggers, custom logic — and explore the extension ecosystem that makes PostgreSQL unstoppable.*

---

[← Back to Index](../INDEX.md)
