# 🔐 Chapter 3B.8 — MongoDB Transactions & Security

> **Level:** 🔴 Advanced
> **Time to Master:** ~4-5 hours
> **Prerequisites:** Chapter 3B.5 (Indexing), Chapter 3B.7 (Replication & Sharding), Chapter 1.5 (ACID vs BASE)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand MongoDB's **single-document atomicity** and why it covers 90% of use cases
- Implement **multi-document ACID transactions** when you truly need them
- Configure **Read/Write Concerns** for precise consistency guarantees
- Set up **Role-Based Access Control (RBAC)** with built-in and custom roles
- Configure **authentication** (SCRAM, x.509, LDAP, Kerberos)
- Enable **encryption at rest** and **in transit** (TLS/SSL)
- Implement **Client-Side Field Level Encryption (CSFLE)** for the most sensitive data
- Set up **auditing** for compliance (SOX, HIPAA, PCI-DSS, GDPR)

---

## ⚛️ Part 1: Transactions in MongoDB

### Single-Document Atomicity — The Default Guarantee

MongoDB guarantees that **operations on a single document are always atomic**. This is the foundation.

```javascript
// This update is FULLY ATOMIC — no partial writes possible
db.accounts.updateOne(
  { _id: "account_123" },
  {
    $inc: { balance: -500 },                         // Debit
    $push: { 
      transactions: {
        type: "withdrawal",
        amount: 500,
        date: new Date(),
        description: "ATM Withdrawal"
      }
    },
    $set: { lastActivity: new Date() }
  }
)

// Even though this modifies 3 fields simultaneously:
// ✅ Either ALL changes apply, or NONE do
// ✅ No other operation can see a "half-updated" document
// ✅ This is true even for embedded arrays and nested objects
```

> 💡 **Key Insight:** Because MongoDB lets you embed related data in one document, many operations that would require multi-row transactions in SQL are handled as single-document atomic operations in MongoDB. **Design your schema to leverage this!**

```
    SQL World (requires transaction):      MongoDB World (single doc = atomic):
    ──────────────────────────              ──────────────────────────────────
    BEGIN TRANSACTION;                      // One document = one atomic op
    UPDATE accounts                        db.accounts.updateOne(
      SET balance = balance - 500            { _id: "acc_123" },
      WHERE id = 'acc_123';                  {
    INSERT INTO transactions                   $inc: { balance: -500 },
      (account_id, amount, type)               $push: { transactions: {
      VALUES ('acc_123', 500, 'debit');            amount: 500, type: "debit"
    UPDATE account_stats                       }},
      SET last_activity = NOW()                $set: { lastActivity: new Date() }
      WHERE account_id = 'acc_123';          }
    COMMIT;                                )
                                           // All atomic — no transaction needed!
    3 tables, 1 transaction                1 document, 0 transactions
```

---

### Multi-Document Transactions — When You Need Them

> Available since: **MongoDB 4.0** (replica sets) and **4.2** (sharded clusters)

Sometimes you **must** modify multiple documents atomically. Classic example: transferring money between two different accounts.

```javascript
// ═══════════════════════════════════════════════════════════
// Multi-Document Transaction: Bank Transfer
// ═══════════════════════════════════════════════════════════

const session = db.getMongo().startSession();

session.startTransaction({
  readConcern: { level: "snapshot" },           // Point-in-time read
  writeConcern: { w: "majority" },              // Durable writes
  readPreference: "primary"                     // Must read from primary
});

try {
  const accountsColl = session.getDatabase("bank").accounts;
  
  // Step 1: Debit from source account
  const debitResult = accountsColl.updateOne(
    { _id: "alice_account", balance: { $gte: 500 } },  // Check sufficient funds!
    { $inc: { balance: -500 } },
    { session }     // ← Pass the session to EVERY operation!
  );
  
  if (debitResult.modifiedCount === 0) {
    throw new Error("Insufficient funds or account not found");
  }
  
  // Step 2: Credit to destination account
  accountsColl.updateOne(
    { _id: "bob_account" },
    { $inc: { balance: 500 } },
    { session }
  );
  
  // Step 3: Record the transfer
  session.getDatabase("bank").transfers.insertOne(
    {
      from: "alice_account",
      to: "bob_account",
      amount: 500,
      date: new Date(),
      status: "completed"
    },
    { session }
  );
  
  // All good — commit!
  session.commitTransaction();
  print("Transfer successful! ✅");
  
} catch (error) {
  // Something went wrong — abort!
  session.abortTransaction();
  print("Transfer failed, rolled back: " + error.message + " ❌");
  
} finally {
  session.endSession();
}
```

