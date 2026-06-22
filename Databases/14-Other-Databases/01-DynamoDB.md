# ⚡ Chapter 3G.1 — DynamoDB — AWS's Serverless NoSQL Powerhouse

> **Level:** 🟡 Intermediate | ⭐ Must-Know | 🔥 High Demand
> **Time to Master:** ~4–5 hours
> **Prerequisites:** Chapter 3A.1 (NoSQL Overview), Chapter 1.6 (CAP Theorem)

> **"DynamoDB doesn't just scale — it scales while you sleep. Zero servers, zero headaches, infinite throughput."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why Amazon built DynamoDB** and the legendary paper behind it
- Master the **Partition Key + Sort Key** model — the foundation of everything
- Design **GSI (Global Secondary Index)** and **LSI (Local Secondary Index)** like a pro
- Understand **DynamoDB Streams** — the real-time event backbone
- Compare **On-Demand vs Provisioned** capacity modes (and when each saves you money)
- Apply the legendary **Single-Table Design** pattern used at Amazon scale
- Know the **pricing model** so you never get a surprise AWS bill

---

## 1. The Origin Story — Why DynamoDB Exists

### Amazon's Worst Day: Christmas 2004

Imagine this: It's Christmas shopping season. Amazon.com is processing millions of orders. And then...

```
🔴 THE DISASTER:
   → Oracle database hit scaling limits
   → Shopping carts started failing
   → Customers couldn't check out
   → Amazon lost MILLIONS in revenue
   → Engineers worked 72-hour shifts
   
   Jeff Bezos said: "NEVER AGAIN."
```

Amazon's top engineers wrote the legendary **Dynamo Paper (2007)** — one of the most influential papers in distributed systems history. The core idea:

> **"We will build a database that NEVER goes down, scales INFINITELY, and requires ZERO database administration."**

DynamoDB (launched **2012**) is the fully managed, production-grade evolution of that vision.

### Who Uses DynamoDB?

```
┌─────────────────────────────────────────────────────────────┐
│                 DynamoDB Powers...                           │
├─────────────────────────────────────────────────────────────┤
│  🛒 Amazon.com       → Shopping cart, order history         │
│  📱 Snapchat         → 10B+ stories/day storage            │
│  🎮 Epic Games       → Fortnite player profiles            │
│  📺 Netflix          → Streaming metadata                   │
│  🏦 Capital One      → Banking transactions                │
│  🎵 Spotify          → Playlist & user data                │
│  🚗 Lyft             → Ride matching in real-time          │
│  📦 Nike             → E-commerce at global scale          │
│  🏥 Samsung Health   → 300M+ users health data             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. DynamoDB Architecture — How It Works Under the Hood

### The 30-Second Mental Model

```
Traditional Database:                    DynamoDB:
┌──────────────┐                        ┌─────────────────────────┐
│   One Big     │                        │  Your Request           │
│   Server      │                        │       │                 │
│ ┌──────────┐ │                        │       ▼                 │
│ │  ALL     │ │                        │  ┌─────────────┐       │
│ │  YOUR    │ │                        │  │  DynamoDB    │       │
│ │  DATA    │ │                        │  │  Router      │       │
│ │  HERE    │ │                        │  └──┬───┬───┬──┘       │
│ └──────────┘ │                        │     │   │   │          │
│ 😰 Hope it  │                        │     ▼   ▼   ▼          │
│   doesn't   │                        │  ┌──┐ ┌──┐ ┌──┐       │
│   crash...  │                        │  │P1│ │P2│ │P3│  ...   │
└──────────────┘                        │  └──┘ └──┘ └──┘       │
                                        │  Data auto-spread      │
                                        │  across partitions     │
                                        └─────────────────────────┘
```

### The Key Concepts

```
┌─────────────────────────────────────────────────────────────────┐
│                   DynamoDB Key Concepts                         │
├──────────────┬──────────────────────────────────────────────────┤
│  Table       │  A collection of items (like a SQL table)       │
│  Item        │  A single record (like a SQL row)               │
│  Attribute   │  A data field (like a SQL column)               │
│  Primary Key │  Uniquely identifies each item                   │
│  Partition   │  Physical storage unit (managed by AWS)          │
│  Region      │  Geographic location of your table               │
├──────────────┴──────────────────────────────────────────────────┤
│  ⚡ KEY DIFFERENCE FROM SQL:                                    │
│  → No fixed schema! Each item can have different attributes    │
│  → Only the Primary Key attributes are required                │
│  → Everything else is flexible per item                        │
└─────────────────────────────────────────────────────────────────┘
```

### How Data Gets Stored — Partitions

When you create a table, DynamoDB automatically creates **partitions** (physical storage nodes). Your data is distributed across these partitions using a **hash function** on the Partition Key.

```
Table: Users
Partition Key: user_id

