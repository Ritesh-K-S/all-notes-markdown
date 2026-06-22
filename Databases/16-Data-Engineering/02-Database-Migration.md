# 🔀 Chapter 5.2 — Database Migration Strategies

> **Level:** 🟡 Intermediate | ⭐ Must-Know
> **Time to Master:** ~3-4 hours
> **Prerequisites:** DDL basics (Chapter 2.2), Transaction concepts (Chapter 1.8)

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **why schema migrations are critical** — and what goes wrong without them
- Master the **top migration tools**: Flyway, Liquibase, Alembic, Knex, Prisma Migrate
- Design **zero-downtime migrations** for production databases
- Implement **version-controlled schemas** — treat your DB like code
- Handle the scariest scenarios: **rollbacks, data migrations, cross-database migrations**
- Build confidence to run migrations on **Friday afternoon** (okay, maybe Monday morning)

---

## 🧠 The Problem — Why Database Migrations Exist

### The Horror Story Without Migrations

```
  Team of 5 developers, 1 shared database, no migration tool:

  Monday:
    Dev A: "I added a 'phone' column to users table on my local DB"
    Dev B: "Cool, I added a 'phone_number' column to users on MY local DB"
    Dev C: "Wait, I deleted the 'fax' column..."

  Wednesday (Deploy Day):
    → Which version of the schema is "correct"?
    → Production DB has NONE of these changes
    → Someone runs ALTER TABLE manually on production... 💀
    → Forgets one column, app crashes at 3 AM
    → Rollback? What rollback? Nobody wrote one.

  Result: 3 AM incident, angry customers, blame game 🔥
```

```
  WITH Migrations:

  ┌─────────────────────────────────────────────────────────────┐
  │  V1_create_users.sql         → CREATE TABLE users (...)     │
  │  V2_add_email_to_users.sql   → ALTER TABLE users ADD email  │
  │  V3_add_phone_to_users.sql   → ALTER TABLE users ADD phone  │
  │  V4_drop_fax_from_users.sql  → ALTER TABLE users DROP fax   │
  └─────────────────────────────────────────────────────────────┘

  Every environment (dev, staging, prod) runs the SAME migrations
  in the SAME order → identical schema EVERYWHERE → zero surprises ✅
```

> **Database migrations are version-controlled scripts that evolve your database schema incrementally, reliably, and reproducibly.**

---

## 📦 Core Concepts

### The Migration Table — Your Schema's Version History

Every migration tool creates a **tracking table** in your database:

```
  schema_version / flyway_schema_history / alembic_version

  ┌─────────┬────────────────────────────────┬─────────────────────┬─────────┐
  │ version │ description                    │ executed_at          │ success │
  ├─────────┼────────────────────────────────┼─────────────────────┼─────────┤
  │ 1       │ Create users table             │ 2026-01-15 10:00:00 │ ✅      │
  │ 2       │ Add email column to users      │ 2026-02-01 14:30:00 │ ✅      │
  │ 3       │ Create orders table            │ 2026-02-15 09:00:00 │ ✅      │
  │ 4       │ Add index on orders.user_id    │ 2026-03-01 11:00:00 │ ✅      │
  │ 5       │ Add phone column to users      │ 2026-06-01 08:00:00 │ ✅      │
  └─────────┴────────────────────────────────┴─────────────────────┴─────────┘

  When you run "migrate":
  1. Tool checks: "What version is the DB at?" → Version 5
  2. Tool checks: "What migrations exist in code?" → V1 through V7
  3. Tool runs: V6 and V7 (only the NEW ones)
  4. Tool records: V6 and V7 in the tracking table
```