### Transaction Execution Flow

```
    ┌───────────────────────────────────────────────────────────────┐
    │                 Transaction Lifecycle                         │
    │                                                               │
    │  1. startSession()                                           │
    │     ↓                                                        │
    │  2. startTransaction()                                       │
    │     ↓                                                        │
    │  3. Perform operations (passing session to each)             │
    │     • Read operations see a SNAPSHOT (point-in-time)         │
    │     • Write operations are BUFFERED                          │
    │     ↓                                                        │
    │  4a. commitTransaction()                                     │
    │      • All writes applied atomically                         │
    │      • Other operations can now see the changes              │
    │      ↓                                                       │
    │  4b. abortTransaction() (if error)                           │
    │      • All writes discarded                                  │
    │      • As if nothing happened                                │
    │     ↓                                                        │
    │  5. endSession()                                             │
    │                                                               │
    │  ⏱️ Default timeout: 60 seconds                               │
    │     If not committed/aborted within 60s → auto-abort!         │
    └───────────────────────────────────────────────────────────────┘
```

---

### Transaction with Retry Logic (Production Pattern)

```javascript
// ═══════════════════════════════════════════════════════════
// Production-grade transaction with retry
// ═══════════════════════════════════════════════════════════

async function runTransactionWithRetry(session, txnFunc) {
  while (true) {
    try {
      await txnFunc(session);
      break;  // Success — exit retry loop
    } catch (error) {
      // If a transient error, retry the ENTIRE transaction
      if (error.hasOwnProperty("errorLabels") && 
          error.errorLabels.includes("TransientTransactionError")) {
        print("TransientTransactionError, retrying transaction...");
        continue;  // Retry
      }
      throw error;  // Non-transient error — don't retry
    }
  }
}

async function commitWithRetry(session) {
  while (true) {
    try {
      await session.commitTransaction();
      print("Transaction committed. ✅");
      break;
    } catch (error) {
      if (error.hasOwnProperty("errorLabels") &&
          error.errorLabels.includes("UnknownTransactionCommitResult")) {
        print("UnknownTransactionCommitResult, retrying commit...");
        continue;  // Safe to retry commit
      }
      throw error;
    }
  }
}

// Usage:
const session = client.startSession();

try {
  await runTransactionWithRetry(session, async (session) => {
    session.startTransaction({
      readConcern: { level: "snapshot" },
      writeConcern: { w: "majority" }
    });
    
    // ... your transaction operations here ...
    
    await commitWithRetry(session);
  });
} finally {
  await session.endSession();
}
```

### Transaction Error Labels

```
    ┌─────────────────────────────────┬──────────────────────────────────┐
    │ Error Label                     │ What To Do                       │
    ├─────────────────────────────────┼──────────────────────────────────┤
    │ TransientTransactionError       │ Retry the ENTIRE transaction     │
    │ (write conflict, failover, etc)│ from startTransaction()          │
    │                                 │                                  │
    │ UnknownTransactionCommitResult  │ Retry ONLY the commit            │
    │ (network error during commit)   │ Transaction may have committed   │
    │                                 │                                  │
    │ No error label                  │ Application error — handle it    │
    │                                 │ Don't retry automatically        │
    └─────────────────────────────────┴──────────────────────────────────┘
```

---

### Transaction Rules & Limitations