Your Data:
  user_id=101 → Hash("101") = 0x3A → Partition A
  user_id=202 → Hash("202") = 0x7F → Partition B
  user_id=303 → Hash("303") = 0x3B → Partition A
  user_id=404 → Hash("404") = 0xC2 → Partition C

┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│  Partition A   │  │  Partition B   │  │  Partition C   │
│  ┌──────────┐  │  │  ┌──────────┐  │  │  ┌──────────┐  │
│  │ user=101 │  │  │  │ user=202 │  │  │  │ user=404 │  │
│  │ user=303 │  │  │  │          │  │  │  │          │  │
│  └──────────┘  │  │  └──────────┘  │  │  └──────────┘  │
│  Max 10GB each │  │  Auto-split   │  │  Auto-managed  │
└────────────────┘  └────────────────┘  └────────────────┘

💡 DynamoDB handles ALL of this automatically.
   You never see, manage, or worry about partitions.
```

### Partition Limits — What You Must Know

| Metric | Limit Per Partition |
|--------|-------------------|
| Storage | 10 GB max |
| Read Capacity | 3,000 RCU (Read Capacity Units) |
| Write Capacity | 1,000 WCU (Write Capacity Units) |

> ⚠️ **Hot Partition Problem**: If one partition key gets way more traffic than others, that single partition becomes a bottleneck — even though DynamoDB has plenty of overall capacity. This is the **#1 DynamoDB design mistake**.

---

## 3. Primary Keys — The Most Important Decision You'll Ever Make

### Two Types of Primary Keys

```
Option 1: Simple Primary Key (Partition Key Only)
─────────────────────────────────────────────────
  ┌──────────────────┐
  │   Partition Key   │  ← Must be UNIQUE per item
  │   (Hash Key)      │
  └──────────────────┘
  
  Example: Users table → Partition Key = user_id
  Each user_id MUST be unique.

Option 2: Composite Primary Key (Partition Key + Sort Key)
─────────────────────────────────────────────────────────
  ┌──────────────────┬──────────────────┐
  │  Partition Key    │   Sort Key       │
  │  (Hash Key)       │  (Range Key)     │
  └──────────────────┴──────────────────┘
  
  Example: Orders table → PK = customer_id + SK = order_date
  Same customer_id can have MULTIPLE orders (different dates).
  The COMBINATION of PK + SK must be unique.
```

### When to Use Which?

| Use Case | Key Type | Example |
|----------|----------|---------|
| Users by ID | Simple PK | `PK = user_id` |
| User's orders | Composite | `PK = user_id, SK = order_date` |
| Forum posts | Composite | `PK = forum_id, SK = post_timestamp` |
| Game scores | Composite | `PK = game_id, SK = score` |
| Config settings | Simple PK | `PK = setting_name` |
| IoT sensor data | Composite | `PK = device_id, SK = timestamp` |

### The Sort Key Superpower

The Sort Key isn't just for uniqueness — it enables **range queries** within a partition:

```python
# ── With Composite Key: PK=customer_id, SK=order_date ──

# Get ALL orders for customer "C100"
table.query(
    KeyConditionExpression=Key('customer_id').eq('C100')
)

# Get orders AFTER January 2024
table.query(
    KeyConditionExpression=
        Key('customer_id').eq('C100') & 
        Key('order_date').gt('2024-01-01')
)

# Get orders BETWEEN two dates
table.query(
    KeyConditionExpression=
        Key('customer_id').eq('C100') & 
        Key('order_date').between('2024-01-01', '2024-06-30')
)

# Get the LATEST order (sort descending, limit 1)
table.query(
    KeyConditionExpression=Key('customer_id').eq('C100'),
    ScanIndexForward=False,  # ← Descending order
    Limit=1
)
```

> 💡 **Pro Tip**: Sort Key supports these operators: `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `begins_with()`. Design your sort keys to leverage these!

### Sort Key Design Patterns (🔥 Game Changer)

```
Pattern 1: Hierarchical Sort Keys
──────────────────────────────────
PK = "COUNTRY#US"
SK = "STATE#CA#CITY#SanFrancisco#ZIP#94105"

→ Query begins_with("STATE#CA") → All California data
→ Query begins_with("STATE#CA#CITY#San") → San Francisco + San Diego
→ Query exact SK → Specific ZIP code

Pattern 2: Time-based Sort Keys
───────────────────────────────
PK = "USER#ritesh"
SK = "ORDER#2024-06-15T10:30:00Z"

→ Query begins_with("ORDER#2024-06") → June 2024 orders
→ Query between("ORDER#2024-01-01", "ORDER#2024-12-31") → All 2024 orders

Pattern 3: Type-prefixed Sort Keys (Single-Table Design!)
─────────────────────────────────────────────────────────
PK = "USER#ritesh"
SK = "PROFILE"           → User profile data
SK = "ORDER#2024-001"    → Order record  
SK = "ADDRESS#home"      → Home address
SK = "ADDRESS#work"      → Work address

→ Query begins_with("ORDER#") → All orders for this user
→ Query SK = "PROFILE" → Just the profile
→ ONE query gets ALL user data! 🚀
```

