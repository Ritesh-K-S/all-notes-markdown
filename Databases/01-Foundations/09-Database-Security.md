# 🛡️ Chapter 1.9 — Database Security Fundamentals

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~3-4 hours
> **Prerequisites:** Chapter 1.8 (Transactions & Concurrency)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand the **3 pillars** of database security (Authentication, Authorization, Encryption)
- Know how **SQL Injection** works and how to **prevent** it in every language
- Implement **encryption at rest** and **in transit** across all major databases
- Design **role-based access control (RBAC)** like an enterprise architect
- Set up **audit logging** to track every data access

---

## 🧠 Why Database Security Matters — The Wake-Up Call

```
╔══════════════════════════════════════════════════════════════════╗
║  REAL-WORLD BREACHES CAUSED BY DATABASE SECURITY FAILURES:     ║
║                                                                  ║
║  • Equifax (2017): 147M records — unpatched Apache Struts       ║
║  • Marriott (2018): 500M records — unencrypted passport data    ║
║  • Capital One (2019): 100M records — misconfigured WAF/DB      ║
║  • Facebook (2019): 540M records — publicly exposed on S3       ║
║  • SolarWinds (2020): Supply chain → database access            ║
║                                                                  ║
║  Average cost of a data breach: $4.45 MILLION (IBM, 2023)      ║
║                                                                  ║
║  Your database is the #1 target. Not your app. Not your API.   ║
║  THE DATABASE.                                                   ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🏛️ The Three Pillars of Database Security

```
          ┌─────────────────────────────────────┐
          │       DATABASE SECURITY              │
          │                                     │
          │    ┌───────────────────────────┐    │
          │    │  1. AUTHENTICATION        │    │
          │    │  "WHO are you?"           │    │
          │    │  → Verify identity        │    │
          │    └───────────┬───────────────┘    │
          │                │                    │
          │    ┌───────────▼───────────────┐    │
          │    │  2. AUTHORIZATION         │    │
          │    │  "WHAT can you do?"       │    │
          │    │  → Grant/deny permissions │    │
          │    └───────────┬───────────────┘    │
          │                │                    │
          │    ┌───────────▼───────────────┐    │
          │    │  3. ENCRYPTION            │    │
          │    │  "Can anyone EAVESDROP?"  │    │
          │    │  → Protect data at rest   │    │
          │    │     and in transit        │    │
          │    └───────────────────────────┘    │
          │                                     │
          │  + Auditing (WHO did WHAT, WHEN?)   │
          │  + Input Validation (SQL Injection) │
          │  + Network Security (Firewalls)     │
          └─────────────────────────────────────┘
```

---

## 🔑 Pillar 1: Authentication — "Who Are You?"

### Authentication Methods

```
╔═══════════════════════════════════════════════════════════════════╗
║ Method              │ Security │ Used By                         ║
╠═════════════════════╪══════════╪═════════════════════════════════╣
║ Username/Password   │ 🟡 Basic │ All databases (default)         ║
║ OS Authentication   │ 🟢 Good  │ SQL Server (Windows Auth)       ║
║                     │          │ Oracle (OS auth), PostgreSQL    ║
║ Certificate-based   │ 🟢 Good  │ PostgreSQL, MongoDB, MySQL      ║
║ Kerberos / LDAP     │ 🟢 Great │ Enterprise (AD integration)     ║
║ IAM (Cloud)         │ 🟢 Great │ AWS RDS, Azure SQL, GCP Cloud  ║
║ SCRAM-SHA-256       │ 🟢 Great │ PostgreSQL 10+, MongoDB 4.0+   ║
║ MFA / 2FA           │ 🔴 Best  │ Cloud databases, Oracle 21c+   ║
╚═════════════════════╧══════════╧═════════════════════════════════╝
```

### Setting Up Authentication

```sql
-- ═══════════════════════════════════════════
-- PostgreSQL: pg_hba.conf (authentication rules)
-- ═══════════════════════════════════════════
-- TYPE  DATABASE  USER       ADDRESS         METHOD
  local  all       all                         scram-sha-256
  host   all       all        127.0.0.1/32    scram-sha-256
  host   all       all        10.0.0.0/8      scram-sha-256
  host   all       all        0.0.0.0/0       reject          -- Block all external!