```
    ┌─────────────────────────────────────────────────────────────────────┐
    │              TRANSACTION RULES TO REMEMBER                          │
    │                                                                     │
    │  ✅ CAN do inside a transaction:                                    │
    │     • CRUD on existing collections                                  │
    │     • Operations across multiple collections                        │
    │     • Operations across multiple databases (4.2+)                   │
    │     • Operations across shards (4.2+)                               │
    │                                                                     │
    │  ❌ CANNOT do inside a transaction:                                  │
    │     • Create/drop collections or indexes                            │
    │     • Affect non-CRUD operations (e.g., createUser)                 │
    │     • Write to system.* collections                                 │
    │     • Use $out or $merge in aggregation pipeline                    │
    │     • Run getMore with a cursor created outside the transaction     │
    │                                                                     │
    │  ⚠️ Limits:                                                         │
    │     • Default timeout: 60 seconds (configurable)                    │
    │     • Transaction oplog entry: max 16MB                             │
    │     • Transactions add ~5-10% overhead                              │
    │     • Long-running transactions hold locks → avoid!                 │
    │     • If you need transactions for everything → wrong DB choice?    │
    │                                                                     │
    │  💡 Best practice:                                                   │
    │     • Keep transactions SHORT (< 1 second)                          │
    │     • Touch as FEW documents as possible                            │
    │     • Use single-doc atomicity when possible (80%+ of cases)        │
    │     • Reserve multi-doc transactions for TRUE cross-document needs  │
    └─────────────────────────────────────────────────────────────────────┘
```

---

### Read/Write Concern + Read Preference = Causal Consistency

> Combine all three to get **causal consistency** — "read your own writes" guarantee.

```javascript
// Causal consistency: If I write something, my next read SEES that write
// Even if reading from a secondary!

const session = db.getMongo().startSession({ causalConsistency: true });

const coll = session.getDatabase("mydb").getCollection("orders");

// Write to primary
coll.insertOne(
  { _id: "order_new", status: "placed" },
  { writeConcern: { w: "majority" } }
);

// Read from secondary — BUT guaranteed to see the write above!
coll.find(
  { _id: "order_new" },
  { readPreference: "secondary", readConcern: { level: "majority" } }
);
// Returns the order even though reading from secondary ✅

session.endSession();
```

```
    Without Causal Consistency:
    ┌─────────────┐     ┌─────────────┐
    │   PRIMARY    │     │  SECONDARY  │
    │              │     │             │
    │ INSERT order │     │             │
    │   ✅ Done     │────►│ (replicating│
    │              │     │  ...lag...) │
    │              │     │             │
    └─────────────┘     └─────────────┘
                              │
    Client reads from secondary ─────► ❌ Order not there yet!
    (replication lag)

    With Causal Consistency:
    Client reads from secondary ─────► ✅ Waits until secondary
                                          has caught up to this
                                          operation time
```

---

## 🛡️ Part 2: MongoDB Security

### Security Checklist Overview

```
    ┌─────────────────────────────────────────────────────────────────────┐
    │               MONGODB SECURITY LAYERS                               │
    │                                                                     │
    │   Layer 1: AUTHENTICATION        "Who are you?"                     │
    │   Layer 2: AUTHORIZATION (RBAC)  "What are you allowed to do?"      │
    │   Layer 3: ENCRYPTION IN TRANSIT "Is the connection secure?"        │
    │   Layer 4: ENCRYPTION AT REST    "Is stored data encrypted?"        │
    │   Layer 5: FIELD-LEVEL ENCRYPTION "Can even DBAs read this field?" │
    │   Layer 6: AUDITING              "What did everyone do?"            │
    │   Layer 7: NETWORK SECURITY      "Who can even connect?"           │
    └─────────────────────────────────────────────────────────────────────┘
```

---

### Layer 1: Authentication — "Who Are You?"

