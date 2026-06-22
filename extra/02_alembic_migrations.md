# Alembic — Database Migrations Complete Learning Notes

---

## Table of Contents

1. [What Are Database Migrations?](#1-what-are-database-migrations)
2. [Why Alembic?](#2-why-alembic)
3. [Installation & Setup](#3-installation--setup)
4. [Understanding Alembic's File Structure](#4-understanding-alembics-file-structure)
5. [Your First Migration — Step by Step](#5-your-first-migration--step-by-step)
6. [Autogenerate — Let Alembic Write Migrations for You](#6-autogenerate--let-alembic-write-migrations-for-you)
7. [Writing Manual Migrations](#7-writing-manual-migrations)
8. [Upgrade and Downgrade](#8-upgrade-and-downgrade)
9. [Migration Operations Reference](#9-migration-operations-reference)
10. [Data Migrations (Not Just Schema)](#10-data-migrations-not-just-schema)
11. [The Revision Chain — How Versions Connect](#11-the-revision-chain--how-versions-connect)
12. [Working with Teams (Branching & Merging)](#12-working-with-teams-branching--merging)
13. [Alembic with FastAPI — Complete Setup](#13-alembic-with-fastapi--complete-setup)
14. [Common Scenarios & Recipes](#14-common-scenarios--recipes)
15. [Troubleshooting](#15-troubleshooting)
16. [Best Practices](#16-best-practices)
17. [Mental Model — The Big Picture](#17-mental-model--the-big-picture)

---

## 1. What Are Database Migrations?

### The Problem

You have a running application with a database full of real data. Now you need to:
- Add a new column `phone_number` to the `users` table
- Rename `full_name` to `display_name`
- Create a new `orders` table

**You can't just delete the database and recreate it** — you'd lose all your data.

**You can't manually run ALTER TABLE on every server** — that doesn't scale, isn't tracked, and is error-prone.

### The Solution: Migrations

A **migration** is a versioned script that describes a database change. Think of it like **Git for your database schema**.

```
Version 001: Create users table
Version 002: Add email column to users
Version 003: Create orders table
Version 004: Add phone_number to users
Version 005: Rename full_name → display_name
```

Each migration has two operations:
- **upgrade()** — apply the change (move forward)
- **downgrade()** — undo the change (roll back)

```
v001 ──► v002 ──► v003 ──► v004 ──► v005
         ◄──upgrade──►
         ◄──downgrade──
```

**Every developer runs the same migrations. Every server runs the same migrations. Your database schema is always in a known, reproducible state.**

---

## 2. Why Alembic?

| Feature | Alembic | Django Migrations | Flyway | Liquibase |
|---------|---------|------------------|--------|-----------|
| Works with SQLAlchemy | ✅ Native | ❌ | ❌ | ❌ |
| Auto-detects model changes | ✅ | ✅ | ❌ | ❌ |
| Python-based migrations | ✅ | ✅ | ❌ (SQL/Java) | ❌ (XML/YAML) |
| Manual migration support | ✅ | ✅ | ✅ | ✅ |
| Branching & merging | ✅ | ✅ | ❌ | ❌ |
| Framework independent | ✅ | ❌ (Django only) | ✅ | ✅ |

**Alembic is made by the same team as SQLAlchemy.** It understands your models natively. It's the standard migration tool for any Python project using SQLAlchemy.

---

## 3. Installation & Setup

### Install

```bash
pip install alembic
```

### Initialize Alembic in Your Project

```bash
cd your-project
alembic init alembic
```

This creates:

```
your-project/
├── alembic.ini              ← Main config file (connection string, etc.)
├── alembic/
│   ├── env.py               ← Migration environment (HOW to run migrations)
│   ├── script.py.mako       ← Template for new migration files
│   ├── README                
│   └── versions/            ← Your migration scripts live here
│       └── (empty)
```

---

## 4. Understanding Alembic's File Structure

### `alembic.ini` — The Config File

```ini
# Main config
[alembic]
# Path to migration scripts
script_location = alembic

# Database connection string
# Usually overridden in env.py for security (don't put passwords here)
sqlalchemy.url = sqlite:///./myapp.db

# Template for migration file names
# file_template = %%(year)d_%%(month).2d_%%(day).2d_%%(rev)s_%%(slug)s
```

> **Security tip:** Don't put your real database password in `alembic.ini`. Override it in `env.py` from environment variables.

### `alembic/env.py` — The Brain

This is the most important file. It tells Alembic:
1. How to connect to the database
2. Where to find your models (so autogenerate works)
3. How to run migrations

```python
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context

# This is the Alembic Config object
config = context.config

# Set up Python logging
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# ★ THIS IS THE KEY LINE ★
# Import your models' Base.metadata so Alembic can compare
# your models against the database and auto-detect changes
from myapp.models import Base
target_metadata = Base.metadata

# ... rest of the file handles online/offline migration modes
```

### `alembic/versions/` — Migration Scripts

Each migration is a Python file in this directory:

```
versions/
├── 001_a1b2c3d4e5f6_create_users_table.py
├── 002_f6e5d4c3b2a1_add_email_to_users.py
└── 003_1a2b3c4d5e6f_create_orders_table.py
```

### `alembic/script.py.mako` — Migration Template

This is a [Mako template](https://www.makotemplates.org/) that generates new migration files. You rarely need to edit it, but understanding it helps:

```mako
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}
"""
from alembic import op
import sqlalchemy as sa

# revision identifiers
revision = ${repr(up_revision)}
down_revision = ${repr(down_revision)}

def upgrade():
    ${upgrades if upgrades else "pass"}

def downgrade():
    ${downgrades if downgrades else "pass"}
```

---

## 5. Your First Migration — Step by Step

Let's walk through a complete example from scratch.

### Step 1: Define Your Model

```python
# myapp/models.py
from sqlalchemy import Column, Integer, String, DateTime, create_engine
from sqlalchemy.orm import DeclarativeBase
from datetime import datetime

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    email = Column(String(200), unique=True, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
```

### Step 2: Configure `env.py`

```python
# alembic/env.py — modify the target_metadata line
from myapp.models import Base
target_metadata = Base.metadata
```

### Step 3: Set the Database URL

Option A — In `alembic.ini`:
```ini
sqlalchemy.url = sqlite:///./myapp.db
```

Option B — In `env.py` (recommended for production):
```python
import os
config.set_main_option("sqlalchemy.url", os.environ["DATABASE_URL"])
```

### Step 4: Generate the Migration

```bash
alembic revision --autogenerate -m "create users table"
```

This creates a file like `versions/a1b2c3d4_create_users_table.py`:

```python
"""create users table

Revision ID: a1b2c3d4e5f6
Revises: 
Create Date: 2026-06-03 10:00:00.000000
"""
from alembic import op
import sqlalchemy as sa

revision = 'a1b2c3d4e5f6'
down_revision = None           # First migration — no parent

def upgrade():
    op.create_table('users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('name', sa.String(length=100), nullable=False),
        sa.Column('email', sa.String(length=200), nullable=False),
        sa.Column('created_at', sa.DateTime(), nullable=True),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('email'),
    )

def downgrade():
    op.drop_table('users')
```

### Step 5: Apply the Migration

```bash
alembic upgrade head
```

Output:
```
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> a1b2c3d4e5f6, create users table
```

**Done!** The `users` table now exists in your database.

### Step 6: Verify

```bash
alembic current
# a1b2c3d4e5f6 (head)
```

---

## 6. Autogenerate — Let Alembic Write Migrations for You

### How It Works

When you run `alembic revision --autogenerate`, Alembic:

1. Reads your Python models (`Base.metadata`)
2. Connects to the database and reads the current schema
3. **Compares** them
4. Generates `upgrade()` and `downgrade()` functions for the differences

```
Your Models (Python)          vs.          Your Database (actual)
┌──────────────────┐                    ┌──────────────────┐
│ users             │                    │ users             │
│   id              │                    │   id              │
│   name            │                    │   name            │
│   email           │                    │   email           │
│   phone  ← NEW!  │                    │                   │
│                   │                    │                   │
│ orders  ← NEW!   │                    │                   │
└──────────────────┘                    └──────────────────┘

Diff detected:
  + Add column 'phone' to 'users'
  + Create table 'orders'
```

### What Autogenerate CAN Detect

| Change | Detected? |
|--------|-----------|
| New table | ✅ |
| Dropped table | ✅ |
| New column | ✅ |
| Dropped column | ✅ |
| Column type change | ✅ (with caveats) |
| Column nullable change | ✅ |
| New index | ✅ |
| Dropped index | ✅ |
| New unique constraint | ✅ |
| New foreign key | ✅ |

### What Autogenerate CANNOT Detect

| Change | Detected? | What To Do |
|--------|-----------|------------|
| Column rename | ❌ (sees drop + add) | Write manual migration |
| Table rename | ❌ | Write manual migration |
| Data changes | ❌ | Write data migration |
| Column default changes | ⚠️ Partial | Check manually |
| Check constraints | ❌ (usually) | Write manually |

### Always Review Autogenerated Migrations!

```bash
# Generate
alembic revision --autogenerate -m "add phone to users"

# ALWAYS open the file and review before running!
# Alembic might generate a drop+add instead of a rename
# It might miss subtle changes
# It might generate unnecessary operations
```

---

## 7. Writing Manual Migrations

Sometimes you need to write migrations by hand.

### Create an Empty Migration

```bash
alembic revision -m "add phone column to users"
```

This creates a skeleton file:

```python
"""add phone column to users

Revision ID: b2c3d4e5f6a7
Revises: a1b2c3d4e5f6
Create Date: 2026-06-03 11:00:00.000000
"""
from alembic import op
import sqlalchemy as sa

revision = 'b2c3d4e5f6a7'
down_revision = 'a1b2c3d4e5f6'

def upgrade():
    pass   # ← You write this

def downgrade():
    pass   # ← And this
```

### Fill in the Operations

```python
def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(20), nullable=True))

def downgrade():
    op.drop_column('users', 'phone')
```

---

## 8. Upgrade and Downgrade

### Upgrade Commands

```bash
# Apply ALL pending migrations (go to latest)
alembic upgrade head

# Apply next 1 migration
alembic upgrade +1

# Apply next 2 migrations
alembic upgrade +2

# Go to a specific revision
alembic upgrade a1b2c3d4e5f6

# See what SQL would be generated (dry run)
alembic upgrade head --sql
```

### Downgrade Commands

```bash
# Roll back 1 migration
alembic downgrade -1

# Roll back 2 migrations
alembic downgrade -2

# Roll back to a specific revision
alembic downgrade a1b2c3d4e5f6

# Roll back EVERYTHING (nuclear option ⚠️)
alembic downgrade base
```

### Check Current State

```bash
# What version is the database at?
alembic current
# Output: a1b2c3d4e5f6 (head)

# Show migration history
alembic history
# Output:
# a1b2c3d4e5f6 -> b2c3d4e5f6a7 (head), add phone to users
# <base> -> a1b2c3d4e5f6, create users table

# More verbose history
alembic history --verbose

# Show pending migrations (not yet applied)
alembic heads
```

### The `alembic_version` Table

Alembic tracks which migration the database is at by storing one row in a special table:

```sql
SELECT * FROM alembic_version;
-- Result: 
-- version_num
-- b2c3d4e5f6a7
```

This is how Alembic knows what to upgrade from.

---

## 9. Migration Operations Reference

The `op` object is your toolkit for database changes inside migrations.

### Table Operations

```python
from alembic import op
import sqlalchemy as sa

# Create a table
op.create_table(
    'products',
    sa.Column('id', sa.Integer, primary_key=True),
    sa.Column('name', sa.String(200), nullable=False),
    sa.Column('price', sa.Numeric(10, 2), nullable=False),
    sa.Column('category_id', sa.Integer, sa.ForeignKey('categories.id')),
    sa.Column('created_at', sa.DateTime, server_default=sa.func.now()),
)

# Drop a table
op.drop_table('products')

# Rename a table
op.rename_table('old_name', 'new_name')
```

### Column Operations

```python
# Add a column
op.add_column('users', sa.Column('age', sa.Integer, nullable=True))

# Add a column with a default (for existing rows)
op.add_column('users', sa.Column('is_active', sa.Boolean,
                                  server_default=sa.text('1'),
                                  nullable=False))

# Drop a column
op.drop_column('users', 'age')

# Rename a column
op.alter_column('users', 'name', new_column_name='display_name')

# Change column type
op.alter_column('users', 'age',
    existing_type=sa.Integer(),
    type_=sa.String(10),
    existing_nullable=True
)

# Make column NOT NULL (requires filling existing NULLs first!)
op.execute("UPDATE users SET phone = 'unknown' WHERE phone IS NULL")
op.alter_column('users', 'phone',
    existing_type=sa.String(20),
    nullable=False
)

# Make column nullable
op.alter_column('users', 'phone',
    existing_type=sa.String(20),
    nullable=True
)
```

### Index Operations

```python
# Create an index
op.create_index('ix_users_email', 'users', ['email'], unique=True)

# Create a composite index
op.create_index('ix_users_name_age', 'users', ['name', 'age'])

# Drop an index
op.drop_index('ix_users_email', table_name='users')
```

### Foreign Key Operations

```python
# Add a foreign key
op.create_foreign_key(
    'fk_posts_author_id',      # constraint name
    'posts',                     # source table
    'users',                     # referenced table
    ['author_id'],               # source columns
    ['id'],                      # referenced columns
    ondelete='CASCADE'
)

# Drop a foreign key
op.drop_constraint('fk_posts_author_id', 'posts', type_='foreignkey')
```

### Constraint Operations

```python
# Add unique constraint
op.create_unique_constraint('uq_users_email', 'users', ['email'])

# Add check constraint
op.create_check_constraint('ck_users_age', 'users', 'age >= 0')

# Drop constraint
op.drop_constraint('uq_users_email', 'users', type_='unique')
```

### Raw SQL

```python
# When op functions aren't enough, use raw SQL
op.execute("UPDATE users SET role = 'viewer' WHERE role IS NULL")

# With SQL Server specific syntax
op.execute("EXEC sp_rename 'users.old_col', 'new_col', 'COLUMN'")
```

---

## 10. Data Migrations (Not Just Schema)

Sometimes you need to move or transform data during a migration.

### Example: Split "name" into "first_name" and "last_name"

```python
from alembic import op
import sqlalchemy as sa
from sqlalchemy.sql import table, column

def upgrade():
    # Step 1: Add new columns
    op.add_column('users', sa.Column('first_name', sa.String(50)))
    op.add_column('users', sa.Column('last_name', sa.String(50)))

    # Step 2: Migrate data
    # Create a lightweight table reference (NOT your ORM model!)
    users = table('users',
        column('id', sa.Integer),
        column('name', sa.String),
        column('first_name', sa.String),
        column('last_name', sa.String),
    )

    # Read current data
    conn = op.get_bind()
    results = conn.execute(sa.select(users.c.id, users.c.name))

    for row in results:
        parts = row.name.split(' ', 1)
        first = parts[0]
        last = parts[1] if len(parts) > 1 else ''

        conn.execute(
            users.update()
            .where(users.c.id == row.id)
            .values(first_name=first, last_name=last)
        )

    # Step 3: Make columns non-nullable (now that data exists)
    op.alter_column('users', 'first_name', nullable=False)
    op.alter_column('users', 'last_name', nullable=False)

    # Step 4: Drop old column
    op.drop_column('users', 'name')

def downgrade():
    op.add_column('users', sa.Column('name', sa.String(100)))

    users = table('users',
        column('id', sa.Integer),
        column('name', sa.String),
        column('first_name', sa.String),
        column('last_name', sa.String),
    )

    conn = op.get_bind()
    results = conn.execute(sa.select(users.c.id, users.c.first_name, users.c.last_name))

    for row in results:
        conn.execute(
            users.update()
            .where(users.c.id == row.id)
            .values(name=f"{row.first_name} {row.last_name}")
        )

    op.drop_column('users', 'first_name')
    op.drop_column('users', 'last_name')
```

> **Important:** Never import your ORM models in migrations! Models change over time, but old migrations must always work. Use `table()` and `column()` from `sqlalchemy.sql` instead.

---

## 11. The Revision Chain — How Versions Connect

Migrations form a **linked list**:

```
base ──► rev_001 ──► rev_002 ──► rev_003 ──► rev_004  (head)
          │  down_revision = None
          │
          ├── rev_002
          │     down_revision = 'rev_001'
          │
          ├── rev_003
          │     down_revision = 'rev_002'
          │
          └── rev_004   ← HEAD (latest)
                down_revision = 'rev_003'
```

Each migration file declares:
```python
revision = 'rev_004'           # My own ID
down_revision = 'rev_003'      # My parent (the one before me)
```

**`head`** = the latest migration  
**`base`** = the starting point (no migrations applied)  
**`current`** = where your database currently is

```
base ──► rev_001 ──► rev_002 ──► rev_003 ──► rev_004
                       ▲                        ▲
                    current                    head
```

Running `alembic upgrade head` applies rev_003 and rev_004.

---

## 12. Working with Teams (Branching & Merging)

### The Problem: Parallel Migrations

Alice creates migration `abc123` (adds `phone` column).  
Bob creates migration `def456` (adds `address` column).  
Both have `down_revision = 'rev_002'`.

```
                 ┌── abc123 (Alice: add phone)
rev_002 ────────┤
                 └── def456 (Bob: add address)
```

Now Alembic has **two heads** and can't determine the order.

### The Fix: Merge Migration

```bash
# Check if there are multiple heads
alembic heads
# abc123 (head)
# def456 (head)

# Create a merge migration
alembic merge abc123 def456 -m "merge alice and bob changes"
```

This creates:
```python
revision = 'merged_789'
down_revision = ('abc123', 'def456')   # ← Two parents!

def upgrade():
    pass   # Nothing to do, just merging the chain

def downgrade():
    pass
```

Now the chain looks like:
```
                 ┌── abc123 ──┐
rev_002 ────────┤              ├── merged_789 (head)
                 └── def456 ──┘
```

### Prevention: Always Pull Before Creating Migrations

```bash
git pull origin main
alembic upgrade head           # Make sure DB is at latest
alembic revision --autogenerate -m "my changes"
```

---

## 13. Alembic with FastAPI — Complete Setup

Here's how to set up Alembic in a FastAPI project, step by step.

### Project Structure

```
my_fastapi_app/
├── alembic.ini
├── alembic/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
├── src/
│   ├── database.py         # Engine + SessionLocal
│   ├── models.py           # SQLAlchemy models
│   └── settings.py         # Configuration
└── main.py
```

### `src/database.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase

DATABASE_URL = "sqlite:///./app.db"
# Or from settings: settings.database_url

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)

class Base(DeclarativeBase):
    pass
```

### `src/models.py`

```python
from sqlalchemy import Column, Integer, String, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from datetime import datetime
from src.database import Base

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    email = Column(String(200), unique=True, nullable=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = "posts"
    id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    author_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    author = relationship("User", back_populates="posts")
```

### `alembic/env.py` (Modified)

```python
import os
import sys
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context

# ★ Add the project root to sys.path so we can import our models
sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# ★ Import ALL your models BEFORE setting target_metadata
# This ensures Alembic knows about every table
from src.database import Base
from src.models import User, Post  # Import all models!

target_metadata = Base.metadata

# ★ Override the database URL from environment variable
def get_url():
    return os.environ.get("DATABASE_URL", "sqlite:///./app.db")

def run_migrations_offline():
    url = get_url()
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
    )
    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online():
    configuration = config.get_section(config.config_ini_section)
    configuration["sqlalchemy.url"] = get_url()
    connectable = engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
        )
        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### `alembic.ini` (Modified)

```ini
[alembic]
script_location = alembic
# Leave URL empty — it's set in env.py
sqlalchemy.url =
```

### Startup Command

```bash
# Generate initial migration
alembic revision --autogenerate -m "initial tables"

# Apply
alembic upgrade head

# Start FastAPI
uvicorn main:app --reload
```

### Optional: Run Migrations on App Startup

```python
# main.py
from alembic.config import Config
from alembic import command

def run_migrations():
    alembic_cfg = Config("alembic.ini")
    command.upgrade(alembic_cfg, "head")

# Run migrations before starting the app
run_migrations()
app = FastAPI()
```

---

## 14. Common Scenarios & Recipes

### Adding a Column with Default Value for Existing Rows

```python
def upgrade():
    # For existing rows, server_default fills the value
    op.add_column('users', sa.Column(
        'is_verified',
        sa.Boolean(),
        server_default=sa.text('0'),  # SQL Server: 0 = False
        nullable=False
    ))

def downgrade():
    op.drop_column('users', 'is_verified')
```

### Renaming a Column

```python
def upgrade():
    op.alter_column('users', 'name', new_column_name='display_name')

def downgrade():
    op.alter_column('users', 'display_name', new_column_name='name')
```

### Adding an Enum Column (Python Enum → VARCHAR)

```python
import enum

class UserRole(enum.Enum):
    ADMIN = "admin"
    EDITOR = "editor"
    VIEWER = "viewer"

def upgrade():
    op.add_column('users', sa.Column(
        'role',
        sa.Enum(UserRole, name='userrole'),
        server_default='viewer',
        nullable=False
    ))

def downgrade():
    op.drop_column('users', 'role')
    # For PostgreSQL, also drop the enum type:
    # op.execute("DROP TYPE userrole")
```

### Creating a Table with Foreign Key

```python
def upgrade():
    op.create_table(
        'comments',
        sa.Column('id', sa.Integer, primary_key=True),
        sa.Column('text', sa.Text, nullable=False),
        sa.Column('user_id', sa.Integer, nullable=False),
        sa.Column('post_id', sa.Integer, nullable=False),
        sa.ForeignKeyConstraint(['user_id'], ['users.id'], ondelete='CASCADE'),
        sa.ForeignKeyConstraint(['post_id'], ['posts.id'], ondelete='CASCADE'),
    )

def downgrade():
    op.drop_table('comments')
```

### Making a Column Non-nullable (Safely)

```python
def upgrade():
    # Step 1: Fill in NULL values first!
    op.execute("UPDATE users SET phone = '' WHERE phone IS NULL")

    # Step 2: Now it's safe to make it non-nullable
    op.alter_column('users', 'phone',
        existing_type=sa.String(20),
        nullable=False
    )

def downgrade():
    op.alter_column('users', 'phone',
        existing_type=sa.String(20),
        nullable=True
    )
```

---

## 15. Troubleshooting

### "Target database is not up to date"

```bash
# Your DB is behind. Apply pending migrations first:
alembic upgrade head
# Then try your command again
```

### "Can't locate revision"

```bash
# Database points to a revision that doesn't exist (deleted migration file?)
# Check current state:
alembic current

# If needed, stamp the DB to a known revision (skips running upgrade/downgrade):
alembic stamp head
# or
alembic stamp a1b2c3d4e5f6
```

### "Multiple heads detected"

```bash
# Two developers created migrations from the same parent
alembic heads
# Merge them:
alembic merge heads -m "merge parallel migrations"
```

### Autogenerate Produces Empty Migration

```python
# Check env.py:
# 1. Are you importing ALL your models?
from src.models import User, Post, Comment  # ← Every model!

# 2. Is target_metadata set correctly?
target_metadata = Base.metadata  # Must be the SAME Base your models use!

# 3. Is the database URL correct?
# Run: alembic current  — does it connect?
```

### Migration Fails Halfway

```bash
# If upgrade fails partway:
# 1. Check the error
# 2. Fix the migration file
# 3. Try again:
alembic upgrade head

# If the database is in a broken state:
# Some databases (SQLite) don't support transactional DDL
# You may need to manually fix the schema and stamp:
alembic stamp <last_good_revision>
```

### "No such table: alembic_version"

```bash
# First time running Alembic? Run upgrade to create it:
alembic upgrade head

# Or stamp it if your schema already exists:
alembic stamp head
```

---

## 16. Best Practices

### 1. Always Review Autogenerated Migrations

```bash
alembic revision --autogenerate -m "description"
# OPEN THE FILE AND READ IT before running upgrade
```

### 2. Write Descriptive Messages

```bash
# ❌ Bad
alembic revision --autogenerate -m "update"

# ✅ Good
alembic revision --autogenerate -m "add phone column to users table"
alembic revision --autogenerate -m "create orders table with user foreign key"
```

### 3. Never Edit Applied Migrations

Once a migration has been applied to any database (dev, staging, prod), **never modify it**. Create a new migration instead.

```
# ❌ Don't do this:
# Edit rev_003 (already applied to prod) to add another column

# ✅ Do this:
alembic revision --autogenerate -m "add missing column"  # Creates rev_004
```

### 4. Test Migrations Both Ways

```bash
# Apply
alembic upgrade head

# Roll back
alembic downgrade -1

# Apply again
alembic upgrade head

# If this cycle works, your migration is solid
```

### 5. Keep Migrations Small and Focused

```bash
# ❌ One massive migration that creates 10 tables, adds 5 columns, and migrates data

# ✅ Separate migrations:
# rev_001: create users table
# rev_002: create posts table
# rev_003: add phone to users
# rev_004: migrate data for phone column
```

### 6. Commit Migrations to Git

Migration files are code. They belong in version control alongside your models.

```bash
git add alembic/versions/abc123_add_phone.py
git commit -m "Add phone column migration"
```

### 7. Don't Use `create_all()` in Production

```python
# ❌ In production code:
Base.metadata.create_all(bind=engine)   # Bypasses migration system!

# ✅ In production:
# Just use Alembic: alembic upgrade head
```

---

## 17. Mental Model — The Big Picture

```
         Your Code World                    Your Database World
    ┌─────────────────────┐            ┌──────────────────────┐
    │                     │            │                      │
    │  models.py          │  Alembic   │  Actual Tables       │
    │  ┌──────────┐       │ ═══════►   │  ┌──────────┐       │
    │  │ class User│       │  compares  │  │ users     │       │
    │  │   name    │       │  & syncs   │  │  name     │       │
    │  │   email   │       │            │  │  email    │       │
    │  │   phone ←NEW     │            │  │           │       │
    │  └──────────┘       │            │  └──────────┘       │
    │                     │            │                      │
    └─────────────────────┘            └──────────────────────┘

    alembic revision --autogenerate     →  Creates migration script
    alembic upgrade head                →  Runs migration script on DB

    Migration script (versions/abc.py):
    ┌──────────────────────────────────────────────┐
    │  def upgrade():                               │
    │      op.add_column('users', Column('phone'))  │
    │                                               │
    │  def downgrade():                             │
    │      op.drop_column('users', 'phone')         │
    └──────────────────────────────────────────────┘
```

### The Workflow Loop

```
1. Modify your Python models (add column, new table, etc.)
       │
2. alembic revision --autogenerate -m "description"
       │
3. Review the generated migration file
       │
4. alembic upgrade head
       │
5. Test your app
       │
6. git commit
       │
   Loop back to 1 when you need more changes
```

### Cheat Sheet

| Command | What It Does |
|---------|-------------|
| `alembic init alembic` | Initialize Alembic in project |
| `alembic revision --autogenerate -m "msg"` | Auto-create migration from model changes |
| `alembic revision -m "msg"` | Create empty migration (manual) |
| `alembic upgrade head` | Apply all pending migrations |
| `alembic upgrade +1` | Apply next migration |
| `alembic downgrade -1` | Undo last migration |
| `alembic downgrade base` | Undo ALL migrations |
| `alembic current` | Show current DB revision |
| `alembic history` | Show all migrations |
| `alembic heads` | Show latest revision(s) |
| `alembic stamp head` | Mark DB as up-to-date (no actual migration) |
| `alembic merge heads -m "msg"` | Merge parallel migrations |
| `alembic upgrade head --sql` | Show SQL without running it |

---

*Alembic is the standard migration tool for SQLAlchemy. Once you internalize the workflow (modify model → autogenerate → review → upgrade → commit), it becomes second nature. The key insight: migrations are just Python scripts that call `op.create_table()`, `op.add_column()`, etc. — nothing magical.*
