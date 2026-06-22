# 3D.3 — Cassandra Data Modeling (Query-First Design) 🟡⭐🔥

> **"In relational databases, you model your data first and write queries later. In Cassandra, you design your queries first and model your data around them. Get this backwards, and Cassandra will make your life miserable."**

---

## 📌 What You'll Master in This Chapter

- **Query-First Design** — the fundamental paradigm shift from relational modeling
- **The Cassandra Data Modeling Workflow** — a step-by-step methodology
- **Partition Key Selection** — the most impactful design decision you'll make
- **Partition Sizing** — how to avoid monster partitions that kill performance
- **Denormalization Patterns** — duplicating data on purpose (and why it's OK)
- **One Table Per Query** — the "table per query" pattern
- **Time-Series Modeling** — the most common Cassandra use case
- **Real-World Data Models** — messaging, e-commerce, IoT, and more
- **Common Anti-Patterns** — mistakes that will destroy your cluster

---

## 🧠 The Mindset Shift — Relational vs Cassandra

Before we start, you **must** rewire your brain. Everything you learned in relational modeling needs to be reconsidered:

```
┌─────────────────────────────────────────────────────────────────┐
│          RELATIONAL THINKING         CASSANDRA THINKING          │
│          ─────────────────           ──────────────────          │
│                                                                   │
│  1. What ENTITIES do I have?    →  What QUERIES do I need?      │
│  2. Normalize to 3NF            →  Denormalize aggressively     │
│  3. Use JOINs for relationships →  Duplicate data in tables     │
│  4. One table per entity        →  One table per query pattern  │
│  5. Schema first, queries later →  Queries first, schema later  │
│  6. Minimize data duplication   →  Embrace data duplication     │
│  7. Disk space is expensive     →  Disk is cheap, latency isn't │
│  8. Reads are fast, writes vary →  Writes are fast, reads vary  │
│  9. Ad-hoc queries are fine     →  Plan every query upfront     │
│ 10. Indexes solve query needs   →  Tables solve query needs     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

> ⭐ **The Core Principle:** In Cassandra, you design your tables to answer **specific queries**. If you have 5 different query patterns, you might need 5 different tables — even if they contain overlapping data. **Disk space is cheap. User latency is not.**

---

## 🔥 1. The Cassandra Data Modeling Workflow

Follow this methodology every time. It works for every use case.

```
┌─────────────────────────────────────────────────────────────────┐
│              CASSANDRA DATA MODELING WORKFLOW                     │
│                                                                   │
│  ┌─────────────────────────────────────┐                        │
│  │ STEP 1: Understand the Application  │                        │
│  │ → What does the app do?             │                        │
│  │ → What entities exist?              │                        │
│  │ → What are the relationships?       │                        │
│  └──────────────┬──────────────────────┘                        │
│                 ▼                                                 │
│  ┌─────────────────────────────────────┐                        │
│  │ STEP 2: Define the Query Patterns   │  ← THE MOST IMPORTANT │
│  │ → List EVERY query the app needs    │     STEP!              │
│  │ → Include filters, sorts, limits    │                        │
│  │ → Include access frequency          │                        │
│  └──────────────┬──────────────────────┘                        │
│                 ▼                                                 │
│  ┌─────────────────────────────────────┐                        │
│  │ STEP 3: Design Tables for Queries   │                        │
│  │ → One table per query pattern       │                        │
│  │ → Choose partition key (filter)     │                        │
│  │ → Choose clustering key (sort)      │                        │
│  └──────────────┬──────────────────────┘                        │
│                 ▼                                                 │
│  ┌─────────────────────────────────────┐                        │
│  │ STEP 4: Validate Partition Sizes    │                        │
│  │ → Calculate rows per partition      │                        │
│  │ → Target: < 100MB per partition     │                        │
│  │ → Adjust keys if too large          │                        │
│  └──────────────┬──────────────────────┘                        │
│                 ▼                                                 │
│  ┌─────────────────────────────────────┐                        │
│  │ STEP 5: Plan Write Paths           │                        │
│  │ → How will data be written?         │                        │
│  │ → Use batches for denormalized sync │                        │
│  │ → Consider write amplification      │                        │
│  └─────────────────────────────────────┘                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔥 2. Complete Example — Messaging App (WhatsApp-style)

Let's walk through the entire workflow with a real example.

### Step 1: Understand the Application

```
Application: Messaging platform (like WhatsApp/Slack)
Entities:    Users, Conversations, Messages, Attachments
Scale:       100M users, 1B messages/day
```

### Step 2: Define Query Patterns

```
┌────┬───────────────────────────────────────────┬──────────────┐
│ Q# │ Query Description                         │ Frequency    │
├────┼───────────────────────────────────────────┼──────────────┤
│ Q1 │ Get user profile by user_id                │ Very High    │
│ Q2 │ List all conversations for a user          │ Very High    │
│ Q3 │ Get messages in a conversation (latest N)  │ Very High    │
│ Q4 │ Get a specific message by ID               │ Medium       │
│ Q5 │ Search messages in a conversation by date  │ Medium       │
│ Q6 │ Get unread message count per conversation  │ High         │
│ Q7 │ Get user's online status                   │ Very High    │
└────┴───────────────────────────────────────────┴──────────────┘
```

### Step 3: Design Tables

```sql
-- ═══════════════════════════════════════════════════════════════
-- Q1: Get user profile by user_id
-- ═══════════════════════════════════════════════════════════════
CREATE TABLE users (
    user_id UUID,
    username TEXT,
    display_name TEXT,
    avatar_url TEXT,
    status_message TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id)
);
-- Partition Key: user_id → one user per partition
-- Query: SELECT * FROM users WHERE user_id = ?;


-- ═══════════════════════════════════════════════════════════════
-- Q2: List all conversations for a user (sorted by last activity)
-- ═══════════════════════════════════════════════════════════════
CREATE TABLE conversations_by_user (
    user_id UUID,
    last_activity TIMESTAMP,
    conversation_id UUID,
    conversation_name TEXT,
    last_message_preview TEXT,
    unread_count INT,
    PRIMARY KEY (user_id, last_activity, conversation_id)
) WITH CLUSTERING ORDER BY (last_activity DESC, conversation_id ASC);
-- Partition Key: user_id → all conversations for one user = one partition
-- Clustering: last_activity DESC → most recent conversations first
-- Query: SELECT * FROM conversations_by_user
--        WHERE user_id = ? LIMIT 20;


-- ═══════════════════════════════════════════════════════════════
-- Q3: Get messages in a conversation (latest first)
-- ═══════════════════════════════════════════════════════════════
CREATE TABLE messages_by_conversation (
    conversation_id UUID,
    message_id TIMEUUID,
    sender_id UUID,
    sender_name TEXT,       -- ⚠️ Denormalized! Duplicated from users table
    body TEXT,
    message_type TEXT,      -- 'text', 'image', 'video', 'file'
    attachment_url TEXT,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
-- Partition Key: conversation_id → all messages in conversation = one partition
-- Clustering: message_id (TIMEUUID) DESC → latest messages first
-- Query: SELECT * FROM messages_by_conversation
--        WHERE conversation_id = ? LIMIT 50;


-- ═══════════════════════════════════════════════════════════════
-- Q5: Search messages by date range
-- ═══════════════════════════════════════════════════════════════
-- Same table as Q3! TIMEUUID supports range queries:
-- SELECT * FROM messages_by_conversation
-- WHERE conversation_id = ?
-- AND message_id > minTimeuuid('2024-01-01')
-- AND message_id < maxTimeuuid('2024-02-01');


-- ═══════════════════════════════════════════════════════════════
-- Q7: Get user's online status
-- ═══════════════════════════════════════════════════════════════
CREATE TABLE user_status (
    user_id UUID PRIMARY KEY,
    is_online BOOLEAN,
    last_seen TIMESTAMP
);
-- Simple key-value lookup with TTL:
-- INSERT INTO user_status (...) VALUES (...) USING TTL 300;
-- If TTL expires → user is considered offline
```

### Why We Duplicated `sender_name` in Messages

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  In SQL, you'd JOIN:                                             │
│  SELECT m.*, u.name FROM messages m                              │
│  JOIN users u ON m.sender_id = u.user_id                        │
│                                                                   │
│  In Cassandra, NO JOINS! So we DENORMALIZE:                     │
│  → Store sender_name directly in the messages table             │
│  → Yes, it's duplicated across millions of messages             │
│  → But reads are now ONE query to ONE partition = FAST           │
│                                                                   │
│  Trade-off:                                                      │
│  ✅ Read: 1 query, 1 partition, sub-millisecond                 │
│  ❌ Update: If user changes name, must update all their messages│
│     (usually handled lazily or not at all — old messages keep   │
│     the old name, which is often acceptable)                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔥 3. Partition Key Selection — The Art

Choosing the right partition key is the **single most important decision** in Cassandra data modeling.

### The Perfect Partition Key

```
┌─────────────────────────────────────────────────────────────────┐
│  A GOOD PARTITION KEY must satisfy ALL of these:                 │
│                                                                   │
│  1. HIGH CARDINALITY                                             │
│     → Many unique values (user_id ✅, country ❌)               │
│     → Ensures data spreads across all nodes                     │
│                                                                   │
│  2. EVEN DISTRIBUTION                                            │
│     → Each partition roughly the same size                      │
│     → No "hot" partitions that one node handles more than others│
│                                                                   │
│  3. ALWAYS IN THE WHERE CLAUSE                                   │
│     → Every query MUST specify the partition key                │
│     → If users always search by X, then X should be the PK     │
│                                                                   │
│  4. RIGHT-SIZED PARTITIONS                                       │
│     → Not too small (millions of tiny partitions = overhead)    │
│     → Not too large (giant partitions = slow reads, GC issues)  │
│     → Target: 10MB - 100MB per partition                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Partition Key Examples — Good vs Bad

| Use Case | Bad Partition Key | Why Bad | Good Partition Key | Why Good |
|----------|------------------|---------|-------------------|----------|
| User profiles | `country` | Low cardinality, massive partitions | `user_id` | Unique per user, perfectly distributed |
| Sensor data | `sensor_id` | Unbounded growth over time | `(sensor_id, date)` | Bounded by day, predictable size |
| Chat messages | `sender_id` | Need messages BY conversation | `conversation_id` | All messages for a chat together |
| Orders | `status` | Only ~5 values, huge partitions | `customer_id` | Orders per customer, good distribution |
| Logs | `log_level` | Only 5 values (DEBUG, INFO...) | `(service, date)` | Bounded, queryable |
| Products | `category` | Uneven (electronics vs niche) | `product_id` | Unique, even distribution |

---

## 🔥 4. Partition Sizing — Avoiding the Monster Partition

A partition that grows without bounds will eventually kill your node. Here's how to calculate and control partition size.

### Calculating Partition Size

```
┌─────────────────────────────────────────────────────────────────┐
│  PARTITION SIZE FORMULA:                                         │
│                                                                   │
│  Size = Nrows × (Srow_avg)                                      │
│                                                                   │
│  Where:                                                          │
│  Nrows = number of rows per partition                            │
│  Srow_avg = average size of one row (all columns)               │
│                                                                   │
│  GUIDELINES:                                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Rows per partition:   < 100,000  (ideal)               │   │
│  │  Size per partition:   < 100 MB   (hard limit ~2 billion)│   │
│  │  Columns per partition: < 100,000 (total cells)         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  EXAMPLE CALCULATION:                                            │
│  ─────────────────────                                           │
│  Table: messages_by_conversation                                 │
│  Avg message size: ~500 bytes                                    │
│  Messages per conversation: ???                                  │
│                                                                   │
│  Scenario A: Small group chat (100 messages/day × 365 days)     │
│  → 36,500 rows × 500 bytes = ~18 MB ✅ Perfect!                │
│                                                                   │
│  Scenario B: Active community (10,000 messages/day × 365 days) │
│  → 3,650,000 rows × 500 bytes = ~1.8 GB ❌ WAY TOO BIG!       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Bucketing — The Solution for Large Partitions

When a partition grows too large, **bucket** it by adding a time component to the partition key:

```sql
-- ❌ BEFORE: Unbounded partition growth
CREATE TABLE messages (
    conversation_id UUID,
    message_id TIMEUUID,
    body TEXT,
    PRIMARY KEY (conversation_id, message_id)
);
-- Problem: conversation with 10M messages = 1 massive partition


-- ✅ AFTER: Bucketed by month
CREATE TABLE messages_by_month (
    conversation_id UUID,
    month TEXT,             -- '2024-01' format
    message_id TIMEUUID,
    body TEXT,
    PRIMARY KEY ((conversation_id, month), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
-- Now: each conversation × month = one partition
-- 10,000 messages/month × 500 bytes = ~5 MB per partition ✅

-- Query latest messages:
SELECT * FROM messages_by_month
WHERE conversation_id = ? AND month = '2024-01'
LIMIT 50;

-- Query older messages (app knows which bucket to query):
SELECT * FROM messages_by_month
WHERE conversation_id = ? AND month = '2023-12'
LIMIT 50;
```

### Bucket Size Guidelines

```
┌──────────────────┬──────────────────┬──────────────────────────┐
│ Write Rate       │ Bucket By        │ Example                  │
├──────────────────┼──────────────────┼──────────────────────────┤
│ < 1000/day       │ Don't bucket     │ User profiles            │
│ 1K - 10K/day     │ Month or Quarter │ Chat messages            │
│ 10K - 100K/day   │ Week or Day      │ Activity feeds           │
│ 100K - 1M/day    │ Day or Hour      │ IoT sensor readings      │
│ > 1M/day         │ Hour or Minute   │ High-frequency trading   │
└──────────────────┴──────────────────┴──────────────────────────┘
```

---

## 🔥 5. Common Data Modeling Patterns

### Pattern 1: Time-Series Data (The #1 Cassandra Pattern)

```sql
-- IoT Sensor Data
CREATE TABLE sensor_readings (
    sensor_id TEXT,
    date TEXT,                        -- Bucket: one partition per sensor per day
    reading_time TIMESTAMP,
    temperature DOUBLE,
    humidity DOUBLE,
    pressure DOUBLE,
    PRIMARY KEY ((sensor_id, date), reading_time)
) WITH CLUSTERING ORDER BY (reading_time DESC)
AND default_time_to_live = 7776000    -- 90 days auto-delete
AND compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_size': 1,
    'compaction_window_unit': 'DAYS'
};

-- Query: Latest readings for a sensor today
SELECT * FROM sensor_readings
WHERE sensor_id = 'sensor-42' AND date = '2024-01-15'
LIMIT 100;

-- Query: Readings in a time range
SELECT * FROM sensor_readings
WHERE sensor_id = 'sensor-42' AND date = '2024-01-15'
AND reading_time > '2024-01-15 10:00:00'
AND reading_time < '2024-01-15 12:00:00';
```

### Pattern 2: Denormalized Lookup Tables

```sql
-- The "duplicate for different access patterns" pattern
-- Need to find users by email AND by user_id?
-- → Create TWO tables!

-- Table 1: Primary lookup by ID
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username TEXT,
    email TEXT,
    name TEXT
);

-- Table 2: Reverse lookup by email
CREATE TABLE users_by_email (
    email TEXT PRIMARY KEY,
    user_id UUID,
    username TEXT,
    name TEXT
);

-- Write to BOTH tables in a batch:
BEGIN BATCH
    INSERT INTO users (user_id, username, email, name)
    VALUES (?, 'alice', 'alice@example.com', 'Alice');
    
    INSERT INTO users_by_email (email, user_id, username, name)
    VALUES ('alice@example.com', ?, 'alice', 'Alice');
APPLY BATCH;

-- Now both queries are fast:
-- SELECT * FROM users WHERE user_id = ?;           ✅ O(1)
-- SELECT * FROM users_by_email WHERE email = ?;     ✅ O(1)
```

### Pattern 3: Leaderboard / Top-N

```sql
-- Gaming leaderboard — top scores per game per day
CREATE TABLE leaderboard (
    game_id TEXT,
    day TEXT,
    score INT,
    player_id UUID,
    player_name TEXT,
    PRIMARY KEY ((game_id, day), score, player_id)
) WITH CLUSTERING ORDER BY (score DESC, player_id ASC);

-- Get top 10 players today:
SELECT * FROM leaderboard
WHERE game_id = 'chess' AND day = '2024-01-15'
LIMIT 10;

-- Result is automatically sorted by score DESC ✅
```

### Pattern 4: Event Sourcing / Activity Feed

```sql
-- User activity feed (Instagram/Twitter style)
CREATE TABLE user_feed (
    user_id UUID,
    activity_time TIMEUUID,
    activity_type TEXT,      -- 'post', 'like', 'comment', 'follow'
    target_id UUID,
    target_type TEXT,        -- 'photo', 'video', 'user'
    description TEXT,
    PRIMARY KEY (user_id, activity_time)
) WITH CLUSTERING ORDER BY (activity_time DESC)
AND default_time_to_live = 2592000;  -- 30 days

-- Get recent activity for a user:
SELECT * FROM user_feed
WHERE user_id = ?
LIMIT 20;
```

### Pattern 5: Shopping Cart

```sql
-- E-commerce shopping cart
CREATE TABLE shopping_cart (
    user_id UUID,
    product_id UUID,
    product_name TEXT,       -- Denormalized from products table
    price DECIMAL,           -- Denormalized (snapshot at add time)
    quantity INT,
    added_at TIMESTAMP,
    PRIMARY KEY (user_id, product_id)
);

-- Get full cart:
SELECT * FROM shopping_cart WHERE user_id = ?;

-- Add/update item (UPSERT — same PK overwrites):
INSERT INTO shopping_cart (user_id, product_id, product_name, price, quantity, added_at)
VALUES (?, ?, 'Widget', 29.99, 2, toTimestamp(now()));

-- Remove item:
DELETE FROM shopping_cart WHERE user_id = ? AND product_id = ?;

-- Clear cart:
DELETE FROM shopping_cart WHERE user_id = ?;
```

### Pattern 6: Wide Row / Sparse Columns

```sql
-- Dynamic attributes per entity (like EAV in SQL, but better)
CREATE TABLE product_attributes (
    product_id UUID,
    attribute_name TEXT,
    attribute_value TEXT,
    PRIMARY KEY (product_id, attribute_name)
);

-- Insert various attributes:
INSERT INTO product_attributes (product_id, attribute_name, attribute_value)
VALUES (?, 'color', 'red');
INSERT INTO product_attributes (product_id, attribute_name, attribute_value)
VALUES (?, 'size', 'XL');
INSERT INTO product_attributes (product_id, attribute_name, attribute_value)
VALUES (?, 'weight', '250g');

-- Get all attributes for a product:
SELECT * FROM product_attributes WHERE product_id = ?;

-- Get specific attribute:
SELECT attribute_value FROM product_attributes
WHERE product_id = ? AND attribute_name = 'color';
```

---

## 🔥 6. Real-World Data Model — E-Commerce Platform

Let's design a complete data model for an e-commerce platform.

### Query Patterns

```
Q1: Get product details by product_id
Q2: List products by category (sorted by price)
Q3: Get all orders for a customer (latest first)
Q4: Get order details (all items in an order)
Q5: Get product reviews (latest first)
Q6: Search products by brand + category
```

### Complete Schema

```sql
-- ═══════════════════════════════════════════════════
-- Q1: Product details
-- ═══════════════════════════════════════════════════
CREATE TABLE products (
    product_id UUID PRIMARY KEY,
    name TEXT,
    description TEXT,
    brand TEXT,
    category TEXT,
    price DECIMAL,
    stock_quantity INT,
    image_urls LIST<TEXT>,
    specifications MAP<TEXT, TEXT>,
    avg_rating FLOAT,
    review_count INT,
    created_at TIMESTAMP
);

-- ═══════════════════════════════════════════════════
-- Q2: Products by category
-- ═══════════════════════════════════════════════════
CREATE TABLE products_by_category (
    category TEXT,
    price DECIMAL,
    product_id UUID,
    name TEXT,
    brand TEXT,
    image_url TEXT,
    avg_rating FLOAT,
    PRIMARY KEY (category, price, product_id)
) WITH CLUSTERING ORDER BY (price ASC, product_id ASC);

-- SELECT * FROM products_by_category
-- WHERE category = 'Electronics' LIMIT 20;

-- ═══════════════════════════════════════════════════
-- Q3: Customer orders
-- ═══════════════════════════════════════════════════
CREATE TABLE orders_by_customer (
    customer_id UUID,
    order_date TIMESTAMP,
    order_id UUID,
    total_amount DECIMAL,
    status TEXT,
    item_count INT,
    PRIMARY KEY (customer_id, order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC, order_id ASC);

-- SELECT * FROM orders_by_customer
-- WHERE customer_id = ? LIMIT 10;

-- ═══════════════════════════════════════════════════
-- Q4: Order details (items in an order)
-- ═══════════════════════════════════════════════════
CREATE TABLE order_items (
    order_id UUID,
    product_id UUID,
    product_name TEXT,      -- Denormalized
    quantity INT,
    unit_price DECIMAL,
    line_total DECIMAL,
    order_status TEXT STATIC,        -- Shared across all items
    shipping_address FROZEN<address> STATIC,
    PRIMARY KEY (order_id, product_id)
);

-- SELECT * FROM order_items WHERE order_id = ?;

-- ═══════════════════════════════════════════════════
-- Q5: Product reviews
-- ═══════════════════════════════════════════════════
CREATE TABLE reviews_by_product (
    product_id UUID,
    review_time TIMEUUID,
    reviewer_id UUID,
    reviewer_name TEXT,     -- Denormalized
    rating INT,
    title TEXT,
    body TEXT,
    helpful_count INT,
    PRIMARY KEY (product_id, review_time)
) WITH CLUSTERING ORDER BY (review_time DESC);

-- SELECT * FROM reviews_by_product
-- WHERE product_id = ? LIMIT 10;

-- ═══════════════════════════════════════════════════
-- Q6: Products by brand + category
-- ═══════════════════════════════════════════════════
CREATE TABLE products_by_brand_category (
    brand TEXT,
    category TEXT,
    product_id UUID,
    name TEXT,
    price DECIMAL,
    avg_rating FLOAT,
    PRIMARY KEY ((brand, category), product_id)
);

-- SELECT * FROM products_by_brand_category
-- WHERE brand = 'Apple' AND category = 'Phones';
```

### The Data Flow (Write Path)

```
┌─────────────────────────────────────────────────────────────────┐
│  When a new PRODUCT is added:                                    │
│                                                                   │
│  Application writes to 3 tables in a BATCH:                     │
│  1. products (main lookup)                                      │
│  2. products_by_category (browse by category)                   │
│  3. products_by_brand_category (brand + category filter)        │
│                                                                   │
│  When a new ORDER is placed:                                     │
│  Application writes to 2 tables in a BATCH:                     │
│  1. orders_by_customer (customer's order history)               │
│  2. order_items (order details)                                 │
│                                                                   │
│  When a REVIEW is posted:                                        │
│  Application writes to:                                          │
│  1. reviews_by_product (the review itself)                      │
│  2. products (update avg_rating, review_count)                  │
│                                                                   │
│  💡 Yes, writing one logical event may touch 2-3 tables.        │
│  This is NORMAL in Cassandra. Write amplification is the        │
│  trade-off for blazing-fast reads.                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔥 7. Handling Relationships in Cassandra

Since there are no JOINs, you need patterns for handling relationships.

### One-to-Many: Clustering Keys

```sql
-- One conversation has many messages
-- → Messages are clustering rows within conversation partition
CREATE TABLE messages (
    conversation_id UUID,    -- The "one"
    message_id TIMEUUID,     -- The "many"
    body TEXT,
    PRIMARY KEY (conversation_id, message_id)
);
```

### Many-to-Many: Dual Tables

```sql
-- Users follow other users (social network)

-- Q: Who does user X follow?
CREATE TABLE following (
    user_id UUID,
    followed_id UUID,
    followed_name TEXT,
    followed_since TIMESTAMP,
    PRIMARY KEY (user_id, followed_id)
);

-- Q: Who follows user X?
CREATE TABLE followers (
    user_id UUID,
    follower_id UUID,
    follower_name TEXT,
    followed_since TIMESTAMP,
    PRIMARY KEY (user_id, follower_id)
);

-- When user A follows user B, write to BOTH tables:
BEGIN BATCH
    INSERT INTO following (user_id, followed_id, followed_name, followed_since)
    VALUES (user_A, user_B, 'Bob', toTimestamp(now()));
    
    INSERT INTO followers (user_id, follower_id, follower_name, followed_since)
    VALUES (user_B, user_A, 'Alice', toTimestamp(now()));
APPLY BATCH;
```

### Hierarchical Data: Materialized Paths

```sql
-- Category hierarchy (Electronics > Phones > Smartphones)
CREATE TABLE categories (
    category_id UUID PRIMARY KEY,
    name TEXT,
    parent_id UUID,
    path TEXT,               -- 'electronics/phones/smartphones'
    depth INT
);

-- Children of a category
CREATE TABLE subcategories (
    parent_id UUID,
    child_id UUID,
    child_name TEXT,
    PRIMARY KEY (parent_id, child_name, child_id)
);
-- Sorted alphabetically by child_name within each parent
```

---

## 🔥 8. Anti-Patterns — Things That Will Break Your Cluster

### Anti-Pattern 1: The Giant Partition

```
❌ BAD:
CREATE TABLE events (
    event_type TEXT,           -- Only ~10 values!
    event_time TIMESTAMP,
    data TEXT,
    PRIMARY KEY (event_type, event_time)
);
-- "login" partition could have BILLIONS of rows!

✅ FIX: Add time bucket
CREATE TABLE events (
    event_type TEXT,
    date TEXT,
    event_time TIMESTAMP,
    data TEXT,
    PRIMARY KEY ((event_type, date), event_time)
);
```

### Anti-Pattern 2: Using Cassandra Like SQL

```
❌ BAD: Normalize and plan to JOIN later
-- Users table + Addresses table + JOIN them
-- CASSANDRA HAS NO JOINS!

✅ FIX: Denormalize — embed address in users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name TEXT,
    home_address FROZEN<address>,
    work_address FROZEN<address>
);
```

### Anti-Pattern 3: Secondary Indexes on High-Cardinality Columns

```
❌ BAD:
CREATE INDEX ON users (email);
-- email is unique for every user = high cardinality
-- Query touches ALL nodes in the cluster