```javascript
// ═══════════════════════════════════════════════════════════
// Enable authentication (mongod.conf)
// ═══════════════════════════════════════════════════════════

// mongod.conf:
// security:
//   authorization: enabled

// ⚠️ Without this, ANYONE can connect and do ANYTHING!
// This is the #1 MongoDB security mistake.

// ═══════════════════════════════════════════════════════════
// Create the first admin user (do this BEFORE enabling auth!)
// ═══════════════════════════════════════════════════════════
use admin

db.createUser({
  user: "superAdmin",
  pwd: passwordPrompt(),     // Prompts for password (never hardcode!)
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "clusterAdmin", db: "admin" }
  ]
})

// ═══════════════════════════════════════════════════════════
// Create application-specific users
// ═══════════════════════════════════════════════════════════
use myAppDB

// Read-write user for the application
db.createUser({
  user: "app_user",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "myAppDB" }
  ]
})

// Read-only user for reporting/analytics
db.createUser({
  user: "report_user",
  pwd: passwordPrompt(),
  roles: [
    { role: "read", db: "myAppDB" }
  ]
})

// Connect with authentication
// mongosh "mongodb://app_user:password@host:27017/myAppDB"
```

### Authentication Mechanisms

```
┌──────────────────────┬───────────────────────────────────────────────────┐
│ Mechanism            │ Details                                           │
├──────────────────────┼───────────────────────────────────────────────────┤
│ SCRAM-SHA-256        │ Default. Username/password with salted challenge. │
│ (Default)            │ Good for most cases. MongoDB 4.0+                │
│                      │                                                   │
│ SCRAM-SHA-1          │ Older default. Still supported.                   │
│                      │                                                   │
│ x.509 Certificates  │ Certificate-based auth. Used for:                 │
│                      │ • Replica set member authentication               │
│                      │ • Client authentication (no passwords!)           │
│                      │                                                   │
│ LDAP (Enterprise)    │ Authenticate against Active Directory / LDAP.     │
│                      │ Centralized user management.                      │
│                      │                                                   │
│ Kerberos (Enterprise)│ Enterprise SSO. Used in Windows/AD environments. │
│                      │                                                   │
│ AWS IAM (Atlas)      │ Use AWS IAM roles for authentication.             │
└──────────────────────┴───────────────────────────────────────────────────┘
```

---

### Layer 2: Authorization (RBAC) — "What Can You Do?"

```javascript
// MongoDB's Role-Based Access Control (RBAC):
// Users → assigned → Roles → grant → Privileges → on → Resources

// ═══════════════════════════════════════════════════════════
// Built-in Roles
// ═══════════════════════════════════════════════════════════
```

```
    ┌──────────────────────────────────────────────────────────────────┐
    │                    BUILT-IN ROLES                                 │
    │                                                                  │
    │  Database-Level:                                                 │
    │  ├── read              → Find, aggregate, listCollections        │
    │  ├── readWrite         → read + insert, update, delete, createIdx│
    │  ├── dbAdmin           → Schema operations, stats, reIndex       │
    │  ├── dbOwner           → readWrite + dbAdmin + userAdmin         │
    │  └── userAdmin         → Create/modify users and roles           │
    │                                                                  │
    │  All-Database:                                                   │
    │  ├── readAnyDatabase   → read on ALL databases                   │
    │  ├── readWriteAnyDatabase → readWrite on ALL databases           │
    │  ├── dbAdminAnyDatabase   → dbAdmin on ALL databases             │
    │  └── userAdminAnyDatabase → userAdmin on ALL databases           │
    │                                                                  │
    │  Cluster-Level:                                                  │
    │  ├── clusterAdmin      → Manage sharding, replica sets           │
    │  ├── clusterManager    → Monitor + manage cluster                │
    │  ├── clusterMonitor    → Read-only cluster monitoring             │
    │  └── hostManager       → Manage servers                          │
    │                                                                  │
    │  Superuser:                                                      │
    │  └── root              → EVERYTHING (use sparingly!)             │
    │                                                                  │
    │  Backup/Restore:                                                 │
    │  ├── backup            → mongodump, fsync+lock                   │
    │  └── restore           → mongorestore                            │
    └──────────────────────────────────────────────────────────────────┘
```

### Custom Roles