---

## 4. Read & Write Operations — The Complete API

### Write Operations

```python
import boto3
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

# ── PutItem: Create or REPLACE an entire item ──
table.put_item(
    Item={
        'user_id': 'U100',
        'name': 'Ritesh',
        'email': 'ritesh@example.com',
        'age': 28,
        'skills': ['Python', 'AWS', 'DynamoDB'],
        'address': {
            'city': 'Delhi',
            'country': 'India'
        }
    }
)
# ⚠️ If user_id='U100' already exists, it OVERWRITES the entire item!

# ── UpdateItem: Modify specific attributes (no overwrite!) ──
table.update_item(
    Key={'user_id': 'U100'},
    UpdateExpression='SET age = :newage, #n = :newname ADD login_count :inc',
    ExpressionAttributeNames={'#n': 'name'},       # 'name' is reserved word
    ExpressionAttributeValues={
        ':newage': 29,
        ':newname': 'Ritesh Singh',
        ':inc': 1                                    # Atomic increment!
    }
)

# ── DeleteItem: Remove an item ──
table.delete_item(
    Key={'user_id': 'U100'}
)

# ── Conditional Writes (Optimistic Locking!) ──
table.put_item(
    Item={'user_id': 'U100', 'name': 'Ritesh', 'version': 1},
    ConditionExpression='attribute_not_exists(user_id)'  # Only if doesn't exist
)

table.update_item(
    Key={'user_id': 'U100'},
    UpdateExpression='SET #n = :name, version = :newver',
    ConditionExpression='version = :curver',        # Optimistic lock!
    ExpressionAttributeNames={'#n': 'name'},
    ExpressionAttributeValues={
        ':name': 'Updated Name',
        ':curver': 1,
        ':newver': 2
    }
)
```

### Read Operations — Query vs Scan

```
┌─────────────────────────────────────────────────────────────────┐
│              Query vs Scan — KNOW THE DIFFERENCE!               │
├────────────────────────────┬────────────────────────────────────┤
│         QUERY ✅           │          SCAN ❌ (Usually)         │
├────────────────────────────┼────────────────────────────────────┤
│ Uses Primary Key / Index   │ Reads EVERY item in the table     │
│ Fast — O(log n)            │ Slow — O(n), reads ALL data       │
│ Cheap — reads only needed  │ Expensive — charged for ALL data  │
│ Efficient at any scale     │ Gets WORSE as table grows         │
│ Returns sorted by Sort Key │ Returns in no particular order    │
├────────────────────────────┴────────────────────────────────────┤
│  🔥 RULE: NEVER use Scan in production unless you truly need   │
│     to process every single item (e.g., data migration).       │
└─────────────────────────────────────────────────────────────────┘
```

```python
# ── Query: Fast, efficient, uses keys ──
response = table.query(
    KeyConditionExpression=Key('customer_id').eq('C100'),
    FilterExpression=Attr('status').eq('shipped'),     # Post-filter (still reads all!)
    ProjectionExpression='order_id, total, #s',        # Only return these fields
    ExpressionAttributeNames={'#s': 'status'},
    ScanIndexForward=False,                            # Descending sort
    Limit=10                                           # Max 10 items
)
items = response['Items']

# ── GetItem: Get exactly ONE item by full Primary Key ──
response = table.get_item(
    Key={'user_id': 'U100'},
    ConsistentRead=True                                # Strong consistency
)
item = response.get('Item')

# ── BatchGetItem: Get up to 100 items in ONE call ──
response = dynamodb.batch_get_item(
    RequestItems={
        'Users': {
            'Keys': [
                {'user_id': 'U100'},
                {'user_id': 'U200'},
                {'user_id': 'U300'}
            ]
        }
    }
)

# ── Scan: Read entire table (AVOID in production!) ──
response = table.scan(
    FilterExpression=Attr('age').gt(25),               # Applied AFTER reading all
    Limit=100
)
```

### Consistency Models

```
┌─────────────────────────────────────────────────────────────┐
│           DynamoDB Consistency Models                        │
├──────────────────────┬──────────────────────────────────────┤
│  Eventually          │  Strongly                            │
│  Consistent Read     │  Consistent Read                     │
│  (Default)           │  (ConsistentRead=True)               │
├──────────────────────┼──────────────────────────────────────┤
│  Returns data from   │  Always returns latest data          │
│  any replica         │  Reads from leader partition         │
│  May be stale by     │  Never stale                        │
│  ~milliseconds       │                                      │
│  0.5 RCU per 4KB     │  1.0 RCU per 4KB                   │
│  (HALF the cost!)    │  (Double the cost)                  │
│  Higher availability │  Slightly lower availability        │
├──────────────────────┴──────────────────────────────────────┤
│  💡 Use Eventually Consistent by default.                   │
│     Use Strongly Consistent only when you MUST have the     │
│     latest data (e.g., financial transactions, inventory).  │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Secondary Indexes — Query Your Data in Multiple Ways

### The Problem

```
Table: Orders
PK = customer_id
SK = order_date