### Migration Types

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    Migration Types                           │
  ├──────────────────────┬───────────────────────────────────────┤
  │                      │                                       │
  │  SCHEMA MIGRATION    │  DATA MIGRATION                       │
  │  (Structure changes) │  (Content changes)                    │
  │                      │                                       │
  │  • CREATE TABLE      │  • Backfill new columns               │
  │  • ALTER TABLE       │  • Transform existing data            │
  │  • ADD/DROP INDEX    │  • Merge/split tables                  │
  │  • ADD/DROP COLUMN   │  • Encrypt sensitive fields           │
  │  • RENAME COLUMN     │  • Migrate to new format              │
  │                      │                                       │
  │  Usually FAST        │  Can be SLOW (millions of rows)       │
  │  Low risk            │  Higher risk (data loss possible)     │
  └──────────────────────┴───────────────────────────────────────┘
```

### Versioned vs Repeatable Migrations

```
  VERSIONED (run once, in order):
  V1__create_users.sql        ← Runs once, tracked by version number
  V2__add_email.sql           ← Must run AFTER V1
  V3__create_orders.sql       ← Must run AFTER V2

  REPEATABLE (run every time they change):
  R__create_views.sql         ← Re-run whenever the file changes
  R__refresh_permissions.sql  ← Great for views, functions, grants

  Why repeatable?
  → Views/functions should reflect the LATEST definition
  → No need to create V47__update_revenue_view.sql every time
  → Tool detects file hash changed → re-runs it
```

---

## 🛠️ Migration Tool Deep Dives

### 1. Flyway — The Industry Standard ⭐🔥

> **Language:** Java (but works with ANY database via SQL files)
> **Used by:** 100K+ companies, Spring Boot default
> **Supports:** Oracle, SQL Server, PostgreSQL, MySQL, MariaDB, SQLite, CockroachDB, 20+ more

```
  Flyway Project Structure:

  my-app/
  ├── src/main/resources/db/migration/
  │   ├── V1__create_users_table.sql
  │   ├── V2__create_orders_table.sql
  │   ├── V3__add_email_to_users.sql
  │   ├── V4__create_products_table.sql
  │   ├── V5__add_index_orders_user_id.sql
  │   └── R__create_revenue_view.sql        ← Repeatable
  └── flyway.conf
```

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    username    VARCHAR(50) NOT NULL UNIQUE,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- V2__create_orders_table.sql
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id     BIGINT NOT NULL,
    total       DECIMAL(10,2) NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- V3__add_email_to_users.sql
ALTER TABLE users ADD COLUMN email VARCHAR(255);
CREATE INDEX idx_users_email ON users(email);

-- V5__add_index_orders_user_id.sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

```bash
# Flyway CLI Commands
flyway migrate         # Run pending migrations
flyway info            # Show migration status
flyway validate        # Verify applied migrations match local files
flyway repair          # Fix the schema history table
flyway clean           # ⚠️ DROP ALL OBJECTS (never in production!)
flyway baseline        # Mark existing DB as V1 (for legacy DBs)
flyway undo            # Rollback last migration (Teams/Enterprise only)
```

```
  Flyway Migration Lifecycle:

  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  Pending  │────►│ Executing│────►│ Applied  │     │  Failed  │
  │           │     │          │     │ (success)│     │ (error)  │
  └──────────┘     └──────────┘     └──────────┘     └──────────┘
                         │                                 │
                         │            flyway repair ───────┘
                         │               (fix & retry)
                         │
                    On error, migration is
                    marked FAILED. DB may be
                    in partial state ⚠️
