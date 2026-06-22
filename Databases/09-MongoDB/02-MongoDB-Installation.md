# 🍃 Chapter 3B.2 — MongoDB Installation & Setup

> **Level:** 🟢 Beginner
> **Time to Master:** ~1–2 hours
> **Prerequisites:** Chapter 3B.1 (MongoDB Architecture & Concepts)

> **"You can't drive a Ferrari by reading about it. Time to get your hands dirty."**

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Know every **MongoDB edition** and which one to pick
- Install MongoDB on **Windows, macOS, and Linux**
- Master **mongosh** — the modern MongoDB shell
- Use **MongoDB Compass** — the visual GUI
- Set up **MongoDB Atlas** — cloud database in 5 minutes
- Understand **connection strings** and connect from any language
- Configure MongoDB for **development and production**

---

## 1. MongoDB Editions — Which One Do You Need?

```
┌──────────────────────────────────────────────────────────────────┐
│                    MongoDB Editions                                │
├────────────────┬──────────────┬──────────────┬───────────────────┤
│                │  Community   │  Enterprise  │  Atlas (Cloud)    │
│                │  (Free)      │  (Paid)      │  (Free Tier!)     │
├────────────────┼──────────────┼──────────────┼───────────────────┤
│  Cost          │  $0          │  $$$         │  Free → $$$       │
│  Use Case      │  Dev/Small   │  Enterprise  │  Any scale        │
│  LDAP Auth     │  ❌          │  ✅          │  ✅               │
│  Encryption    │  At-rest ❌  │  ✅          │  ✅               │
│  Auditing      │  ❌          │  ✅          │  ✅               │
│  In-Memory     │  ❌          │  ✅          │  ❌               │
│  Ops Manager   │  ❌          │  ✅          │  Built-in         │
│  Support       │  Community   │  24/7 SLA    │  Included         │
│  Management    │  You         │  You         │  MongoDB Inc.     │
│  Best For      │  Learning    │  Banks/Corp  │  Most teams       │
│                │  & Startups  │              │                   │
└────────────────┴──────────────┴──────────────┴───────────────────┘
```

> 💡 **Recommendation**: 
> - **Learning?** → Use **Atlas Free Tier** (zero setup, nothing to install)
> - **Local dev?** → Install **Community Edition**
> - **Production?** → **Atlas** or **Enterprise** (depending on cloud vs on-premise)

---

## 2. Install MongoDB Community Edition

### 2.1 Windows Installation

```
Step-by-Step: Windows 10/11

1. DOWNLOAD
   → Go to: https://www.mongodb.com/try/download/community
   → Select: Windows, MSI package, latest version
   → Download the .msi file

2. INSTALL
   → Run the MSI installer
   → Choose "Complete" installation
   → ✅ Check "Install MongoDB as a Service"
       (This makes MongoDB start automatically with Windows)
   → ✅ Check "Install MongoDB Compass" 
       (Free GUI tool — we'll use it later)

3. VERIFY
   → Open PowerShell/CMD:
```

```powershell
# Check if mongod is running as a service
Get-Service MongoDB

# Check MongoDB version
mongod --version

# Should output something like:
# db version v7.0.x
# Build Info: ...
```

```
4. DEFAULT PATHS (Windows)
   ┌──────────────────────────────────────────────────┐
   │  Data Directory:  C:\Program Files\MongoDB\      │
   │                   Server\7.0\data\               │
   │  Log File:        C:\Program Files\MongoDB\      │
   │                   Server\7.0\log\mongod.log      │
   │  Config File:     C:\Program Files\MongoDB\      │
   │                   Server\7.0\bin\mongod.cfg      │
   │  Binary:          C:\Program Files\MongoDB\      │
   │                   Server\7.0\bin\mongod.exe      │
   └──────────────────────────────────────────────────┘

5. ADD TO PATH (if not auto-added)
   → Add to System Environment Variables:
   C:\Program Files\MongoDB\Server\7.0\bin
```

### 2.2 macOS Installation