✅ "Get all orders for customer C100"          → Query on PK
✅ "Get orders for C100 between Jan-Jun 2024"  → Query on PK + SK range
❌ "Get all orders with status = 'shipped'"    → SCAN! (No key for status)
❌ "Get all orders by product_id"              → SCAN! (No key for product)
```

**Solution**: Secondary Indexes — create alternative access patterns without duplicating data.

### GSI (Global Secondary Index)

```
┌──────────────────────────────────────────────────────────────┐
│              Global Secondary Index (GSI)                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Think of a GSI as a "second table" that DynamoDB            │
│  automatically maintains for you.                            │
│                                                              │
│  Base Table:                    GSI:                         │
│  PK = customer_id               PK = status                 │
│  SK = order_date                SK = order_date              │
│  ┌────────┬──────────┬───────┐  ┌────────┬──────────┐       │
│  │cust_id │order_date│status │  │status  │order_date│       │
│  │C100    │2024-01-15│shipped│→ │shipped │2024-01-15│       │
│  │C100    │2024-03-20│pending│→ │pending │2024-03-20│       │
│  │C200    │2024-02-10│shipped│→ │shipped │2024-02-10│       │
│  └────────┴──────────┴───────┘  └────────┴──────────┘       │
│                                                              │
│  Now you can Query by status!                                │
│  "Get all shipped orders" → Query GSI with PK = "shipped"   │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  Properties:                                                 │
│  • Different PK (and optionally SK) from base table         │
│  • Spans ALL partitions (hence "Global")                    │
│  • Has its OWN throughput (separate RCU/WCU)                │
│  • Eventually consistent only (no strong consistency!)      │
│  • Up to 20 GSIs per table                                  │
│  • Can project a subset of attributes (saves cost!)         │
└──────────────────────────────────────────────────────────────┘
```

```python
# ── Create a table with GSI ──
table = dynamodb.create_table(
    TableName='Orders',
    KeySchema=[
        {'AttributeName': 'customer_id', 'KeyType': 'HASH'},
        {'AttributeName': 'order_date', 'KeyType': 'RANGE'}
    ],
    AttributeDefinitions=[
        {'AttributeName': 'customer_id', 'AttributeType': 'S'},
        {'AttributeName': 'order_date', 'AttributeType': 'S'},
        {'AttributeName': 'status', 'AttributeType': 'S'}
    ],
    GlobalSecondaryIndexes=[
        {
            'IndexName': 'StatusDateIndex',
            'KeySchema': [
                {'AttributeName': 'status', 'KeyType': 'HASH'},
                {'AttributeName': 'order_date', 'KeyType': 'RANGE'}
            ],
            'Projection': {'ProjectionType': 'ALL'},
            'ProvisionedThroughput': {
                'ReadCapacityUnits': 10,
                'WriteCapacityUnits': 5
            }
        }
    ],
    ProvisionedThroughput={
        'ReadCapacityUnits': 10,
        'WriteCapacityUnits': 10
    }
)