```javascript
// Create a custom role with fine-grained permissions
use admin

db.createRole({
  role: "orderProcessor",
  privileges: [
    {
      // Can read and update orders, but NOT delete or insert
      resource: { db: "myAppDB", collection: "orders" },
      actions: ["find", "update"]
    },
    {
      // Can read products (but nothing else)
      resource: { db: "myAppDB", collection: "products" },
      actions: ["find"]
    },
    {
      // Can insert into audit_log
      resource: { db: "myAppDB", collection: "audit_log" },
      actions: ["insert"]
    }
  ],
  roles: []  // No inherited roles
})

// Assign custom role to user
db.createUser({
  user: "order_worker",
  pwd: passwordPrompt(),
  roles: [
    { role: "orderProcessor", db: "admin" }
  ]
})

// This user can:
// ✅ db.orders.find({ status: "pending" })
// ✅ db.orders.updateOne({ _id: "o1" }, { $set: { status: "shipped" } })
// ✅ db.products.find({ _id: "p1" })
// ✅ db.audit_log.insertOne({ event: "order_processed" })
// ❌ db.orders.insertOne({ ... })          → DENIED!
// ❌ db.orders.deleteOne({ ... })          → DENIED!
// ❌ db.users.find({ ... })               → DENIED!
```

### Available Actions (Privileges) — Quick Reference

```
    ┌───────────────┬──────────────────────────────────────────────────┐
    │ Category      │ Actions                                          │
    ├───────────────┼──────────────────────────────────────────────────┤
    │ Query & Write │ find, insert, update, remove                     │
    │ Index         │ createIndex, dropIndex, listIndexes              │
    │ Collection    │ createCollection, dropCollection, collStats      │
    │ Database      │ dropDatabase, listCollections                    │
    │ Diagnostic    │ dbStats, serverStatus, collStats, top            │
    │ Replication   │ replSetGetConfig, replSetGetStatus               │
    │ Sharding      │ addShard, removeShard, enableSharding            │
    │ User Mgmt     │ createUser, dropUser, grantRole, revokeRole     │
    └───────────────┴──────────────────────────────────────────────────┘
```

---

### Layer 3: Encryption in Transit (TLS/SSL)

```yaml
# mongod.conf — Enable TLS
net:
  tls:
    mode: requireTLS                        # Require all connections use TLS
    certificateKeyFile: /etc/ssl/mongodb.pem  # Server certificate + key
    CAFile: /etc/ssl/ca.pem                  # Certificate Authority
    allowConnectionsWithoutCertificates: false  # Require client certs too!
```

```javascript
// Connect with TLS
// mongosh "mongodb://host:27017/mydb" --tls --tlsCAFile /etc/ssl/ca.pem

// Connection string with TLS
// mongodb://user:pass@host:27017/mydb?tls=true&tlsCAFile=/etc/ssl/ca.pem
```

```
    Without TLS:                        With TLS:
    ┌────────┐   plain text   ┌────────┐    ┌────────┐  encrypted  ┌────────┐
    │ Client │───────────────►│MongoDB │    │ Client │─────🔒────►│MongoDB │
    └────────┘  passwords     └────────┘    └────────┘  cert-based └────────┘
                visible!                                 verified!
                Wireshark                                Can't snoop
                can read! 😱                              ✅
```

---

### Layer 4: Encryption at Rest

```yaml
# mongod.conf — Encrypt data files on disk
security:
  enableEncryption: true
  encryptionCipherMode: AES256-CBC     # or AES256-GCM (recommended)
  encryptionKeyFile: /etc/ssl/mongodb-keyfile
  # OR use a Key Management System (KMS):
  # kmip:
  #   serverName: kmip-server.example.com
  #   port: 5696
  #   clientCertificateFile: /etc/ssl/kmip-client.pem
```

```
    Without Encryption at Rest:           With Encryption at Rest:
    ┌──────────────────────────┐          ┌──────────────────────────┐
    │ Disk File (data/db)      │          │ Disk File (data/db)      │
    │                          │          │                          │
    │ {"name": "John Doe",     │          │ 8f2a1b3c4d5e6f7a8b9c... │
    │  "ssn": "123-45-6789",   │          │ encrypted binary data    │
    │  "salary": 95000}        │          │ 7d4e5f6a7b8c9d0e1f2a... │
    │                          │          │                          │
    │ Stolen disk = all data   │          │ Stolen disk = useless    │
    │ exposed! 😱               │          │ without the key ✅        │
    └──────────────────────────┘          └──────────────────────────┘
```