```bash
# Using Homebrew (Recommended)

# Step 1: Tap the MongoDB Homebrew formula
brew tap mongodb/brew

# Step 2: Install MongoDB Community Edition
brew install mongodb-community@7.0

# Step 3: Start MongoDB
brew services start mongodb-community@7.0

# Step 4: Verify
mongod --version

# To stop MongoDB:
brew services stop mongodb-community@7.0

# Default paths (macOS):
# Data:    /usr/local/var/mongodb  (Intel)
#          /opt/homebrew/var/mongodb (Apple Silicon)
# Log:     /usr/local/var/log/mongodb/mongo.log
# Config:  /usr/local/etc/mongod.conf
```

### 2.3 Linux Installation (Ubuntu/Debian)

```bash
# Step 1: Import MongoDB public GPG key
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# Step 2: Create the list file
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Step 3: Reload packages & install
sudo apt-get update
sudo apt-get install -y mongodb-org

# Step 4: Start MongoDB
sudo systemctl start mongod

# Step 5: Enable auto-start on boot
sudo systemctl enable mongod

# Step 6: Verify
sudo systemctl status mongod
mongod --version

# Default paths (Linux):
# Data:    /var/lib/mongodb
# Log:     /var/log/mongodb/mongod.log
# Config:  /etc/mongod.conf
```

### 2.4 Linux Installation (RHEL/CentOS/Amazon Linux)

```bash
# Step 1: Create repo file
sudo tee /etc/yum.repos.d/mongodb-org-7.0.repo << 'EOF'
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-7.0.asc
EOF

# Step 2: Install
sudo yum install -y mongodb-org

# Step 3: Start & enable
sudo systemctl start mongod
sudo systemctl enable mongod

# Step 4: Verify
mongod --version
```

### 2.5 Docker Installation (Any OS — Quick & Clean)

```bash
# Pull the latest MongoDB image
docker pull mongo:7.0

# Run MongoDB container
docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -v mongodb_data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secret123 \
  mongo:7.0

# Connect to it
docker exec -it mongodb mongosh -u admin -p secret123

# Docker Compose (recommended for projects):
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: secret123
    restart: unless-stopped

volumes:
  mongodb_data:
```

```bash
# Start with:
docker-compose up -d

# Stop with:
docker-compose down
```

> 💡 **Pro Tip**: Docker is the fastest way to get MongoDB running. No system-level installation, easy cleanup. Perfect for development.

---

## 3. MongoDB Atlas — Cloud Setup (Zero Install)

> **Atlas is MongoDB's managed cloud service.** No installation, no maintenance, automatic backups, auto-scaling. This is what most teams use in production.

### 3.1 Setting Up Atlas (5-Minute Guide)

```
Step-by-Step:

1. GO TO: https://www.mongodb.com/cloud/atlas/register
   → Sign up (Google, GitHub, or email)

2. CREATE ORGANIZATION & PROJECT
   → Organization: "MyOrg" (or your company name)
   → Project: "MyProject"

3. BUILD A CLUSTER
   ┌──────────────────────────────────────────────────┐
   │  Choose a tier:                                   │
   │                                                   │
   │  🟢 M0 (Free Forever!)                           │
   │     → 512 MB storage                             │
   │     → Shared cluster                              │
   │     → Perfect for learning & small projects       │
   │                                                   │
   │  🟡 M10+ (Paid — starts ~$57/month)              │
   │     → Dedicated cluster                           │
   │     → More storage, better performance            │
   │                                                   │
   │  🔴 M30+ (Production)                            │
   │     → Full features, auto-scaling                 │
   │     → Backups, monitoring, alerts                 │
   └──────────────────────────────────────────────────┘
   
   → Select M0 FREE
   → Choose cloud provider: AWS / GCP / Azure
   → Choose region: closest to you (e.g., Mumbai, Virginia)
   → Click "Create Cluster" (takes ~3 minutes)

4. CONFIGURE ACCESS
   a) Database User:
      → Username: myAppUser
      → Password: (use the auto-generated strong password)
      → Role: Read and write to any database
   
   b) Network Access:
      → Add IP Address: "Allow Access from Anywhere" (for development)
         OR add your specific IP (for production)
      → This modifies the IP whitelist

5. GET CONNECTION STRING
   → Click "Connect" → "Connect your application"
   → Choose your driver (Node.js, Python, etc.)
   → Copy the connection string:
```

```
mongodb+srv://myAppUser:<password>@cluster0.abc123.mongodb.net/
           ↑               ↑                ↑
           │               │                │
        username       your password     Atlas host
```