✅ FIX: Create a lookup table
CREATE TABLE users_by_email (
    email TEXT PRIMARY KEY,
    user_id UUID
);
```

### Anti-Pattern 4: Queue / Delete-Heavy Workloads

```
❌ BAD: Using Cassandra as a message queue
-- INSERT → process → DELETE → repeat
-- Creates MASSIVE numbers of tombstones
-- Reads slow down as tombstones accumulate

✅ FIX: Use a real queue (Kafka, RabbitMQ) or use TTL instead of DELETE
```

### Anti-Pattern 5: Reading Before Writing

```
❌ BAD:
-- "Read the current value, modify it, write it back"
-- This is a read-modify-write cycle
-- In a distributed system, another node may write between your read and write!
-- You need LWT (slow) to make this safe.

✅ FIX:
-- Use counters for counting
-- Use LWT only when truly needed
-- Design to avoid read-before-write entirely
-- Write the final state, not the delta
```

### Anti-Pattern 6: ALLOW FILTERING Everywhere

```
❌ BAD:
SELECT * FROM users WHERE age > 25 ALLOW FILTERING;
-- Full table scan across ALL nodes!

✅ FIX:
-- Either create a table designed for this query
-- Or use SAI indexes if the pattern is occasional
-- Or move this query to an analytics system (Spark + Cassandra)
```

---

## 📋 Data Modeling Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│              CASSANDRA DATA MODELING RULES                        │
│                                                                   │
│  1. Queries come FIRST, tables come SECOND                      │
│  2. One table per query pattern                                 │
│  3. Partition key = your WHERE clause equality filter            │
│  4. Clustering key = your ORDER BY / range filter               │
│  5. Denormalize aggressively — duplicating data is OK            │
│  6. Keep partitions under 100MB / 100K rows                     │
│  7. Use time buckets for unbounded growth                       │
│  8. Use batches to keep denormalized tables in sync             │
│  9. Avoid secondary indexes on high-cardinality columns         │
│  10. Never use ALLOW FILTERING in production                    │
│  11. Never use Cassandra as a queue (delete-heavy = bad)        │
│  12. Avoid read-before-write patterns                           │
│  13. Use TIMEUUID for time-ordered data                         │
│  14. Use TTL for auto-expiring data                             │
│  15. Test with realistic data volumes — problems appear at scale│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Chapter Summary — What You Now Know

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  ✅ Query-first design: define queries → then design tables     │
│  ✅ One table per query pattern (denormalize!)                   │
│  ✅ Partition key = distribution + query filter                  │
│  ✅ Clustering key = sort order within partition                 │
│  ✅ Keep partitions < 100MB (use time bucketing)                │
│  ✅ Duplicate data across tables for different access patterns  │
│  ✅ Use batches to sync denormalized tables                     │
│  ✅ Time-series: (entity, time_bucket) as partition key         │
│  ✅ Many-to-many: two lookup tables                             │
│  ✅ Avoid: giant partitions, ALLOW FILTERING, queue patterns    │
│  ✅ Write amplification is the cost of fast reads               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⏭️ What's Next?

| Next Chapter | What You'll Learn |
|-------------|-------------------|
| [3D.4 — Cassandra Operations & Tuning](./04-Cassandra-Operations.md) | nodetool, repair, monitoring, compaction tuning, production best practices |

---

> **"In Cassandra, data modeling IS performance tuning. Get the model right, and the cluster practically runs itself. Get it wrong, and no amount of hardware will save you."** 🏗️