-- Create user with strong password
CREATE USER app_user WITH PASSWORD 'K$9x!mP2@vL7nQ' 
    VALID UNTIL '2025-12-31'        -- Password expiry
    CONNECTION LIMIT 10;             -- Max connections

-- ═══════════════════════════════════════════
-- MySQL: Authentication setup
-- ═══════════════════════════════════════════
CREATE USER 'app_user'@'10.0.0.%'                -- Only from internal network!
    IDENTIFIED WITH caching_sha2_password         -- Strong auth plugin
    BY 'K$9x!mP2@vL7nQ'
    PASSWORD EXPIRE INTERVAL 90 DAY               -- Force rotation
    FAILED_LOGIN_ATTEMPTS 3                       -- Lock after 3 failures
    PASSWORD_LOCK_TIME 1;                         -- Lock for 1 day

-- ═══════════════════════════════════════════
-- SQL Server: Authentication modes
-- ═══════════════════════════════════════════
-- Windows Authentication (recommended for domain environments)
CREATE LOGIN [DOMAIN\app_service] FROM WINDOWS;

-- SQL Authentication
CREATE LOGIN app_user WITH PASSWORD = 'K$9x!mP2@vL7nQ',
    CHECK_POLICY = ON,           -- Enforce Windows password policy
    CHECK_EXPIRATION = ON;       -- Password expiration

-- ═══════════════════════════════════════════
-- Oracle: Authentication setup
-- ═══════════════════════════════════════════
CREATE USER app_user IDENTIFIED BY "K$9x!mP2@vL7nQ"
    DEFAULT TABLESPACE app_data
    QUOTA 500M ON app_data
    PROFILE app_profile;          -- Password policies via profiles

-- Create password profile
CREATE PROFILE app_profile LIMIT
    FAILED_LOGIN_ATTEMPTS 3
    PASSWORD_LOCK_TIME 1
    PASSWORD_LIFE_TIME 90
    PASSWORD_REUSE_TIME 365
    PASSWORD_REUSE_MAX 12;

-- ═══════════════════════════════════════════
-- MongoDB: Authentication
-- ═══════════════════════════════════════════
// Enable authentication in mongod.conf:
// security:
//   authorization: enabled

// Create admin user
db.createUser({
    user: "app_user",
    pwd: "K$9x!mP2@vL7nQ",
    roles: [{ role: "readWrite", db: "myapp" }],
    mechanisms: ["SCRAM-SHA-256"]     // Strong auth mechanism
});
```

### Password Security Rules

```
╔═══════════════════════════════════════════════════════════════╗
║                 PASSWORD SECURITY CHECKLIST                   ║
╠═══════════════════════════════════════════════════════════════╣
║ ✅ Minimum 16 characters                                     ║
║ ✅ Mix: uppercase, lowercase, numbers, special characters    ║
║ ✅ Never use default passwords (sa, root, admin, password)   ║
║ ✅ Rotate every 60-90 days                                   ║
║ ✅ Don't store in code — use secrets managers:               ║
║    → AWS Secrets Manager                                     ║
║    → Azure Key Vault                                         ║
║    → HashiCorp Vault                                         ║
║    → Environment variables (minimum)                         ║
║ ✅ Lock accounts after 3-5 failed attempts                   ║
║ ✅ Use connection strings with SSL/TLS                       ║
║ ❌ NEVER: hardcode passwords in source code                  ║
║ ❌ NEVER: share database credentials across environments     ║
║ ❌ NEVER: use root/sa account for application connections    ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## 🎭 Pillar 2: Authorization — "What Can You Do?"

### The Principle of Least Privilege

> **Give each user/application the MINIMUM permissions needed to do their job — nothing more.**

```
❌ WRONG (seen in production way too often):
   GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%';
   -- App can DROP DATABASE, create users, read all data 💀

✅ RIGHT:
   GRANT SELECT, INSERT, UPDATE ON myapp.orders TO 'app_user'@'10.0.0.%';
   -- App can only read/write orders table, nothing else
```

### Role-Based Access Control (RBAC)