> ⚠️ **Security Note**: Never hardcode passwords in your source code. Use environment variables.

---

## 4. mongosh — The Modern MongoDB Shell

> `mongosh` replaced the old `mongo` shell. It's a **full-featured JavaScript REPL** with syntax highlighting, auto-completion, and more.

### 4.1 Installing mongosh

```bash
# Windows (if not installed with MongoDB):
# Download from: https://www.mongodb.com/try/download/shell
# Or via winget:
winget install MongoDB.Shell

# macOS:
brew install mongosh

# Linux (Ubuntu/Debian):
sudo apt-get install -y mongodb-mongosh

# Verify:
mongosh --version
```

### 4.2 Connecting to MongoDB

```bash
# Connect to local MongoDB (default: localhost:27017)
mongosh

# Connect to specific host/port
mongosh "mongodb://localhost:27017"

# Connect with authentication
mongosh "mongodb://admin:password@localhost:27017/admin"

# Connect to Atlas
mongosh "mongodb+srv://myUser:myPass@cluster0.abc123.mongodb.net/mydb"

# Connect to specific database
mongosh "mongodb://localhost:27017/ecommerce"
```

### 4.3 Essential mongosh Commands — Your First 20 Minutes

```javascript
// ═══════════════════════════════════════════════════════
//  DATABASE COMMANDS
// ═══════════════════════════════════════════════════════

// Show all databases
show dbs
// Output:
// admin    40.00 KiB
// config   72.00 KiB
// local    40.00 KiB

// Create / switch to a database
use ecommerce
// Note: Database is created only when you first insert data

// Show current database
db
// Output: ecommerce

// Drop (delete) a database
db.dropDatabase()    // ⚠️ Careful! This is permanent!

// ═══════════════════════════════════════════════════════
//  COLLECTION COMMANDS
// ═══════════════════════════════════════════════════════

// Show all collections in current database
show collections

// Create a collection explicitly
db.createCollection("users")

// Or just start inserting — MongoDB creates it automatically!
db.products.insertOne({ name: "iPhone" })  // Creates "products" collection

// Drop a collection
db.users.drop()    // ⚠️ Careful!

// ═══════════════════════════════════════════════════════
//  QUICK CRUD (Preview — Full guide in Chapter 3B.3)
// ═══════════════════════════════════════════════════════

// Insert a document
db.users.insertOne({
  name: "Ritesh",
  age: 28,
  city: "Delhi"
})

// Find all documents
db.users.find()

// Find with condition
db.users.find({ age: { $gt: 25 } })

// Pretty print results
db.users.find().pretty()

// Count documents
db.users.countDocuments()

// ═══════════════════════════════════════════════════════
//  HELP & UTILITY
// ═══════════════════════════════════════════════════════

// Get help
help

// See methods available on a collection
db.users.help()

// Clear the screen
cls

// Exit mongosh
exit
// Or press Ctrl+C twice
```

### 4.4 mongosh Pro Features

```javascript
// ── Auto-Complete ──
// Type db.us<TAB> → auto-completes to db.users

// ── Multi-Line Editing ──
// Just press Enter to continue on next line:
db.users.find({
  age: { $gt: 25 },
  city: "Delhi"
})

// ── Command History ──
// Press ↑/↓ arrows to cycle through previous commands

// ── Inline Help ──
// Type a method name without () to see its signature:
db.users.insertOne
// Shows: [Function: insertOne]

// ── Config ──
config.set("inspectDepth", 10)     // Show deeper nested objects
config.set("historyLength", 5000)  // Keep more history

// ── Snippets ──
// Load and run a JavaScript file:
load("/path/to/script.js")

// ── Output Formatting ──
// mongosh uses EJSON by default (Extended JSON)
// Use .toArray() for array output:
db.users.find().toArray()
```

---

## 5. MongoDB Compass — The Visual GUI

> **Compass** is MongoDB's official GUI. Think of it as the "SSMS" or "pgAdmin" for MongoDB.

### 5.1 What Compass Can Do