```

#### Flyway Configuration

```properties
# flyway.conf
flyway.url=jdbc:postgresql://localhost:5432/myapp
flyway.user=app_admin
flyway.password=${DB_PASSWORD}
flyway.schemas=public
flyway.locations=classpath:db/migration
flyway.baselineOnMigrate=true
flyway.validateOnMigrate=true
flyway.outOfOrder=false          # Force sequential execution
flyway.cleanDisabled=true        # NEVER allow clean in prod!
```

#### Spring Boot Integration (Zero Config!)

```yaml
# application.yml — Flyway runs automatically on app startup
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
```

---

### 2. Liquibase — The Flexible Powerhouse

> **Unique Feature:** Changelog can be SQL, XML, YAML, or JSON
> **Best For:** Teams needing rollback support, multi-database support

```xml
<!-- db/changelog/db.changelog-master.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <changeSet id="1" author="ritesh">
        <createTable tableName="users">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="username" type="VARCHAR(50)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="email" type="VARCHAR(255)"/>
            <column name="created_at" type="TIMESTAMP" defaultValueComputed="CURRENT_TIMESTAMP"/>
        </createTable>

        <!-- ROLLBACK is built-in! -->
        <rollback>
            <dropTable tableName="users"/>
        </rollback>
    </changeSet>

    <changeSet id="2" author="ritesh">
        <addColumn tableName="users">
            <column name="phone" type="VARCHAR(20)"/>
        </addColumn>

        <rollback>
            <dropColumn tableName="users" columnName="phone"/>
        </rollback>
    </changeSet>

    <changeSet id="3" author="ritesh" context="!test">
        <!-- This changeSet ONLY runs in non-test environments -->
        <createIndex tableName="users" indexName="idx_users_email">
            <column name="email"/>
        </createIndex>
    </changeSet>

</databaseChangeLog>
```

```yaml
# Same changeset in YAML format (Liquibase supports both!)
databaseChangeLog:
  - changeSet:
      id: 1
      author: ritesh
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: username
                  type: VARCHAR(50)
                  constraints:
                    nullable: false
                    unique: true
              - column:
                  name: email
                  type: VARCHAR(255)
      rollback:
        - dropTable:
            tableName: users
```

```bash
# Liquibase CLI Commands
liquibase update                 # Apply pending changesets
liquibase rollbackCount 1        # Rollback last 1 changeset
liquibase rollbackToDate 2026-06-01  # Rollback to specific date
liquibase status                 # Show pending changesets
liquibase diff                   # Compare two databases
liquibase generateChangeLog      # Reverse-engineer existing DB
liquibase validate               # Check changelog syntax
```

---

### 3. Alembic — Python's Migration Tool (SQLAlchemy)

> **Best For:** Python/Flask/FastAPI applications using SQLAlchemy

```python
# alembic/versions/001_create_users.py
"""create users table

Revision ID: a1b2c3d4e5f6
Revises: (none)
Create Date: 2026-06-01 10:00:00
"""
from alembic import op
import sqlalchemy as sa

# Revision identifiers
revision = 'a1b2c3d4e5f6'
down_revision = None

def upgrade():
    op.create_table(
        'users',
        sa.Column('id', sa.BigInteger(), primary_key=True, autoincrement=True),
        sa.Column('username', sa.String(50), nullable=False, unique=True),
        sa.Column('email', sa.String(255)),
        sa.Column('created_at', sa.DateTime(), server_default=sa.func.now()),
    )
    op.create_index('idx_users_email', 'users', ['email'])

def downgrade():
    op.drop_index('idx_users_email')
    op.drop_table('users')
```

```python
# alembic/versions/002_add_phone_to_users.py
"""add phone to users

Revision ID: b2c3d4e5f6g7
Revises: a1b2c3d4e5f6
Create Date: 2026-06-02 14:00:00
"""
from alembic import op
import sqlalchemy as sa

revision = 'b2c3d4e5f6g7'
down_revision = 'a1b2c3d4e5f6'

def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(20)))

def downgrade():
    op.drop_column('users', 'phone')
```

```bash
# Alembic Commands
alembic init alembic              # Initialize Alembic in your project
alembic revision -m "create users"  # Create new migration
alembic revision --autogenerate -m "add phone"  # Auto-detect model changes!
alembic upgrade head              # Run all pending migrations
alembic downgrade -1              # Rollback last migration
alembic history                   # Show migration history
alembic current                   # Show current DB version
```

> 💡 **Killer Feature: `--autogenerate`** — Alembic compares your SQLAlchemy models to the actual DB and GENERATES the migration automatically!

---

### 4. Prisma Migrate — Modern TypeScript/JavaScript

```typescript
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  username  String   @unique @db.VarChar(50)
  email     String?  @db.VarChar(255)
  phone     String?  @db.VarChar(20)    // ← Add this, run prisma migrate
  orders    Order[]
  createdAt DateTime @default(now())
}