# ── Query the GSI ──
response = table.query(
    IndexName='StatusDateIndex',
    KeyConditionExpression=Key('status').eq('shipped')
)
```

### LSI (Local Secondary Index)

```
┌──────────────────────────────────────────────────────────────┐
│              Local Secondary Index (LSI)                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Same Partition Key as base table, but DIFFERENT Sort Key.   │
│                                                              │
│  Base Table:                    LSI:                         │
│  PK = customer_id               PK = customer_id (SAME!)    │
│  SK = order_date                SK = total_amount            │
│                                                              │
│  "Get customer C100's orders sorted by amount"               │
│  → Query LSI with PK=C100, sorted by total_amount           │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  Properties:                                                 │
│  • SAME PK as base table (hence "Local" to partition)       │
│  • Different SK — gives an alternative sort order           │
│  • Supports Strongly Consistent reads!                      │
│  • Shares throughput with base table (no separate RCU/WCU)  │
│  • Up to 5 LSIs per table                                   │
│  • ⚠️ MUST be created at table creation time (not after!)   │
│  • ⚠️ Limits partition size to 10GB (including LSI data)    │
└──────────────────────────────────────────────────────────────┘
```

### GSI vs LSI — Quick Decision Guide

| Feature | GSI 🌍 | LSI 📍 |
|---------|--------|--------|
| Partition Key | Different from base | Same as base |
| Sort Key | Optional different | Required different |
| When to Create | Anytime | Table creation only! |
| Consistency | Eventually only | Eventually + Strong |
| Throughput | Separate (own RCU/WCU) | Shared with base |
| Partition Limit | No extra limit | 10GB per partition (with LSI) |
| Max per Table | 20 | 5 |
| **When to Use** | New access pattern | Alternative sort within same PK |

> 💡 **Pro Tip**: When in doubt, use **GSI**. It's more flexible, can be added later, and doesn't impose partition size limits. LSI is a specialized tool for when you need strong consistency on an alternative sort order.

---

## 6. DynamoDB Streams — Real-Time Event Processing

### What Are Streams?

DynamoDB Streams captures a **time-ordered sequence of item-level changes** in your table — every insert, update, and delete.

```
┌─────────────────────────────────────────────────────────────┐
│               DynamoDB Streams — The Event Backbone          │
│                                                              │
│  Table Changes:           Stream:            Consumers:      │
│                                                              │
│  PutItem(user=101)  →  ┌──────────┐  →  🔧 Lambda Function │
│  UpdateItem(user=202)→ │  Stream   │  →  📊 Analytics       │
│  DeleteItem(user=303)→ │  Records  │  →  🔄 Replicate to   │
│                        │  (24hr    │      another table     │
│                        │   window) │  →  📧 Send email      │
│                        └──────────┘  →  🔍 Update ES index  │
│                                                              │
│  Each record contains:                                       │
│  • The full item (old image, new image, or both)            │
│  • Event type (INSERT, MODIFY, REMOVE)                      │
│  • Timestamp of the change                                  │
│  • Sequence number (ordering within a shard)                │
└─────────────────────────────────────────────────────────────┘
```

### Stream View Types

| View Type | What You Get | Use Case |
|-----------|-------------|----------|
| `KEYS_ONLY` | Only the key attributes | Lightweight triggers, cache invalidation |
| `NEW_IMAGE` | Entire item after the change | Replication, search indexing |
| `OLD_IMAGE` | Entire item before the change | Audit trails, undo operations |
| `NEW_AND_OLD_IMAGES` | Both before and after | Change comparison, compliance |

### Common Stream Patterns

```
Pattern 1: Cross-Region Replication (Global Tables)
────────────────────────────────────────────────────
  US-East Table ──Stream──→ Replicator ──→ EU-West Table
                                      ──→ AP-Tokyo Table
  (DynamoDB Global Tables does this automatically!)

Pattern 2: Search Indexing
──────────────────────────
  DynamoDB ──Stream──→ Lambda ──→ Elasticsearch/OpenSearch
  
  Every product update automatically updates the search index!

Pattern 3: Event-Driven Workflows
──────────────────────────────────
  Order Table ──Stream──→ Lambda ──→ SNS ──→ Email notification
                                         ──→ Inventory update
                                         ──→ Analytics pipeline

Pattern 4: Materialized Views / Aggregations
────────────────────────────────────────────
  Orders ──Stream──→ Lambda ──→ CustomerStats table
  
  Maintains real-time aggregates (total orders, total spent, etc.)
```

```python
# ── Enable Streams on a table ──
client = boto3.client('dynamodb')
client.update_table(
    TableName='Orders',
    StreamSpecification={
        'StreamEnabled': True,
        'StreamViewType': 'NEW_AND_OLD_IMAGES'
    }
)

# ── Lambda function to process stream events ──
def lambda_handler(event, context):
    for record in event['Records']:
        event_name = record['eventName']  # INSERT, MODIFY, REMOVE
        
        if event_name == 'INSERT':
            new_item = record['dynamodb']['NewImage']
            print(f"New order: {new_item}")
            # → Send welcome email, update analytics, etc.
            
        elif event_name == 'MODIFY':
            old_item = record['dynamodb']['OldImage']
            new_item = record['dynamodb']['NewImage']
            print(f"Updated: {old_item} → {new_item}")
            # → Detect status changes, trigger workflows
            
        elif event_name == 'REMOVE':
            old_item = record['dynamodb']['OldImage']
            print(f"Deleted: {old_item}")
            # → Clean up related data, audit log