```
┌──────────────────────────────────────────────────────────────┐
│                   MongoDB Compass Features                    │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  📊 VISUAL DATA EXPLORER                                     │
│     → Browse collections, view documents as JSON/Table/List   │
│     → Edit documents inline (click & edit)                    │
│     → Drag & drop field reordering                            │
│                                                               │
│  🔍 QUERY BUILDER                                            │
│     → Visual query builder (no code needed!)                  │
│     → Syntax highlighting & validation                        │
│     → Save favorite queries                                   │
│                                                               │
│  📈 AGGREGATION PIPELINE BUILDER                             │
│     → Build pipelines stage by stage                          │
│     → Preview output at each stage                            │
│     → Export pipeline code to your language                    │
│                                                               │
│  📉 PERFORMANCE INSIGHTS                                     │
│     → Real-time performance stats                             │
│     → explain() plan visualization                            │
│     → Index suggestions                                       │
│                                                               │
│  📐 SCHEMA ANALYZER                                          │
│     → Auto-detect schema from documents                       │
│     → See field types, frequencies, distributions             │
│     → Find schema inconsistencies                             │
│                                                               │
│  ✅ VALIDATION RULES                                         │
│     → Create/edit JSON Schema validators visually             │
│                                                               │
│  🔑 INDEX MANAGEMENT                                         │
│     → View existing indexes                                   │
│     → Create/drop indexes via GUI                             │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 Connecting Compass

```
1. Open MongoDB Compass

2. Paste your connection string:
   ┌───────────────────────────────────────────────────────────┐
   │  New Connection                                           │
   │                                                           │
   │  URI: mongodb://localhost:27017                           │
   │       OR                                                  │
   │  URI: mongodb+srv://user:pass@cluster0.abc.mongodb.net   │
   │                                                           │
   │  [ Advanced Options ]                                     │
   │    → Authentication                                       │
   │    → TLS/SSL                                              │
   │    → SSH Tunnel                                           │
   │    → Read Preference                                      │
   │                                                           │
   │  [  Connect  ]                                            │
   └───────────────────────────────────────────────────────────┘

3. You'll see:
   → List of databases on the left
   → Click a database → see collections
   → Click a collection → see documents
```

> 💡 **Pro Tip**: Compass is **amazing for building aggregation pipelines**. You can build them visually, see the output at each stage, and then export the pipeline code for your app. We'll use this heavily in Chapter 3B.4.

---

## 6. Connection Strings — The Universal Key

### 6.1 Standard Connection String Format

```
mongodb://[username:password@]host1[:port1][,...hostN[:portN]][/database][?options]

Examples:

# Local, no auth
mongodb://localhost:27017

# Local, with auth
mongodb://admin:secret123@localhost:27017/admin

# Remote server
mongodb://dbuser:pass@db.mycompany.com:27017/myapp

# Replica Set
mongodb://host1:27017,host2:27017,host3:27017/mydb?replicaSet=myRS

# Atlas (SRV format — DNS-based)
mongodb+srv://user:pass@cluster0.abc123.mongodb.net/mydb?retryWrites=true&w=majority
```

### 6.2 Important Connection Options

```
┌────────────────────────┬──────────────────────────────────────┐
│  Option                │  Purpose                             │
├────────────────────────┼──────────────────────────────────────┤
│  retryWrites=true      │  Auto-retry failed writes            │
│  w=majority            │  Write concern (majority = safe)     │
│  readPreference=       │  Where to read from in replica set   │
│    secondaryPreferred  │                                      │
│  maxPoolSize=50        │  Max connection pool size             │
│  connectTimeoutMS=     │  Connection timeout                  │
│    10000               │                                      │
│  authSource=admin      │  Which DB has the user credentials   │
│  ssl=true              │  Enable TLS/SSL encryption           │
│  replicaSet=myRS       │  Name of replica set to connect to   │
│  appName=myApp         │  Identify your app in server logs    │
└────────────────────────┴──────────────────────────────────────┘
```

### 6.3 Connecting from Different Languages

```javascript
// ── Node.js (using official driver) ──
const { MongoClient } = require('mongodb');

const uri = "mongodb+srv://user:pass@cluster0.abc123.mongodb.net/";
const client = new MongoClient(uri);