```
                    RBAC Architecture

     ┌─────────┐     ┌─────────┐     ┌─────────┐
     │  Users  │     │  Roles  │     │ Privs   │
     └────┬────┘     └────┬────┘     └────┬────┘
          │               │               │
     ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
     │ alice   │────►│ analyst │────►│ SELECT  │
     │ bob     │────►│         │     │ on sales│
     └─────────┘     └─────────┘     └─────────┘
     ┌─────────┐     ┌─────────┐     ┌─────────┐
     │ charlie │────►│ app_svc │────►│ SELECT  │
     │ app_v2  │────►│         │     │ INSERT  │
     └─────────┘     └─────────┘     │ UPDATE  │
                                     │on orders│
                                     └─────────┘
     ┌─────────┐     ┌─────────┐     ┌─────────┐
     │ dave    │────►│  dba    │────►│ ALL on  │
     └─────────┘     └─────────┘     │ database│
                                     └─────────┘

  Benefits:
  ✅ Add new user → assign role → done (no per-user grants)
  ✅ Change permissions → update role → affects all users in role
  ✅ Easy to audit: "Who has DELETE access?" → check roles
```

### Implementing RBAC

```sql
-- ═══════════════════════════════════════════
-- PostgreSQL RBAC
-- ═══════════════════════════════════════════

-- Create roles (no login — these are "groups")
CREATE ROLE readonly;
CREATE ROLE readwrite;
CREATE ROLE admin;

-- Grant permissions to roles
GRANT CONNECT ON DATABASE myapp TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;

GRANT readonly TO readwrite;  -- Inherit all readonly permissions
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite;

GRANT readwrite TO admin;
GRANT CREATE ON SCHEMA public TO admin;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO admin;

-- Assign users to roles
CREATE USER analyst_alice WITH PASSWORD 'xxx' IN ROLE readonly;
CREATE USER app_service WITH PASSWORD 'xxx' IN ROLE readwrite;
CREATE USER dba_dave WITH PASSWORD 'xxx' IN ROLE admin;

-- Row-Level Security (PostgreSQL) — users see only THEIR data!
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_orders ON orders 
    USING (user_id = current_user_id());  -- Users only see their own orders!

-- ═══════════════════════════════════════════
-- MySQL RBAC
-- ═══════════════════════════════════════════

-- Create roles
CREATE ROLE 'app_read', 'app_write', 'app_admin';

-- Grant to roles
GRANT SELECT ON myapp.* TO 'app_read';
GRANT INSERT, UPDATE, DELETE ON myapp.* TO 'app_write';
GRANT ALL PRIVILEGES ON myapp.* TO 'app_admin';

-- Assign roles to users
GRANT 'app_read' TO 'analyst'@'%';
GRANT 'app_read', 'app_write' TO 'app_service'@'10.0.0.%';

-- Activate roles
SET DEFAULT ROLE 'app_read', 'app_write' TO 'app_service'@'10.0.0.%';

-- ═══════════════════════════════════════════
-- SQL Server RBAC
-- ═══════════════════════════════════════════

-- Database roles
CREATE ROLE app_reader;
CREATE ROLE app_writer;

GRANT SELECT ON SCHEMA::dbo TO app_reader;
GRANT INSERT, UPDATE, DELETE ON SCHEMA::dbo TO app_writer;

-- Add users to roles
ALTER ROLE app_reader ADD MEMBER [analyst_alice];
ALTER ROLE app_writer ADD MEMBER [app_service];

-- Column-level security
GRANT SELECT ON dbo.employees (name, department) TO app_reader;
-- Can see name & department, but NOT salary!
DENY SELECT ON dbo.employees (salary, ssn) TO app_reader;

-- ═══════════════════════════════════════════
-- Oracle RBAC
-- ═══════════════════════════════════════════

-- Create roles
CREATE ROLE app_readonly;
CREATE ROLE app_readwrite;

-- Grant privileges to roles
GRANT SELECT ON schema.orders TO app_readonly;
GRANT SELECT, INSERT, UPDATE ON schema.orders TO app_readwrite;

-- Grant roles to users
GRANT app_readonly TO analyst_alice;
GRANT app_readwrite TO app_service;

-- Virtual Private Database (VPD) — row-level security
-- Users automatically see only their department's data
BEGIN
    DBMS_RLS.ADD_POLICY(
        object_schema   => 'HR',
        object_name     => 'EMPLOYEES',
        policy_name     => 'dept_policy',
        function_schema => 'HR',
        policy_function => 'dept_security_fn',
        statement_types => 'SELECT,UPDATE,DELETE'
    );
END;

-- ═══════════════════════════════════════════
-- MongoDB RBAC
-- ═══════════════════════════════════════════

// Built-in roles
db.createUser({
    user: "analyst",
    pwd: "xxx",
    roles: [
        { role: "read", db: "myapp" }           // Read-only
    ]
});

db.createUser({
    user: "app_service",
    pwd: "xxx",
    roles: [
        { role: "readWrite", db: "myapp" },      // Read + Write
        { role: "read", db: "analytics" }         // Read-only on analytics DB
    ]
});

// Custom role
db.createRole({
    role: "orderManager",
    privileges: [
        { resource: { db: "myapp", collection: "orders" },
          actions: ["find", "insert", "update"] },
        { resource: { db: "myapp", collection: "inventory" },
          actions: ["find"] }                     // Read-only on inventory
    ],
    roles: []
});
```