---

### Layer 5: Client-Side Field Level Encryption (CSFLE)

> **The gold standard** — data is encrypted **before** it reaches the server. Even DBAs and root users **cannot** read encrypted fields!

```javascript
// ═══════════════════════════════════════════════════════════
// CSFLE: Client-Side Field Level Encryption
// Available in MongoDB 4.2+ (Enterprise & Atlas)
// ═══════════════════════════════════════════════════════════

// How it works:
// 1. Application encrypts sensitive fields BEFORE sending to MongoDB
// 2. MongoDB stores encrypted binary data — it NEVER sees the plaintext
// 3. Application decrypts when reading back

// Two modes:
// • Automatic Encryption: Driver handles encrypt/decrypt transparently
// • Explicit Encryption: You manually encrypt/decrypt in code
```

```
    CSFLE Architecture:

    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                 │
    │  Application                           MongoDB Server           │
    │  ┌──────────────────────────┐          ┌──────────────────────┐ │
    │  │                          │          │                      │ │
    │  │ { name: "John Doe",     │  ──────► │ { name: "John Doe",  │ │
    │  │   ssn: "123-45-6789" }  │          │   ssn: BinData(6,    │ │
    │  │          │               │          │    "Ax3f7...encrypted│ │
    │  │          ▼               │          │     ...") }          │ │
    │  │   Driver encrypts SSN   │          │                      │ │
    │  │   using local key       │          │  Server stores binary│ │
    │  │   before sending        │          │  CANNOT decrypt it!  │ │
    │  └──────────────────────────┘          └──────────────────────┘ │
    │                                                                 │
    │  Even a compromised server or malicious DBA cannot read SSN!   │
    └─────────────────────────────────────────────────────────────────┘
```

### Encryption Schema (Automatic Encryption)

```javascript
// Define which fields to encrypt and how
const encryptedFieldsMap = {
  "mydb.patients": {
    fields: [
      {
        path: "ssn",
        bsonType: "string",
        keyId: UUID("..."),           // Data Encryption Key
        queries: { queryType: "equality" }  // Can query on encrypted field!
      },
      {
        path: "medicalRecords",
        bsonType: "array",
        keyId: UUID("...")
        // No queries — this field is just stored encrypted
      },
      {
        path: "insurance.policyNumber",
        bsonType: "string",
        keyId: UUID("..."),
        queries: { queryType: "equality" }
      }
    ]
  }
};

// The driver automatically encrypts/decrypts these fields
// Application code looks normal:
db.patients.insertOne({
  name: "John Doe",           // Stored in plaintext
  ssn: "123-45-6789",         // Auto-encrypted before sending!
  age: 35,                    // Stored in plaintext
  medicalRecords: [...]       // Auto-encrypted before sending!
});

// You can even QUERY on encrypted fields (Queryable Encryption, 7.0+):
db.patients.find({ ssn: "123-45-6789" });
// Driver encrypts the query value → server matches encrypted blobs
// Server NEVER sees "123-45-6789" in plaintext!
```

### Deterministic vs Random Encryption

```
    ┌──────────────────────────────────────────────────────────────────┐
    │                   ENCRYPTION ALGORITHMS                          │
    │                                                                  │
    │  Deterministic:                                                  │
    │    Same plaintext → same ciphertext (always)                     │
    │    ✅ Can do equality queries (find by SSN)                      │
    │    ❌ Less secure (pattern analysis possible)                    │
    │    Use for: SSN, email, policy numbers                           │
    │                                                                  │
    │  Random:                                                         │
    │    Same plaintext → different ciphertext (every time)            │
    │    ❌ Cannot query on this field                                 │
    │    ✅ Most secure (no pattern analysis)                          │
    │    Use for: Medical records, credit card numbers, notes          │
    └──────────────────────────────────────────────────────────────────┘
```

### Key Management for CSFLE