```

---

## 7. Capacity Modes — On-Demand vs Provisioned

### The Two Modes

```
┌─────────────────────────────────────────────────────────────┐
│              Capacity Modes Comparison                       │
├────────────────────────────┬────────────────────────────────┤
│     ON-DEMAND 💸           │    PROVISIONED ⚙️              │
│     (Pay-per-request)      │    (Reserved capacity)         │
├────────────────────────────┼────────────────────────────────┤
│ Pay only for what you use  │ Set RCU/WCU in advance        │
│ No capacity planning       │ Must estimate traffic          │
│ Auto-scales instantly      │ Auto-scaling (with lag)        │
│ Higher per-request cost    │ Lower per-request cost         │
│ No throttling*             │ Throttling if capacity hit     │
│ Great for unpredictable    │ Great for predictable          │
│ traffic (spiky workloads)  │ traffic (steady workloads)     │
├────────────────────────────┴────────────────────────────────┤
│  * On-Demand has a 40K RCU / 40K WCU instant limit         │
│    (can be increased via AWS support)                       │
└─────────────────────────────────────────────────────────────┘
```

### Understanding RCU & WCU (Provisioned Mode)

```
RCU (Read Capacity Units):
──────────────────────────
1 RCU = 1 Strongly Consistent read/sec for items up to 4KB
      = 2 Eventually Consistent reads/sec for items up to 4KB

Example: Item size = 10KB
  → Strongly Consistent:   ⌈10/4⌉ = 3 RCU per read
  → Eventually Consistent: ⌈10/4⌉ / 2 = 1.5 → 2 RCU per read (round up)

WCU (Write Capacity Units):
───────────────────────────
1 WCU = 1 write/sec for items up to 1KB

Example: Item size = 3KB
  → ⌈3/1⌉ = 3 WCU per write

📊 Capacity Calculation Example:
   "We need 500 reads/sec (eventually consistent) for 8KB items"
   → RCUs per read = ⌈8/4⌉ = 2
   → Eventually consistent = 2 / 2 = 1 RCU per read
   → Total = 500 × 1 = 500 RCU needed
```

### When to Use Which Mode

| Scenario | Mode | Why |
|----------|------|-----|
| New app, unknown traffic | On-Demand | No guessing needed |
| Black Friday/flash sales | On-Demand | Handles spikes instantly |
| Steady enterprise workload | Provisioned | 5-7x cheaper |
| Dev/test environments | On-Demand | Minimal cost when idle |
| Cost-optimized production | Provisioned + Auto-scaling | Best of both worlds |
| Batch processing jobs | Provisioned (temporarily high) | Control costs |

> 💡 **Pro Tip**: Start with **On-Demand** while learning your traffic patterns. After 2-4 weeks of metrics, switch to **Provisioned with Auto-Scaling** to save 60-80% costs.

---

## 8. Single-Table Design — The DynamoDB Way 🔥

### Why Single-Table?

In SQL, you'd have 5-10 tables for a typical application. In DynamoDB, many experts recommend putting **everything in ONE table**.

```
SQL World (Multiple Tables):
┌──────┐  ┌──────────┐  ┌────────┐  ┌──────────┐
│Users │  │  Orders  │  │Products│  │ Reviews  │
└──────┘  └──────────┘  └────────┘  └──────────┘
    JOIN       JOIN          JOIN
    
DynamoDB Single-Table Design:
┌──────────────────────────────────────────────────────┐
│                    ONE TABLE                          │
│  Users, Orders, Products, Reviews — ALL HERE         │
│  Distinguished by PK/SK patterns                     │
└──────────────────────────────────────────────────────┘
```

### Why? Because of How DynamoDB Works:

1. **No JOINs** → You can't combine data from multiple tables in one query
2. **One Query, One Table** → Fetching from 3 tables = 3 API calls = 3x latency
3. **Transactions span tables** → But single-table is more efficient
4. **GSI limits** → 20 per table × 1 table = 20 GSIs for all your patterns

### Real-World Single-Table Example: E-Commerce

```
┌──────────────────┬──────────────────────┬────────────────────────────┐
│  PK              │  SK                  │  Attributes                │
├──────────────────┼──────────────────────┼────────────────────────────┤
│  USER#ritesh     │  PROFILE             │  name, email, joined_date  │
│  USER#ritesh     │  ORDER#2024-001      │  total, status, items      │
│  USER#ritesh     │  ORDER#2024-002      │  total, status, items      │
│  USER#ritesh     │  ADDRESS#home        │  street, city, zip         │
│  USER#ritesh     │  ADDRESS#work        │  street, city, zip         │
│  USER#ritesh     │  REVIEW#PROD#laptop  │  rating, comment, date     │
│                  │                      │                            │
│  PRODUCT#laptop  │  METADATA            │  name, price, category     │
│  PRODUCT#laptop  │  REVIEW#ritesh       │  rating, comment, date     │
│  PRODUCT#laptop  │  REVIEW#priya        │  rating, comment, date     │
│                  │                      │                            │
│  ORDER#2024-001  │  ORDERDETAIL         │  customer_id, total, date  │
│  ORDER#2024-001  │  ITEM#laptop         │  qty, price                │
│  ORDER#2024-001  │  ITEM#mouse          │  qty, price                │
└──────────────────┴──────────────────────┴────────────────────────────┘
```

### Access Patterns Solved by This Design:

```
Pattern 1: "Get user profile"
  → Query: PK = "USER#ritesh", SK = "PROFILE"
  → Result: 1 item (instant!)