model Order {
  id        Int      @id @default(autoincrement())
  userId    Int
  total     Decimal  @db.Decimal(10, 2)
  status    String   @default("pending")
  user      User     @relation(fields: [userId], references: [id])
  createdAt DateTime @default(now())
}
```

```bash
# Prisma Migrate Commands
npx prisma migrate dev --name add_phone_to_users   # Create + apply migration (dev)
npx prisma migrate deploy                           # Apply migrations (production)
npx prisma migrate reset                            # ⚠️ Reset DB + re-apply all
npx prisma migrate status                           # Check migration status
npx prisma db push                                  # Push schema WITHOUT migration files
```

---

### 5. Tool Comparison Matrix

| Feature | Flyway | Liquibase | Alembic | Prisma | Knex.js |
|---------|--------|-----------|---------|--------|---------|
| **Language** | Java / SQL | Java / XML/YAML/SQL | Python | TypeScript/JS | JavaScript |
| **Format** | SQL files | XML/YAML/JSON/SQL | Python code | Prisma schema | JS/TS code |
| **Rollback** | Paid only | ✅ Built-in | ✅ Built-in | ✅ Built-in | ✅ Built-in |
| **Auto-generate** | ❌ | ✅ (diff) | ✅ (autogenerate) | ✅ (from schema) | ❌ |
| **Multi-DB** | ✅ 20+ | ✅ 50+ | ✅ (via SQLAlchemy) | PG, MySQL, SQLite, SQL Server | PG, MySQL, SQLite, MSSQL |
| **Spring Boot** | ✅ Native | ✅ Native | ❌ | ❌ | ❌ |
| **Complexity** | Simple | Moderate | Moderate | Simple | Simple |
| **Best For** | Java/Spring | Enterprise, multi-DB | Python apps | Node.js/TypeScript | Node.js (Knex) |
| **Pricing** | Free (Community) | Free (Community) | Free (Open Source) | Free (Open Source) | Free (Open Source) |

---

## ⚡ Zero-Downtime Migrations — The Art of Not Breaking Production

### The Problem

```
  ❌ Naive Migration (Causes Downtime):

  1. Take app offline (maintenance mode)
  2. Run ALTER TABLE (locks table for minutes/hours on large tables)
  3. Deploy new app code
  4. Bring app back online
  5. Users angry, revenue lost 💸

  ✅ Zero-Downtime Migration:

  1. App keeps running
  2. Migration happens in background
  3. New code deployed gradually
  4. Users notice NOTHING
  5. You look like a hero 🦸
```

### Strategy 1: Expand-Contract Pattern (The Gold Standard)

```
  Scenario: Rename column "name" → "full_name" in users table

  ❌ WRONG (breaks running app):
  ALTER TABLE users RENAME COLUMN name TO full_name;
  → Old app code: SELECT name FROM users → 💥 ERROR

  ✅ RIGHT (expand → migrate → contract):

  Phase 1: EXPAND (add new column, keep old one)
  ┌─────────────────────────────────────────────────────┐
  │ ALTER TABLE users ADD COLUMN full_name VARCHAR(100); │
  │ UPDATE users SET full_name = name;                   │
  │ -- Create trigger to keep both in sync               │
  │ CREATE TRIGGER sync_name                             │
  │   BEFORE INSERT OR UPDATE ON users                   │
  │   FOR EACH ROW                                       │
  │   SET NEW.full_name = COALESCE(NEW.full_name, NEW.name); │
  └─────────────────────────────────────────────────────┘
  → Old code reads "name" ✅
  → New code reads "full_name" ✅

  Phase 2: MIGRATE (deploy new app code)
  ┌─────────────────────────────────────────────────────┐
  │ Deploy new app version that reads/writes "full_name" │
  │ Old instances still use "name" — both work fine      │
  └─────────────────────────────────────────────────────┘

  Phase 3: CONTRACT (remove old column, once all code updated)
  ┌─────────────────────────────────────────────────────┐
  │ DROP TRIGGER sync_name;                              │
  │ ALTER TABLE users DROP COLUMN name;                  │
  └─────────────────────────────────────────────────────┘
  → Only "full_name" exists. Clean! ✅