async function main() {
  await client.connect();
  console.log("Connected to MongoDB!");
  
  const db = client.db("ecommerce");
  const users = db.collection("users");
  
  // Insert
  await users.insertOne({ name: "Ritesh", age: 28 });
  
  // Find
  const user = await users.findOne({ name: "Ritesh" });
  console.log(user);
  
  await client.close();
}

main().catch(console.error);
```

```python
# ── Python (using pymongo) ──
from pymongo import MongoClient

uri = "mongodb+srv://user:pass@cluster0.abc123.mongodb.net/"
client = MongoClient(uri)

db = client["ecommerce"]
users = db["users"]

# Insert
users.insert_one({"name": "Ritesh", "age": 28})

# Find
user = users.find_one({"name": "Ritesh"})
print(user)

client.close()
```

```java
// ── Java (using official driver) ──
import com.mongodb.client.*;
import org.bson.Document;

MongoClient client = MongoClients.create(
    "mongodb+srv://user:pass@cluster0.abc123.mongodb.net/"
);
MongoDatabase db = client.getDatabase("ecommerce");
MongoCollection<Document> users = db.getCollection("users");

// Insert
users.insertOne(new Document("name", "Ritesh").append("age", 28));

// Find
Document user = users.find(new Document("name", "Ritesh")).first();
System.out.println(user.toJson());

client.close();
```

```csharp
// ── C# (.NET — using official driver) ──
using MongoDB.Driver;
using MongoDB.Bson;

var client = new MongoClient(
    "mongodb+srv://user:pass@cluster0.abc123.mongodb.net/"
);
var db = client.GetDatabase("ecommerce");
var users = db.GetCollection<BsonDocument>("users");

// Insert
var doc = new BsonDocument { { "name", "Ritesh" }, { "age", 28 } };
await users.InsertOneAsync(doc);

// Find
var filter = Builders<BsonDocument>.Filter.Eq("name", "Ritesh");
var user = await users.Find(filter).FirstOrDefaultAsync();
Console.WriteLine(user);
```

---

## 7. MongoDB Configuration — The Config File

### 7.1 mongod.conf / mongod.cfg — Key Settings

```yaml
# ── /etc/mongod.conf (Linux) or mongod.cfg (Windows) ──

# Where data is stored
storage:
  dbPath: /var/lib/mongodb          # Data directory
  journal:
    enabled: true                    # Always keep this ON!
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2                 # WiredTiger cache (default: 50% RAM - 1GB)
    collectionConfig:
      blockCompressor: snappy        # snappy (default), zlib, zstd, none

# Network settings
net:
  port: 27017                        # Default port
  bindIp: 127.0.0.1                  # localhost only (safe default)
  # bindIp: 0.0.0.0                  # Listen on all interfaces (remote access)
  maxIncomingConnections: 65536

# Logging
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
  verbosity: 0                       # 0=Info, 1-5=Debug levels

# Security (ALWAYS enable in production!)
security:
  authorization: enabled             # Enforce auth
  # keyFile: /path/to/keyfile        # For replica set auth

# Replication
replication:
  replSetName: myReplicaSet          # Replica set name
  oplogSizeMB: 2048                  # Oplog size (default: 5% of disk)

# Profiling (performance monitoring)
operationProfiling:
  mode: slowOp                       # off, slowOp, all
  slowOpThresholdMs: 100             # Log queries slower than 100ms
```

### 7.2 Critical Production Settings

```
┌────────────────────────────────────────────────────────────────┐
│            Production Checklist                                 │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✅ MUST DO:                                                    │
│     □ Enable authentication (security.authorization: enabled)   │
│     □ Bind to specific IPs (not 0.0.0.0)                       │
│     □ Enable TLS/SSL for all connections                        │
│     □ Use replica set (minimum 3 nodes)                         │
│     □ Set WiredTiger cache to ~50% of available RAM             │
│     □ Enable slow query logging (operationProfiling)            │
│     □ Set up monitoring (Atlas, Ops Manager, or Prometheus)     │
│     □ Configure automated backups                               │
│     □ Use a dedicated data partition/volume                     │
│                                                                 │
│  ❌ NEVER DO:                                                   │
│     □ Run without authentication                                │
│     □ Expose port 27017 to the internet without firewall        │
│     □ Use root/admin credentials in application code            │
│     □ Disable journaling                                        │
│     □ Run on NFS or network-mounted storage                     │
│     □ Set cacheSizeGB to 100% of RAM (OS needs memory too!)     │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