Pattern 2: "Get all orders for a user"
  → Query: PK = "USER#ritesh", SK begins_with("ORDER#")
  → Result: All orders, sorted by date

Pattern 3: "Get complete user data (profile + orders + addresses)"
  → Query: PK = "USER#ritesh"
  → Result: EVERYTHING about this user in ONE query! 🚀

Pattern 4: "Get all reviews for a product"
  → Query: PK = "PRODUCT#laptop", SK begins_with("REVIEW#")
  → Result: All reviews

Pattern 5: "Get order with all items"
  → Query: PK = "ORDER#2024-001"
  → Result: Order details + all line items

Pattern 6: "Get all shipped orders" (needs GSI!)
  → GSI: PK = status, SK = order_date
  → Query GSI: PK = "shipped"
```

### The Single-Table Design Process

```
Step 1: List ALL your access patterns first
──────────────────────────────────────────
  1. Get user by ID
  2. Get user's orders
  3. Get order details with items
  4. Get product reviews
  5. Get orders by status
  6. Get user's addresses
  ... list EVERY query your app will make

Step 2: Design PK/SK to satisfy the most patterns
──────────────────────────────────────────────────
  Use entity-type prefixes: USER#, ORDER#, PRODUCT#
  Use Sort Key hierarchies for related data

Step 3: Add GSIs for remaining patterns
────────────────────────────────────────
  GSI1: Inverted index (GSI-PK = SK, GSI-SK = PK)
  GSI2: Status-based queries
  GSI3: Date-range queries

Step 4: Validate — walk through each access pattern
────────────────────────────────────────────────────
  Can every pattern be satisfied by Query (not Scan)?
  If yes → Design is solid! ✅
```

> ⚠️ **Important**: Single-Table Design is **powerful but complex**. For simpler applications or teams new to DynamoDB, **multi-table design** is perfectly fine. Use single-table when you need maximum performance and are comfortable with the complexity.

---

## 9. DynamoDB Transactions

```
# ── TransactWriteItems: All-or-nothing writes ──
client.transact_write_items(
    TransactItems=[
        {
            'Put': {
                'TableName': 'Orders',
                'Item': {
                    'PK': {'S': 'ORDER#2024-003'},
                    'SK': {'S': 'METADATA'},
                    'total': {'N': '99.99'},
                    'status': {'S': 'confirmed'}
                }
            }
        },
        {
            'Update': {
                'TableName': 'Orders',
                'Key': {
                    'PK': {'S': 'PRODUCT#laptop'},
                    'SK': {'S': 'METADATA'}
                },
                'UpdateExpression': 'SET stock = stock - :qty',
                'ExpressionAttributeValues': {':qty': {'N': '1'}},
                'ConditionExpression': 'stock > :zero',
                'ExpressionAttributeValues': {
                    ':qty': {'N': '1'},
                    ':zero': {'N': '0'}
                }
            }
        }
    ]
)
# Both succeed or BOTH fail — true ACID! (up to 100 items / 4MB)
```

### Transaction Limits

| Feature | Limit |
|---------|-------|
| Items per transaction | 100 |
| Total size per transaction | 4 MB |
| Cost | 2x normal WCU/RCU |
| Cross-table | ✅ Yes (within same region) |
| Cross-region | ❌ No |

---

## 10. Global Tables — Multi-Region Replication

```
┌──────────────────────────────────────────────────────────────┐
│              DynamoDB Global Tables                           │
│                                                              │
│    US-East-1          EU-West-1          AP-Tokyo-1          │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐         │
│   │  Orders  │◄────►│  Orders  │◄────►│  Orders  │         │
│   │  Table   │      │  Table   │      │  Table   │         │
│   └──────────┘      └──────────┘      └──────────┘         │
│        ▲                  ▲                  ▲              │
│        │                  │                  │              │
│     US Users           EU Users          Asia Users         │
│                                                              │
│  • Active-Active: Read AND Write in ANY region               │
│  • Sub-second replication between regions                    │
│  • Automatic conflict resolution (last-writer-wins)         │
│  • No application code changes needed!                      │
│  • 99.999% SLA (five nines!)                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 11. DynamoDB vs Other Databases