```

### Strategy 2: Online Schema Change Tools

For **massive tables** (millions/billions of rows) where ALTER TABLE would lock too long:

```
  Tools that perform ALTER TABLE without locking:

  ┌──────────────────────────────────────────────────────────┐
  │  pt-online-schema-change (Percona Toolkit) — MySQL       │
  │  gh-ost (GitHub Online Schema Transformer) — MySQL       │
  │  pg_repack — PostgreSQL                                  │
  │  ALTER TABLE ... ALGORITHM=INPLACE — MySQL 5.6+          │
  │  ALTER TABLE ... CONCURRENTLY — PostgreSQL (for indexes) │
  └──────────────────────────────────────────────────────────┘

  How gh-ost works (MySQL):

  ┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
  │ Original     │     │ Ghost Table      │     │ Final        │
  │ Table        │────►│ (copy with new   │────►│ Table        │
  │ (users)      │     │  schema)         │     │ (users)      │
  └──────────────┘     └──────────────────┘     └──────────────┘
       │                      ▲
       │  Binlog listener     │
       └──────────────────────┘
       (captures changes during copy)

  1. Creates ghost table with new schema
  2. Copies data in small batches (no lock!)
  3. Binlog listener captures live changes during copy
  4. Atomic RENAME TABLE swap at the end
  5. Total lock time: ~1 second
```

```bash
# gh-ost example — Add column to 100M row table with zero downtime
gh-ost \
  --host=db-primary.company.com \
  --database=myapp \
  --table=users \
  --alter="ADD COLUMN phone VARCHAR(20)" \
  --allow-on-master \
  --chunk-size=1000 \
  --max-load=Threads_running=25 \
  --critical-load=Threads_running=50 \
  --execute

# PostgreSQL — Create index without locking
CREATE INDEX CONCURRENTLY idx_orders_date ON orders(created_at);
-- Normal CREATE INDEX locks the table. CONCURRENTLY doesn't! ⚡
```

### Strategy 3: Backward-Compatible Migrations

```
  Rules for Zero-Downtime Migrations:

  ✅ SAFE Operations (can do anytime):
  ├── ADD COLUMN (with NULL default or server default)
  ├── ADD INDEX (use CONCURRENTLY on PostgreSQL)
  ├── CREATE TABLE
  ├── ADD new value to ENUM
  └── Widen a column (VARCHAR(50) → VARCHAR(100))

  ⚠️ DANGEROUS Operations (need expand-contract):
  ├── RENAME COLUMN
  ├── RENAME TABLE
  ├── CHANGE COLUMN TYPE
  ├── ADD NOT NULL constraint (to existing column with NULLs)
  └── DROP COLUMN (old code may reference it)

  ❌ NEVER Do (without preparation):
  ├── DROP TABLE
  ├── DROP COLUMN (without verifying no code uses it)
  └── SHRINK a column (VARCHAR(100) → VARCHAR(50) — data loss!)
```

---

## 🔄 Rollback Strategies

### Strategy 1: Down Migrations (Recommended for Dev/Staging)

```python
# Alembic — Every migration has upgrade() and downgrade()
def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(20)))

def downgrade():
    op.drop_column('users', 'phone')

# Rollback: alembic downgrade -1
```

### Strategy 2: Forward-Fix (Recommended for Production)

```
  Instead of rolling BACK, roll FORWARD with a fix:

  V5__add_phone.sql           ← This had a bug
  V6__fix_phone_default.sql   ← Fix the bug with a NEW migration

  Why?
  → Rollbacks can fail too (data may have changed)
  → Forward is always safe with version tracking
  → History is preserved — you can see what happened