---

## 💉 SQL Injection — The #1 Database Attack

> **SQL Injection has been the #1 web vulnerability for 20+ years (OWASP Top 10).**

### How SQL Injection Works

```
Normal Login:
  Username: alice
  Password: secret123
  
  Query: SELECT * FROM users WHERE username = 'alice' AND password = 'secret123';
  → Returns Alice's record ✅

SQL Injection Attack:
  Username: admin' --
  Password: anything
  
  Query: SELECT * FROM users WHERE username = 'admin' --' AND password = 'anything';
                                                       ↑
                                                   Comment! Everything after -- is ignored!
  
  Effective query: SELECT * FROM users WHERE username = 'admin'
  → Attacker logs in as admin WITHOUT knowing the password! 💀
```

### Devastating Injection Examples

```sql
-- ═══════════════════════════════════════════
-- Attack 1: Data theft (UNION-based)
-- ═══════════════════════════════════════════
-- Input: ' UNION SELECT username, password, null FROM users --
-- Query becomes:
SELECT name, description, price FROM products WHERE id = ''
UNION SELECT username, password, null FROM users --'
-- → Dumps ALL usernames and passwords! 💀

-- ═══════════════════════════════════════════
-- Attack 2: Drop entire database
-- ═══════════════════════════════════════════
-- Input: '; DROP TABLE users; --
-- Query becomes:
SELECT * FROM users WHERE id = ''; DROP TABLE users; --'
-- → Deletes the entire users table! 💀

-- ═══════════════════════════════════════════
-- Attack 3: Read server files (MySQL)
-- ═══════════════════════════════════════════
-- Input: ' UNION SELECT LOAD_FILE('/etc/passwd'), null, null --
-- → Reads the server's password file! 💀

-- ═══════════════════════════════════════════
-- Attack 4: Blind injection (time-based)
-- ═══════════════════════════════════════════
-- Input: ' OR IF(SUBSTRING(password,1,1)='a', SLEEP(5), 0) --
-- If page takes 5 seconds → first character of password is 'a'
-- Repeat for each character → extract entire password! 💀
```

### Prevention — The ONLY Way: Parameterized Queries

```python
# ═══════════════════════════════════════════
# Python (any database)
# ═══════════════════════════════════════════

# ❌ VULNERABLE — String concatenation
query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{password}'"
cursor.execute(query)  # 💀 SQL INJECTION!

# ✅ SAFE — Parameterized query
cursor.execute(
    "SELECT * FROM users WHERE username = %s AND password = %s",
    (username, password)  # Parameters are safely escaped
)
```

```javascript
// ═══════════════════════════════════════════
// Node.js — Different libraries
// ═══════════════════════════════════════════

// ❌ VULNERABLE
const query = `SELECT * FROM users WHERE id = ${userId}`;  // 💀
db.query(query);

// ✅ SAFE — Parameterized
db.query("SELECT * FROM users WHERE id = $1", [userId]);         // PostgreSQL (pg)
db.query("SELECT * FROM users WHERE id = ?", [userId]);          // MySQL (mysql2)
```

```java
// ═══════════════════════════════════════════
// Java — PreparedStatement
// ═══════════════════════════════════════════

// ❌ VULNERABLE — String concatenation
String query = "SELECT * FROM users WHERE id = " + userId;  // 💀
Statement stmt = conn.createStatement();
stmt.executeQuery(query);

// ✅ SAFE — PreparedStatement
PreparedStatement pstmt = conn.prepareStatement(
    "SELECT * FROM users WHERE id = ? AND status = ?"
);
pstmt.setInt(1, userId);
pstmt.setString(2, status);
ResultSet rs = pstmt.executeQuery();
```