```
┌──────────────────────────────────────────────────────────────────────┐
│                    DynamoDB vs The World                             │
├──────────────┬──────────┬──────────┬──────────┬─────────────────────┤
│  Feature     │ DynamoDB │ MongoDB  │ Cassandra│ PostgreSQL          │
├──────────────┼──────────┼──────────┼──────────┼─────────────────────┤
│  Managed     │ Fully ☁️  │ Atlas ☁️  │ Astra ☁️  │ RDS/Aurora ☁️       │
│  Scaling     │ Auto     │ Manual   │ Manual   │ Vertical            │
│  Ops Effort  │ Zero     │ Low-Med  │ Medium   │ Medium              │
│  Query Power │ Limited  │ Rich     │ Limited  │ Full SQL            │
│  Transactions│ Yes (100)│ Yes      │ Light    │ Full ACID           │
│  Cost Model  │ Pay/use  │ Varies   │ Varies   │ Instance-based      │
│  JOINs       │ ❌       │ $lookup  │ ❌       │ ✅ Full             │
│  Consistency │ Tunable  │ Tunable  │ Tunable  │ Strong              │
│  Vendor Lock │ AWS only │ No       │ No       │ No                  │
│  Learning    │ Medium   │ Easy     │ Medium   │ Easy (SQL)          │
├──────────────┴──────────┴──────────┴──────────┴─────────────────────┤
│  💡 Choose DynamoDB when: Serverless, AWS-native, need zero-ops,    │
│     predictable latency at any scale, event-driven architectures.  │
│  ❌ Avoid DynamoDB when: Complex queries, ad-hoc analytics,        │
│     many-to-many relationships, or you need to avoid vendor lock.  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 12. Best Practices & Common Mistakes

### ✅ Do This

```
1. Design for access patterns FIRST, schema second
2. Use composite keys (PK + SK) for related data
3. Use GSIs for alternative access patterns
4. Use On-Demand for unpredictable traffic
5. Use Provisioned + Auto-Scaling for steady workloads
6. Keep items small (< 400KB limit, aim for < 10KB)
7. Use DynamoDB Streams for event-driven architecture
8. Enable Point-in-Time Recovery (PITR) for backups
9. Use DAX (DynamoDB Accelerator) for microsecond reads
10. Monitor with CloudWatch (throttle events, latency, errors)
```

### ❌ Don't Do This

```
1. DON'T use Scan in production (always Query with keys)
2. DON'T use a single attribute as PK with low cardinality
   (e.g., PK = "status" with values: "active"/"inactive" → hot partition!)
3. DON'T store large blobs (use S3 + store URL in DynamoDB)
4. DON'T forget about GSI write costs (every base write = GSI write too)
5. DON'T over-index (each GSI doubles your write costs for projected attrs)
6. DON'T use Strongly Consistent reads everywhere (2x cost, lower availability)
7. DON'T start with single-table design if your team is new to DynamoDB
8. DON'T put timestamps as Partition Key (creates hot partitions!)
9. DON'T ignore the 400KB item size limit
10. DON'T forget FilterExpression doesn't reduce RCU consumption
```

---

## 13. DAX — DynamoDB Accelerator (Microsecond Cache)

```
Without DAX:                        With DAX:
App → DynamoDB (1-10ms)             App → DAX Cache (microseconds!)
                                         ↓ (cache miss)
                                    DynamoDB (1-10ms)

DAX is an in-memory cache that sits in front of DynamoDB:
  • Drop-in replacement (same API — just change endpoint!)
  • Microsecond response times for cached reads
  • Write-through cache (writes go to both DAX and DynamoDB)
  • Ideal for read-heavy workloads
  • NOT for: strongly consistent reads, write-heavy workloads
```

---

## 🧠 Chapter Summary — Key Takeaways

```
┌─────────────────────────────────────────────────────────────────┐
│           DynamoDB — What You Must Remember                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. FULLY MANAGED: Zero servers, zero patching, zero headaches  │
│  2. KEY DESIGN IS EVERYTHING: PK + SK = your most important    │
│     decision. Design for access patterns, not entities.        │
│  3. QUERY, NEVER SCAN: If you're scanning, redesign your keys  │
│  4. GSI > LSI: GSIs are more flexible, can be added anytime    │
│  5. STREAMS = EVENT BACKBONE: Lambda + Streams = serverless    │
│     event-driven architecture                                  │
│  6. ON-DEMAND: Start here, switch to Provisioned later         │
│  7. SINGLE-TABLE: Powerful but not mandatory — evaluate first  │
│  8. GLOBAL TABLES: Multi-region active-active in one click     │
│  9. 400KB ITEM LIMIT: Keep items lean                          │
│  10. VENDOR LOCK: DynamoDB = AWS only. Plan accordingly.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

| Next Chapter | Topic |
|-------------|-------|
| [3G.2 — CouchDB & PouchDB](./02-CouchDB-PouchDB.md) | Offline-first databases with built-in sync |
| [3G.3 — InfluxDB & Time-Series](./03-TimeSeries-InfluxDB.md) | Time-series databases for IoT & monitoring |

---

> 💡 **Final Thought**: DynamoDB isn't just a database — it's a **paradigm shift**. You stop thinking about schemas and start thinking about **access patterns**. Once that clicks, you'll design systems that scale to millions of users without breaking a sweat.

---

*Chapter 3G.1 — Complete. You now think in Partition Keys and Sort Keys. Welcome to the DynamoDB world.* 🚀