```

### Strategy 3: Database Snapshots (Emergency)

```
  Before any risky migration:

  1. Take a snapshot/backup:
     PostgreSQL: pg_dump mydb > pre_migration_backup.sql
     MySQL:      mysqldump mydb > pre_migration_backup.sql
     AWS RDS:    Create automated snapshot

  2. Run migration

  3. If disaster:
     Restore from snapshot (RTO depends on DB size)
```

---

## 🧪 Migration Best Practices

### 1. Naming Conventions

```
  Flyway:     V{version}__{description}.sql
              V001__create_users_table.sql
              V002__add_email_to_users.sql

  Liquibase:  YYYYMMDD-N-description.xml
              20260601-1-create-users.xml

  Alembic:    {hash}_{description}.py  (auto-generated)
              a1b2c3_create_users.py

  General Rules:
  ├── Use descriptive names (not V5.sql)
  ├── Use snake_case
  ├── Include the table name when possible
  ├── Keep it short but meaningful
  └── NEVER rename or edit an already-applied migration
```

### 2. One Change Per Migration

```
  ❌ BAD — Kitchen Sink Migration:
  V5__bunch_of_changes.sql
  → Creates 3 tables, adds 5 columns, drops 2 indexes, modifies data
  → If it fails halfway, which part failed?
  → Can't rollback partially

  ✅ GOOD — Atomic Migrations:
  V5__create_products_table.sql
  V6__add_category_to_products.sql
  V7__create_index_products_category.sql
  → Each one is focused, reversible, debuggable
```

### 3. Test Migrations Before Production

```
  Migration Pipeline:

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │   Dev    │───►│   CI     │───►│  Staging  │───►│   Prod   │
  │          │    │          │    │           │    │          │
  │ flyway   │    │ flyway   │    │ flyway    │    │ flyway   │
  │ migrate  │    │ migrate  │    │ migrate   │    │ migrate  │
  │          │    │(test DB) │    │(prod copy)│    │          │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘
                       │
                  Run against a
                  fresh DB + a copy
                  of production data
```

### 4. Handle Large Table Migrations Carefully

```sql
-- ❌ BAD: Backfill 100M rows in one transaction
UPDATE users SET full_name = name;  -- Locks table, fills transaction log

-- ✅ GOOD: Batch updates
DO $$
DECLARE
    batch_size INT := 10000;
    rows_updated INT;
BEGIN
    LOOP
        UPDATE users
        SET full_name = name
        WHERE full_name IS NULL
        AND id IN (
            SELECT id FROM users WHERE full_name IS NULL LIMIT batch_size
        );

        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        RAISE NOTICE 'Updated % rows', rows_updated;
        COMMIT;

        EXIT WHEN rows_updated = 0;

        PERFORM pg_sleep(0.1);  -- Brief pause to reduce load
    END LOOP;
END $$;
```

### 5. Version Control Your Migrations

```
  Migrations are CODE → they belong in Git!

  my-app/
  ├── src/
  ├── db/
  │   └── migrations/
  │       ├── V001__create_users.sql
  │       ├── V002__create_orders.sql
  │       └── V003__add_phone.sql
  ├── .gitignore
  └── README.md

  Rules:
  ├── NEVER modify an applied migration (create a new one instead)
  ├── NEVER delete migrations from version control
  ├── Migrations should be reviewed in PRs like any other code
  ├── Run migrations in CI/CD pipeline automatically
  └── Tag releases that include migration changes
```

---

## 🔥 Cross-Database Migration (Moving Between Database Systems)

### Scenario: MySQL → PostgreSQL

```
  ┌──────────┐                              ┌──────────────┐
  │  MySQL   │ ─── pgloader / AWS DMS ───► │ PostgreSQL   │
  │ (source) │                              │ (target)     │
  └──────────┘                              └──────────────┘

  Challenges:
  ├── Data type differences (TINYINT, ENUM, AUTO_INCREMENT vs SERIAL)
  ├── SQL dialect differences (LIMIT vs FETCH, IFNULL vs COALESCE)
  ├── Stored procedure rewrite (completely different syntax)
  ├── Sequence handling
  └── Character encoding differences