```csharp
// ═══════════════════════════════════════════
// C# / .NET — SqlParameter
// ═══════════════════════════════════════════

// ❌ VULNERABLE
string query = $"SELECT * FROM users WHERE id = {userId}";  // 💀

// ✅ SAFE — SqlParameter
using var cmd = new SqlCommand(
    "SELECT * FROM users WHERE id = @userId AND status = @status", conn);
cmd.Parameters.AddWithValue("@userId", userId);
cmd.Parameters.AddWithValue("@status", status);
```

```javascript
// ═══════════════════════════════════════════
// MongoDB — NoSQL Injection Prevention
// ═══════════════════════════════════════════

// ❌ VULNERABLE — User input as query operator
const user = req.body;
db.users.find({ username: user.username, password: user.password });
// Attack: { "username": "admin", "password": { "$ne": "" } }
// → Matches admin with ANY non-empty password! 💀

// ✅ SAFE — Validate/sanitize input types
const username = String(req.body.username);  // Force string type
const password = String(req.body.password);
db.users.find({ username: username, password: password });

// ✅ SAFER — Use mongoose with schema validation
const userSchema = new Schema({
    username: { type: String, required: true },
    password: { type: String, required: true }
});
```

### SQL Injection Prevention Checklist

```
╔═══════════════════════════════════════════════════════════════╗
║           SQL INJECTION PREVENTION — COMPLETE                 ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Layer 1: CODE                                               ║
║  ✅ ALWAYS use parameterized queries / prepared statements   ║
║  ✅ Use ORM (Hibernate, SQLAlchemy, Prisma, Sequelize)       ║
║  ✅ Validate input types (is this really an integer?)        ║
║  ✅ Whitelist allowed values for dynamic column/table names  ║
║  ❌ NEVER concatenate user input into SQL strings            ║
║                                                               ║
║  Layer 2: DATABASE                                           ║
║  ✅ App user has MINIMUM privileges (no DROP, no GRANT)      ║
║  ✅ Use stored procedures with parameterized inputs          ║
║  ✅ Disable dangerous features (xp_cmdshell, LOAD_FILE)     ║
║                                                               ║
║  Layer 3: NETWORK                                            ║
║  ✅ Web Application Firewall (WAF) — blocks known patterns  ║
║  ✅ Database not directly accessible from internet           ║
║  ✅ API rate limiting (slow down brute-force attacks)        ║
║                                                               ║
║  Layer 4: MONITORING                                         ║
║  ✅ Log all failed queries and unusual patterns              ║
║  ✅ Alert on: UNION SELECT, DROP, xp_cmdshell in logs       ║
║  ✅ Regular penetration testing                              ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## 🔐 Pillar 3: Encryption — Protecting Data

### Encryption at Rest vs In Transit

```
╔══════════════════════════════════════════════════════════════════╗
║                                                                  ║
║  ENCRYPTION AT REST                ENCRYPTION IN TRANSIT         ║
║  (Data on disk)                    (Data over network)           ║
║                                                                  ║
║  ┌─────────────┐                   ┌──────┐   TLS   ┌──────┐   ║
║  │  Database   │                   │ App  │◄═══════►│  DB  │   ║
║  │  ┌───────┐  │                   │Server│ 🔒🔒🔒  │Server│   ║
║  │  │🔒Data │  │                   └──────┘         └──────┘   ║
║  │  │🔒🔒🔒 │  │                                                ║
║  │  └───────┘  │                   Without TLS:                 ║
║  │  Encrypted  │                   │ App  │──plain──►│  DB  │   ║
║  │  on disk    │                   Attacker can READ everything! ║
║  └─────────────┘                                                 ║
║                                                                  ║
║  Protects against:                 Protects against:             ║
║  • Stolen hard drives             • Network sniffing             ║
║  • Unauthorized file access       • Man-in-the-middle attacks    ║
║  • Cloud storage breaches         • Packet interception          ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Encryption In Transit (TLS/SSL)