```
    ┌─────────────────────────────────────────────────────────────────────┐
    │                    KEY HIERARCHY                                     │
    │                                                                     │
    │   Customer Master Key (CMK)     ← Stored in KMS (AWS, Azure, GCP)  │
    │         │                                                           │
    │         │ encrypts                                                  │
    │         ▼                                                           │
    │   Data Encryption Key (DEK)     ← Stored in MongoDB key vault      │
    │         │                         (encrypted by CMK)                │
    │         │ encrypts                                                  │
    │         ▼                                                           │
    │   Your Data Fields              ← SSN, credit card, medical records│
    │                                                                     │
    │   KMS Options:                                                      │
    │   • AWS KMS                                                         │
    │   • Azure Key Vault                                                 │
    │   • Google Cloud KMS                                                │
    │   • Local key file (dev only — NOT for production!)                 │
    └─────────────────────────────────────────────────────────────────────┘
```

---

### Layer 6: Auditing (Enterprise Feature)

```yaml
# mongod.conf — Enable audit logging
auditLog:
  destination: file                     # file, console, or syslog
  format: JSON                          # JSON or BSON
  path: /var/log/mongodb/audit.json
  filter: '{
    atype: {
      $in: [
        "authenticate",
        "createUser",
        "dropUser",
        "createCollection",
        "dropCollection",
        "dropDatabase",
        "insert",
        "update",
        "delete"
      ]
    }
  }'
```

```javascript
// Sample audit log entry:
{
  "atype": "authCheck",                   // Action type
  "ts": ISODate("2024-03-15T14:30:00Z"), // Timestamp
  "local": { "ip": "192.168.1.100", "port": 27017 },
  "remote": { "ip": "192.168.1.50", "port": 45678 },
  "users": [{ "user": "app_user", "db": "myAppDB" }],
  "roles": [{ "role": "readWrite", "db": "myAppDB" }],
  "param": {
    "command": "find",
    "ns": "myAppDB.orders"
  },
  "result": 0                            // 0 = success, 13 = unauthorized
}

// Compliance mappings:
// PCI-DSS Req 10: Track all access to cardholder data → ✅ Audit logging
// HIPAA: Track access to PHI → ✅ Audit + CSFLE
// GDPR: Data protection → ✅ CSFLE + Encryption at rest
// SOX: Financial data audit trail → ✅ Audit logging + Document versioning
```

---

### Layer 7: Network Security

```yaml
# mongod.conf — Network restrictions
net:
  port: 27017
  bindIp: 192.168.1.100,127.0.0.1   # Only listen on specific interfaces
  # bindIp: 0.0.0.0                  # ❌ NEVER in production! Listens on ALL interfaces

  # IP whitelist (MongoDB 3.6+)
  # Use firewall rules or cloud security groups instead
```

```
    Network Security Layers:

    ┌───────────────────────────────────────────────────────────────┐
    │  Internet                                                     │
    │     │                                                         │
    │     ▼                                                         │
    │  ┌──────────────────┐                                         │
    │  │   Firewall / SG  │  ← Only allow port 27017 from app IPs  │
    │  └────────┬─────────┘                                         │
    │           │                                                   │
    │  ┌────────▼─────────┐                                         │
    │  │    VPN / VPC     │  ← MongoDB should NEVER be on public    │
    │  │                  │    internet                               │
    │  └────────┬─────────┘                                         │
    │           │                                                   │
    │  ┌────────▼─────────┐                                         │
    │  │  bindIp restrict │  ← Only accept connections from known   │
    │  │                  │    IP addresses                           │
    │  └────────┬─────────┘                                         │
    │           │                                                   │
    │  ┌────────▼─────────┐                                         │
    │  │   TLS/SSL        │  ← Encrypt all traffic                  │
    │  └────────┬─────────┘                                         │
    │           │                                                   │
    │  ┌────────▼─────────┐                                         │
    │  │  Authentication  │  ← Verify identity                      │
    │  └────────┬─────────┘                                         │
    │           │                                                   │
    │  ┌────────▼─────────┐                                         │
    │  │  Authorization   │  ← Check permissions                    │
    │  └──────────────────┘                                         │
    └───────────────────────────────────────────────────────────────┘
```

---

## 🚨 MongoDB Security Anti-Patterns