```

```
  Common Type Mappings:

  ┌──────────────────┬────────────────────┬───────────────────┐
  │     MySQL        │   PostgreSQL       │   SQL Server      │
  ├──────────────────┼────────────────────┼───────────────────┤
  │ TINYINT          │ SMALLINT           │ TINYINT           │
  │ INT AUTO_INCR    │ SERIAL / IDENTITY  │ INT IDENTITY      │
  │ DOUBLE           │ DOUBLE PRECISION   │ FLOAT             │
  │ DATETIME         │ TIMESTAMP          │ DATETIME2         │
  │ TEXT             │ TEXT               │ NVARCHAR(MAX)     │
  │ BLOB             │ BYTEA              │ VARBINARY(MAX)    │
  │ ENUM('a','b')    │ VARCHAR + CHECK    │ VARCHAR + CHECK   │
  │ JSON             │ JSONB              │ NVARCHAR(MAX)     │
  │ BOOLEAN          │ BOOLEAN            │ BIT               │
  └──────────────────┴────────────────────┴───────────────────┘
```

### Cross-DB Migration Tools

| Tool | From → To | Type |
|------|-----------|------|
| **pgloader** | MySQL/SQLite → PostgreSQL | Open source, fast |
| **AWS DMS** | Any → Any (AWS) | Managed service |
| **Azure DMS** | Any → Azure SQL | Managed service |
| **Oracle SQL Developer** | Any → Oracle | Free tool |
| **SSMA** | Oracle/MySQL/Access → SQL Server | Free (Microsoft) |
| **ora2pg** | Oracle → PostgreSQL | Open source |

---

## 📊 Migration Decision Flowchart

```
  Need to change the database schema?
  │
  ├── Is it a NEW project?
  │   └── YES → Pick a migration tool, start with V1
  │
  ├── Is it an EXISTING project without migrations?
  │   └── YES → Baseline the current schema (flyway baseline / liquibase changelogSync)
  │             Then add new migrations going forward
  │
  ├── Is the table > 10M rows?
  │   ├── Adding column? → ADD COLUMN with DEFAULT NULL (fast on modern DBs)
  │   ├── Adding index? → CREATE INDEX CONCURRENTLY (PG) or online DDL (MySQL 5.6+)
  │   ├── Modifying column? → Use gh-ost / pt-osc / pg_repack
  │   └── Backfilling data? → Batch updates with commits
  │
  ├── Is it a production deployment?
  │   ├── YES → Test on staging with prod-like data first
  │   ├── Use expand-contract for risky changes
  │   └── Have a rollback plan (snapshot + forward-fix migration)
  │
  └── Is it a column rename/drop?
      └── YES → Expand-contract pattern (3-phase deployment)
```

---

## 🏆 Key Takeaways

| Concept | Remember This |
|---------|---------------|
| **Migrations = Version Control for DB** | Every schema change is a tracked, reproducible script |
| **Never edit applied migrations** | Always create a new migration to fix mistakes |
| **Idempotent migrations** | Use IF NOT EXISTS, IF EXISTS checks |
| **Zero-downtime** | Expand-contract pattern for breaking changes |
| **Large tables** | Use online DDL tools (gh-ost, pt-osc, pg_repack) |
| **Always have rollback** | Down migration (dev) or forward-fix (prod) |
| **Test before production** | Run against staging with real data volumes |
| **One change per migration** | Small, focused, debuggable |
| **Flyway** | SQL files, simple, Spring Boot native |
| **Liquibase** | XML/YAML, rollback support, enterprise-grade |
| **Alembic** | Python, SQLAlchemy, auto-generate |
| **Prisma** | TypeScript, schema-first, modern DX |

---

> 🚀 **Next Up:** [Chapter 5.3 — ORMs & Database Access Layers](./03-ORMs.md) — Hibernate, Entity Framework, SQLAlchemy, Prisma — learn the pros, cons, and the infamous N+1 problem.

---

*Last Updated: June 2026*