```sql
-- ═══════════════════════════════════════════
-- PostgreSQL — Enable SSL
-- ═══════════════════════════════════════════
-- postgresql.conf:
--   ssl = on
--   ssl_cert_file = '/path/to/server.crt'
--   ssl_key_file = '/path/to/server.key'

-- pg_hba.conf — Force SSL for remote connections:
--   hostssl  all  all  0.0.0.0/0  scram-sha-256

-- Connection string:
-- postgresql://user:pass@host:5432/db?sslmode=verify-full&sslrootcert=ca.crt

-- ═══════════════════════════════════════════
-- MySQL — Enable TLS
-- ═══════════════════════════════════════════
-- my.cnf:
--   [mysqld]
--   require_secure_transport = ON
--   ssl-ca = /path/to/ca.pem
--   ssl-cert = /path/to/server-cert.pem
--   ssl-key = /path/to/server-key.pem

-- Force TLS for a user:
ALTER USER 'app_user'@'%' REQUIRE SSL;
-- Or require specific certificate:
ALTER USER 'app_user'@'%' REQUIRE X509;

-- ═══════════════════════════════════════════
-- SQL Server — Force Encryption
-- ═══════════════════════════════════════════
-- SQL Server Configuration Manager → SQL Server Network Configuration
-- → Protocols → Properties → Force Encryption = Yes

-- Connection string:
-- Server=myserver;Database=mydb;Encrypt=True;TrustServerCertificate=False;

-- ═══════════════════════════════════════════
-- MongoDB — TLS
-- ═══════════════════════════════════════════
-- mongod.conf:
--   net:
--     tls:
--       mode: requireTLS
--       certificateKeyFile: /path/to/server.pem
--       CAFile: /path/to/ca.pem

-- Connection string:
-- mongodb://user:pass@host:27017/db?tls=true&tlsCAFile=ca.pem
```

### Encryption at Rest

```sql
-- ═══════════════════════════════════════════
-- PostgreSQL — pgcrypto + TDE (via extensions)
-- ═══════════════════════════════════════════
-- Column-level encryption with pgcrypto:
CREATE EXTENSION pgcrypto;

-- Encrypt sensitive data
INSERT INTO users (name, ssn_encrypted) 
VALUES ('Alice', pgp_sym_encrypt('123-45-6789', 'my-secret-key'));

-- Decrypt when needed
SELECT name, pgp_sym_decrypt(ssn_encrypted::bytea, 'my-secret-key') as ssn
FROM users;

-- ═══════════════════════════════════════════
-- SQL Server — Transparent Data Encryption (TDE)
-- ═══════════════════════════════════════════
-- Encrypts entire database files on disk — transparent to app!
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongPassword!';
CREATE CERTIFICATE TDE_Cert WITH SUBJECT = 'TDE Certificate';
CREATE DATABASE ENCRYPTION KEY
    WITH ALGORITHM = AES_256
    ENCRYPTION BY SERVER CERTIFICATE TDE_Cert;
ALTER DATABASE MyDatabase SET ENCRYPTION ON;
-- Done! All data/log/backup files are now encrypted 🔒

-- Always Encrypted (SQL Server) — data encrypted EVEN from DBA!
CREATE COLUMN MASTER KEY CMK1 WITH (
    KEY_STORE_PROVIDER_NAME = 'AZURE_KEY_VAULT',
    KEY_PATH = 'https://myvault.vault.azure.net/keys/CMK1'
);

-- ═══════════════════════════════════════════
-- Oracle — Transparent Data Encryption (TDE)
-- ═══════════════════════════════════════════
-- Tablespace-level encryption:
CREATE TABLESPACE secure_data
    DATAFILE '/path/to/secure01.dbf' SIZE 500M
    ENCRYPTION USING 'AES256'
    DEFAULT STORAGE(ENCRYPT);

-- Column-level encryption:
ALTER TABLE employees MODIFY (ssn ENCRYPT);

-- ═══════════════════════════════════════════
-- MySQL — InnoDB Tablespace Encryption
-- ═══════════════════════════════════════════
-- my.cnf:
--   early-plugin-load = keyring_file.so
--   keyring_file_data = /var/lib/mysql/keyring

ALTER TABLE sensitive_data ENCRYPTION = 'Y';

-- ═══════════════════════════════════════════
-- MongoDB — Encryption at Rest
-- ═══════════════════════════════════════════
-- mongod.conf:
--   security:
--     enableEncryption: true
--     encryptionKeyFile: /path/to/keyfile

-- Client-Side Field Level Encryption (CSFLE) — strongest option
// Even the DATABASE SERVER can't read encrypted fields!
const encryptedClient = new MongoClient(uri, {
    autoEncryption: {
        keyVaultNamespace: "encryption.__keyVault",
        kmsProviders: { aws: { accessKeyId: "...", secretAccessKey: "..." } },
        schemaMap: {
            "mydb.users": {
                properties: {
                    ssn: { encrypt: { algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic" } }
                }
            }
        }
    }
});
```