> ⚠️ **Security Warning**: MongoDB without authentication is a **wide-open door**. Thousands of MongoDB instances have been hacked because of this. ALWAYS enable `security.authorization: enabled` in production!

---

## 8. Other MongoDB Tools You Should Know

```
┌──────────────────┬────────────────────────────────────────────┐
│  Tool            │  Purpose                                   │
├──────────────────┼────────────────────────────────────────────┤
│  mongosh         │  Interactive shell (JavaScript REPL)       │
│  Compass         │  Visual GUI for querying & management      │
│  mongodump       │  Export database to BSON files (backup)    │
│  mongorestore    │  Import BSON files (restore backup)        │
│  mongoexport     │  Export collection to JSON/CSV             │
│  mongoimport     │  Import JSON/CSV into a collection         │
│  mongostat       │  Real-time server statistics               │
│  mongotop        │  Track read/write time per collection      │
│  Atlas CLI       │  Manage Atlas from command line            │
└──────────────────┴────────────────────────────────────────────┘
```

### Quick Examples

```bash
# ── Backup a database ──
mongodump --db ecommerce --out /backup/2024-01-15/

# ── Restore a database ──
mongorestore --db ecommerce /backup/2024-01-15/ecommerce/

# ── Export a collection to JSON ──
mongoexport --db ecommerce --collection users --out users.json

# ── Import from JSON ──
mongoimport --db ecommerce --collection users --file users.json

# ── Import from CSV ──
mongoimport --db ecommerce --collection products \
  --type csv --headerline --file products.csv

# ── Real-time stats ──
mongostat --rowcount 10    # Show 10 snapshots

# ── Track time per collection ──
mongotop 5                 # Update every 5 seconds
```

---

## 9. Verify Your Setup — Quick Smoke Test

Run this in `mongosh` to verify everything works:

```javascript
// ═══════════════════════════════════════════════════════
//  SMOKE TEST — Run after installation
// ═══════════════════════════════════════════════════════

// 1. Check server status
db.serverStatus().version
// Expected: "7.0.x" (or your installed version)

// 2. Create a test database
use testDB

// 3. Insert a document
db.smokeTest.insertOne({
  message: "MongoDB is working!",
  timestamp: new Date(),
  status: "success"
})

// 4. Read it back
db.smokeTest.findOne()
// Should return your document with an auto-generated _id

// 5. Check write concern
db.smokeTest.insertOne(
  { test: "write concern" },
  { writeConcern: { w: 1, j: true } }
)

// 6. Check WiredTiger stats
db.serverStatus().wiredTiger.cache["bytes currently in the cache"]

// 7. Clean up
db.smokeTest.drop()
use admin
db.dropDatabase()  // Drop testDB — only if you want to clean up

// 8. Final check
print("✅ MongoDB is installed and working correctly!")
print("Server version: " + db.version())
print("Shell version: " + version())
```

Expected output:
```
MongoDB is installed and working correctly!
Server version: 7.0.x
Shell version: 2.x.x
```

---

## 🧠 Key Takeaways

```
┌──────────────────────────────────────────────────────────────┐
│                   CHAPTER 3B.2 SUMMARY                        │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Three editions: Community (free), Enterprise, Atlas       │
│                                                               │
│  2. Atlas Free Tier = best way to start (zero install)        │
│                                                               │
│  3. mongosh = modern shell with JS REPL, auto-complete        │
│                                                               │
│  4. Compass = visual GUI for querying & pipeline building     │
│                                                               │
│  5. Connection strings: mongodb:// or mongodb+srv://          │
│                                                               │
│  6. Config file: storage, network, security, replication      │
│                                                               │
│  7. ALWAYS enable authentication in production!               │
│                                                               │
│  8. mongodump/mongorestore for backup, mongoexport/import     │
│     for JSON/CSV                                              │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔗 What's Next?

| Next Chapter | Topic |
|-------------|-------|
| **[3B.3 — MongoDB CRUD — The Complete Guide](./03-MongoDB-CRUD.md)** | Master every read and write operation MongoDB offers |

---

> **"The best database is the one you can actually connect to. Now you can. Let's start writing queries."**