```
    ┌────────────────────────────────────────────────────────────────────┐
    │                                                                    │
    │  ❌ #1: No authentication enabled                                  │
    │     Default MongoDB has NO auth! Anyone can connect.               │
    │     Fix: ALWAYS enable security.authorization                      │
    │                                                                    │
    │  ❌ #2: Using root/admin user for the application                  │
    │     "It's easier" → until someone drops the database               │
    │     Fix: Create specific users with minimal permissions            │
    │                                                                    │
    │  ❌ #3: MongoDB exposed to the internet                            │
    │     bindIp: 0.0.0.0 without firewall = instant compromise         │
    │     Fix: VPC/VPN, firewall, bindIp to private IPs only            │
    │                                                                    │
    │  ❌ #4: Default port 27017 on public internet                      │
    │     Bots scan this port 24/7 looking for unprotected MongoDB       │
    │     Fix: Change port AND use firewall AND use auth                 │
    │                                                                    │
    │  ❌ #5: Passwords in connection strings in code                    │
    │     mongodb://admin:SuperSecret123@host:27017                      │
    │     Fix: Use environment variables or secrets manager              │
    │                                                                    │
    │  ❌ #6: No TLS/SSL                                                 │
    │     Credentials sent in plaintext over the network                 │
    │     Fix: Enable TLS for ALL connections                            │
    │                                                                    │
    │  ❌ #7: Single user for all applications                           │
    │     Can't track who did what. One compromised app = all access     │
    │     Fix: One user per application/service with minimal roles       │
    └────────────────────────────────────────────────────────────────────┘
```

---

## 💡 Production Security Checklist

```
    ┌─────────────────────────────────────────────────────────────────────┐
    │            MONGODB SECURITY HARDENING CHECKLIST                     │
    │                                                                     │
    │  Authentication & Authorization:                                    │
    │  □ Enable security.authorization in mongod.conf                    │
    │  □ Create specific users with minimum required roles               │
    │  □ Use SCRAM-SHA-256 (or stronger: x.509, LDAP)                   │
    │  □ Rotate passwords regularly                                      │
    │  □ No root user for application connections                        │
    │  □ Custom roles for fine-grained access control                    │
    │                                                                     │
    │  Network:                                                           │
    │  □ Bind to private IP only (never 0.0.0.0)                         │
    │  □ Use VPC/VPN — no public internet exposure                       │
    │  □ Firewall allows only known application IPs                      │
    │  □ Change default port (27017) if on shared network                │
    │                                                                     │
    │  Encryption:                                                        │
    │  □ TLS/SSL for all client connections                              │
    │  □ TLS for replica set member communication                        │
    │  □ Encryption at rest enabled                                      │
    │  □ CSFLE for PII/sensitive fields (SSN, credit cards, health data) │
    │  □ Key management via KMS (AWS KMS, Azure Key Vault, etc.)         │
    │                                                                     │
    │  Monitoring & Auditing:                                             │
    │  □ Audit logging enabled for auth + CRUD events                    │
    │  □ Monitor for failed authentication attempts                      │
    │  □ Alert on unusual query patterns                                 │
    │  □ Regular security assessments                                    │
    │                                                                     │
    │  Operational:                                                        │
    │  □ Keep MongoDB updated (security patches!)                        │
    │  □ Connection strings use env vars / secrets manager               │
    │  □ Backup encryption enabled                                       │
    │  □ Test recovery procedures regularly                              │
    └─────────────────────────────────────────────────────────────────────┘
```

---

## 🧭 What's Next?

Your MongoDB is transaction-safe and hardened. Now let's explore the **managed cloud** that handles most of this for you:

**Next Chapter → [3B.9 MongoDB Atlas & Cloud](./09-MongoDB-Atlas.md)** 🟡🔥

> _"Security isn't a feature — it's a foundation. A database without security is just a public data dump."_

---

[← Previous: MongoDB Replication & Sharding](./07-MongoDB-Replication-Sharding.md) | [Index](../INDEX.md) | [Next: MongoDB Atlas & Cloud →](./09-MongoDB-Atlas.md)