---

## 📋 Audit Logging — Who Did What, When?

> **If you can't prove what happened, you can't investigate breaches or meet compliance.**

```
╔═══════════════════════════════════════════════════════════════╗
║                   AUDIT LOG ENTRY                             ║
╠═══════════════════════════════════════════════════════════════╣
║ Timestamp:  2024-03-15 14:23:45.123 UTC                      ║
║ User:       app_service@10.0.0.42                            ║
║ Action:     UPDATE                                           ║
║ Object:     public.users (row id=12345)                      ║
║ Old Value:  email = 'old@email.com'                          ║
║ New Value:  email = 'new@email.com'                          ║
║ Query:      UPDATE users SET email=$1 WHERE id=$2            ║
║ Session ID: abc-123-def                                      ║
║ App:        web-api-v2                                       ║
╚═══════════════════════════════════════════════════════════════╝
```

### Implementation

```sql
-- ═══════════════════════════════════════════
-- PostgreSQL — pgaudit extension
-- ═══════════════════════════════════════════
-- postgresql.conf:
--   shared_preload_libraries = 'pgaudit'
--   pgaudit.log = 'write, ddl'          -- Log writes and DDL
--   pgaudit.log_catalog = off

-- Or per-table audit with triggers:
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,
    action TEXT NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_by TEXT DEFAULT current_user,
    changed_at TIMESTAMPTZ DEFAULT now()
);

CREATE OR REPLACE FUNCTION audit_trigger() RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, action, old_data, new_data)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        CASE WHEN TG_OP IN ('UPDATE','DELETE') THEN row_to_json(OLD)::jsonb END,
        CASE WHEN TG_OP IN ('INSERT','UPDATE') THEN row_to_json(NEW)::jsonb END
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_audit
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_trigger();

-- ═══════════════════════════════════════════
-- SQL Server — Built-in Auditing
-- ═══════════════════════════════════════════
CREATE SERVER AUDIT MyAudit
    TO FILE (FILEPATH = 'C:\AuditLogs\', MAXSIZE = 1 GB);
ALTER SERVER AUDIT MyAudit WITH (STATE = ON);

CREATE DATABASE AUDIT SPECIFICATION MyDBSpec
    FOR SERVER AUDIT MyAudit
    ADD (SELECT, INSERT, UPDATE, DELETE ON SCHEMA::dbo BY public);
ALTER DATABASE AUDIT SPECIFICATION MyDBSpec WITH (STATE = ON);

-- ═══════════════════════════════════════════
-- Oracle — Unified Auditing (12c+)
-- ═══════════════════════════════════════════
CREATE AUDIT POLICY sensitive_data_policy
    ACTIONS UPDATE ON hr.employees,
            SELECT ON hr.employees,
            DELETE ON hr.employees;
AUDIT POLICY sensitive_data_policy;

-- ═══════════════════════════════════════════
-- MongoDB — Audit Log
-- ═══════════════════════════════════════════
-- mongod.conf (Enterprise only):
--   auditLog:
--     destination: file
--     format: JSON
--     path: /var/log/mongodb/audit.json
--     filter: '{ atype: { $in: ["authCheck", "createCollection", "dropCollection"] } }'
```

---

## 🏗️ Security Architecture — Complete Picture

```
                    DEFENSE IN DEPTH

    ┌─────────────────────────────────────────────────────┐
    │                    INTERNET                          │
    │                       │                              │
    │              ┌────────▼────────┐                    │
    │  Layer 1:    │   WAF / CDN    │  Block SQL injection│
    │  Network     │   (CloudFlare) │  patterns, DDoS     │
    │              └────────┬────────┘                    │
    │                       │                              │
    │              ┌────────▼────────┐                    │
    │  Layer 2:    │   Load Balancer │  TLS termination   │
    │  Transport   │                │                     │
    │              └────────┬────────┘                    │
    │                       │                              │
    │              ┌────────▼────────┐                    │
    │  Layer 3:    │   App Server   │  Parameterized      │
    │  Application │   (API)       │  queries, input      │
    │              │               │  validation          │
    │              └────────┬────────┘                    │
    │                       │ TLS/SSL                      │
    │              ┌────────▼────────┐                    │
    │  Layer 4:    │   DB Firewall  │  Whitelist IPs,     │
    │  Database    │   (pgbouncer)  │  connection limits  │
    │  Network     └────────┬────────┘                    │
    │                       │                              │
    │              ┌────────▼────────┐                    │
    │  Layer 5:    │   DATABASE     │                     │
    │  Database    │ ┌────────────┐ │  RBAC, Row-Level    │
    │  Access      │ │Auth + RBAC │ │  Security, Audit    │
    │              │ └────────────┘ │                     │
    │              │ ┌────────────┐ │                     │
    │  Layer 6:    │ │Encryption  │ │  TDE, Column-level  │
    │  Data        │ │at Rest  🔒│ │  encryption         │
    │              │ └────────────┘ │                     │
    │              └────────────────┘                     │
    └─────────────────────────────────────────────────────┘
```

---

## 🧪 Security Quick Reference by Database

```
╔══════════════════╦══════════════╦══════════════╦═══════════════╦══════════════╗
║ Feature          ║ PostgreSQL   ║ MySQL        ║ SQL Server    ║ Oracle       ║
╠══════════════════╬══════════════╬══════════════╬═══════════════╬══════════════╣
║ Auth methods     ║ SCRAM, cert, ║ SHA-2, cert, ║ Windows,      ║ Password,    ║
║                  ║ LDAP, GSSAPI ║ LDAP, PAM    ║ SQL, Azure AD ║ Kerberos,    ║
║                  ║              ║              ║               ║ LDAP, cert   ║
╠══════════════════╬══════════════╬══════════════╬═══════════════╬══════════════╣
║ Row-Level Sec    ║ ✅ Native    ║ ❌ (views)   ║ ✅ Native     ║ ✅ VPD       ║
╠══════════════════╬══════════════╬══════════════╬═══════════════╬══════════════╣
║ Column Encrypt   ║ pgcrypto     ║ AES_ENCRYPT  ║ Always        ║ TDE column   ║
║                  ║              ║              ║ Encrypted     ║              ║
╠══════════════════╬══════════════╬══════════════╬═══════════════╬══════════════╣
║ TDE              ║ Enterprise   ║ ✅ InnoDB    ║ ✅ Enterprise ║ ✅ Advanced  ║
║                  ║ (or pgcrypto)║              ║               ║ Security     ║
╠══════════════════╬══════════════╬══════════════╬═══════════════╬══════════════╣
║ Audit            ║ pgaudit      ║ Enterprise   ║ ✅ Built-in   ║ ✅ Unified   ║
║                  ║ extension    ║ Audit plugin ║               ║ Auditing     ║
╠══════════════════╬══════════════╬══════════════╬═══════════════╬══════════════╣
║ Data Masking     ║ Extensions   ║ Enterprise   ║ ✅ Dynamic    ║ ✅ Data      ║
║                  ║              ║              ║ Data Masking  ║ Redaction    ║
╚══════════════════╩══════════════╩══════════════╩═══════════════╩══════════════╝
```

---

## 🔑 Key Takeaways

```
✅ Authentication: Use strong methods (SCRAM, certificates, IAM) — never default passwords
✅ Authorization: Principle of Least Privilege — grant MINIMUM needed, use RBAC
✅ SQL Injection: ALWAYS use parameterized queries — #1 rule, no exceptions
✅ Encryption in Transit: TLS/SSL on ALL database connections — no plain text
✅ Encryption at Rest: TDE for full database, column-level for sensitive fields
✅ Audit Logging: Track WHO did WHAT, WHEN — required for compliance
✅ Defense in Depth: Multiple layers — WAF, app validation, DB permissions, encryption
✅ Row-Level Security: Users see only their own data (PostgreSQL, SQL Server, Oracle)
✅ Never expose databases directly to the internet
✅ Rotate credentials regularly, use secrets managers (Vault, AWS Secrets Manager)
```

---

## 🔗 What's Next?

**Chapter 1.10 → [Backup, Recovery & Disaster Planning](./10-Backup-and-Recovery.md)**
Where we learn the last line of defense — because no matter how good your security is, disasters happen.

---

> *"Security is not a product, but a process. And the database is where that process matters most."* — Bruce Schneier (adapted)
