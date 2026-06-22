# SQLAlchemy & ORM — Complete Learning Notes

---

## Table of Contents

1. [What is an ORM?](#1-what-is-an-orm)
2. [Why SQLAlchemy?](#2-why-sqlalchemy)
3. [The Two Faces of SQLAlchemy](#3-the-two-faces-of-sqlalchemy)
4. [Installation & Setup](#4-installation--setup)
5. [The Engine — Your Database Connection](#5-the-engine--your-database-connection)
6. [The Session — Your Conversation with the DB](#6-the-session--your-conversation-with-the-db)
7. [Defining Models (Tables as Python Classes)](#7-defining-models-tables-as-python-classes)
8. [Column Types Reference](#8-column-types-reference)
9. [CRUD Operations](#9-crud-operations)
10. [Querying — The Heart of SQLAlchemy](#10-querying--the-heart-of-sqlalchemy)
11. [Relationships — Connecting Tables](#11-relationships--connecting-tables)
12. [Enums in Models](#12-enums-in-models)
13. [Indexes and Constraints](#13-indexes-and-constraints)
14. [Events and Hooks](#14-events-and-hooks)
15. [Async SQLAlchemy](#15-async-sqlalchemy)
16. [Common Patterns & Best Practices](#16-common-patterns--best-practices)
17. [Debugging & Troubleshooting](#17-debugging--troubleshooting)
18. [Mental Model — How It All Fits Together](#18-mental-model--how-it-all-fits-together)

---

## 1. What is an ORM?

### The Problem

Imagine you're building a Python app that needs to store users in a database. Without an ORM, you'd write raw SQL everywhere:

```python
import pyodbc

conn = pyodbc.connect("Driver={SQL Server};Server=localhost;Database=mydb;")
cursor = conn.cursor()

# Creating a user
cursor.execute(
    "INSERT INTO users (name, email, age) VALUES (?, ?, ?)",
    ("Ritesh", "ritesh@example.com", 28)
)
conn.commit()

# Fetching a user
cursor.execute("SELECT * FROM users WHERE id = ?", (1,))
row = cursor.fetchone()
# row is a tuple: (1, 'Ritesh', 'ritesh@example.com', 28)
# Which index is email? Was it 2 or 3? 🤔
```

**Problems with this approach:**
- You're mixing Python logic with SQL strings
- Rows come back as tuples — you must remember column positions
- No validation — typos in SQL crash at runtime
- SQL injection risk if you forget parameterized queries
- Switching databases (e.g., SQLite → PostgreSQL) means rewriting SQL
- No way to represent relationships cleanly

### The Solution: ORM

**ORM = Object-Relational Mapping.** It maps:

| Database Concept | Python Concept |
|------------------|----------------|
| Table            | Class          |
| Row              | Object (instance) |
| Column           | Attribute      |
| Foreign Key      | Reference to another object |
| SQL Query        | Python method call |

With an ORM, the same code becomes:

```python
# Define once
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    email = Column(String(200))
    age = Column(Integer)

# Create a user
user = User(name="Ritesh", email="ritesh@example.com", age=28)
session.add(user)
session.commit()

# Fetch a user
user = session.query(User).filter(User.id == 1).first()
print(user.name)   # "Ritesh"  ← not row[1], but user.name!
print(user.email)   # "ritesh@example.com"
```

**You're writing Python, not SQL.** The ORM translates it to SQL behind the scenes.

---

## 2. Why SQLAlchemy?

SQLAlchemy is the **most popular and powerful ORM for Python**. Here's why:

| Feature | SQLAlchemy | Django ORM | Peewee |
|---------|-----------|------------|--------|
| Standalone (no framework) | ✅ | ❌ (tied to Django) | ✅ |
| Raw SQL when needed | ✅ | Partial | Partial |
| Complex queries | ✅ Excellent | Limited | Limited |
| Async support | ✅ (v2.0+) | ✅ (v4.1+) | ❌ |
| Database agnostic | ✅ | ✅ | ✅ |
| Production battle-tested | ✅ (15+ years) | ✅ | ⚠️ |

**Key philosophy:** SQLAlchemy doesn't hide SQL from you — it gives you a Python API that *thinks* in SQL. When you need raw SQL, you can always drop down to it.

---

## 3. The Two Faces of SQLAlchemy

SQLAlchemy has **two layers**, and understanding this is crucial:

```
┌─────────────────────────────────────────────┐
│              SQLAlchemy ORM                  │  ← High level (classes, objects)
│   (Models, Sessions, Relationships)          │
├─────────────────────────────────────────────┤
│           SQLAlchemy Core                    │  ← Low level (tables, SQL expressions)
│   (Engine, Connection, Table, select())      │
├─────────────────────────────────────────────┤
│              DBAPI (pyodbc, psycopg2)        │  ← Database driver
├─────────────────────────────────────────────┤
│           Actual Database                    │  ← SQL Server, PostgreSQL, etc.
│              (SQL Server)                    │
└─────────────────────────────────────────────┘
```

### Core (Low Level)
```python
from sqlalchemy import create_engine, Table, Column, Integer, String, MetaData, select

metadata = MetaData()
users = Table('users', metadata,
    Column('id', Integer, primary_key=True),
    Column('name', String(100)),
)

# Core-style query (SQL expression language)
stmt = select(users).where(users.c.name == "Ritesh")
```

### ORM (High Level) — What you'll use 95% of the time
```python
from sqlalchemy.orm import DeclarativeBase, Session

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(100))

# ORM-style query
user = session.query(User).filter(User.name == "Ritesh").first()
```

**Rule of thumb:** Use the ORM for everything. Drop to Core only for complex bulk operations or performance-critical queries.

---

## 4. Installation & Setup

### Install SQLAlchemy + Your Database Driver

```bash
# SQLAlchemy itself
pip install sqlalchemy

# Database drivers (pick one):
pip install psycopg2-binary    # PostgreSQL
pip install pyodbc              # SQL Server
pip install pymysql             # MySQL
# SQLite comes built-in with Python — no extra driver needed
```

### Connection URLs by Database

```python
# SQLite (file-based, great for learning/testing)
"sqlite:///./mydb.sqlite3"

# SQLite (in-memory, perfect for unit tests)
"sqlite:///:memory:"

# PostgreSQL
"postgresql://user:password@localhost:5432/mydb"

# SQL Server
"mssql+pyodbc://user:password@localhost/mydb?driver=ODBC+Driver+17+for+SQL+Server"

# MySQL
"mysql+pymysql://user:password@localhost:3306/mydb"
```

---

## 5. The Engine — Your Database Connection

The **Engine** is the starting point. Think of it as the **connection factory** — it knows HOW to connect to your database and manages a **pool of connections** for efficiency.

```python
from sqlalchemy import create_engine

# Create an engine
engine = create_engine(
    "sqlite:///./myapp.db",
    echo=True,          # Print all SQL to console (great for learning!)
    pool_size=5,         # Keep 5 connections ready
    max_overflow=10,     # Allow up to 10 extra connections under load
    pool_pre_ping=True,  # Test connections before using them
)
```

### What `echo=True` Looks Like

When you turn on `echo=True`, every SQL statement appears in your console:

```
2026-06-03 10:00:00 INFO sqlalchemy.engine.Engine
SELECT users.id, users.name, users.email
FROM users
WHERE users.id = ?
2026-06-03 10:00:00 INFO sqlalchemy.engine.Engine [generated in 0.00012s] (1,)
```

**This is your best debugging tool when learning.** Turn it on, run your code, and see exactly what SQL is generated.

### Connection Pooling Explained

```
Your App:    request1  request2  request3  request4  request5
                │         │         │         │         │
Engine Pool: [conn1]  [conn2]  [conn3]  [conn4]  [conn5]
                │         │         │         │         │
Database:    ════════════════════════════════════════════════
```

Without pooling, every request would open a new connection (slow). The pool **reuses connections**, which is much faster.

---

## 6. The Session — Your Conversation with the DB

If the Engine is the connection factory, the **Session** is your **workspace** — a temporary holding area where you stage changes before committing them to the database.

### Mental Model: Session as a Shopping Cart

```
Session (your cart):
  ┌──────────────────────────────────────┐
  │  + new User("Ritesh")    [NEW]       │  ← added but not saved yet
  │  ~ user.name = "Updated" [DIRTY]     │  ← modified but not saved
  │  - old_user               [DELETED]  │  ← marked for deletion
  └──────────────────────────────────────┘
                    │
                commit()  ← "checkout" — all changes go to DB at once
                    │
              ┌─────────┐
              │ Database │
              └─────────┘
```

### Creating a Session

```python
from sqlalchemy.orm import sessionmaker

# Create a session factory (do this ONCE)
SessionLocal = sessionmaker(bind=engine)

# Create a session instance (do this PER REQUEST / PER OPERATION)
session = SessionLocal()

try:
    # ... do stuff ...
    session.commit()      # Save all changes
except Exception:
    session.rollback()    # Undo all changes on error
finally:
    session.close()       # Return connection to pool
```

### The Modern Way — Context Manager

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    with session.begin():        # Auto-commit on success, auto-rollback on error
        user = User(name="Ritesh")
        session.add(user)
    # commit() happens automatically here
# close() happens automatically here
```

### FastAPI Pattern — Dependency Injection

```python
from sqlalchemy.orm import Session

def get_db():
    """Yield a session per request, auto-close when done."""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# In your route:
@app.get("/users/{id}")
def get_user(id: int, db: Session = Depends(get_db)):
    return db.query(User).filter(User.id == id).first()
```

### Session States — What Happens to Objects

```python
user = User(name="Ritesh")
# State: TRANSIENT — exists in Python, DB doesn't know about it

session.add(user)
# State: PENDING — session knows about it, not yet in DB

session.commit()
# State: PERSISTENT — exists in both Python and DB
# user.id is now populated (e.g., 1)

session.expunge(user)
# State: DETACHED — removed from session, but has DB identity

session.delete(user)
session.commit()
# Object is deleted from DB
```

```
                    add()          commit()
  TRANSIENT ──────────────► PENDING ──────────► PERSISTENT
                                                    │
                                          expunge() │ delete()+commit()
                                                    │
                                               DETACHED
```

---

## 7. Defining Models (Tables as Python Classes)

### SQLAlchemy 2.0 Style (Recommended)

```python
from sqlalchemy import String, Integer, Float, Text, Boolean, DateTime
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from datetime import datetime
import uuid

# Step 1: Create a base class
class Base(DeclarativeBase):
    pass

# Step 2: Define your models
class User(Base):
    __tablename__ = "users"       # ← actual table name in DB

    # Primary key — auto-incrementing integer
    id: Mapped[int] = mapped_column(primary_key=True)

    # Required string (NOT NULL in DB)
    name: Mapped[str] = mapped_column(String(100))

    # Required unique string
    email: Mapped[str] = mapped_column(String(200), unique=True, index=True)

    # Optional field (nullable)
    bio: Mapped[str | None] = mapped_column(Text, default=None)

    # Field with default value
    is_active: Mapped[bool] = mapped_column(default=True)

    # Timestamp with server-side default
    created_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.utcnow
    )

    def __repr__(self):
        return f"<User(id={self.id}, name='{self.name}')>"
```

### Legacy Style (Still Works, Still Common)

```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(100), nullable=False)
    email = Column(String(200), unique=True, nullable=False, index=True)
    bio = Column(Text, nullable=True)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
```

> **Both styles work.** The 2.0 style with `Mapped[]` gives you better type hints and IDE support. Most existing tutorials and projects use the legacy style. Learn both.

### UUID Primary Keys

```python
import uuid
from sqlalchemy.dialects.mssql import UNIQUEIDENTIFIER  # SQL Server
# from sqlalchemy.dialects.postgresql import UUID         # PostgreSQL

class InterviewSession(Base):
    __tablename__ = "interview_sessions"

    id = Column(
        UNIQUEIDENTIFIER,
        primary_key=True,
        default=uuid.uuid4    # Python generates the UUID
    )
```

### Creating Tables from Models

```python
# Create ALL tables defined via Base
Base.metadata.create_all(bind=engine)

# Drop ALL tables (careful!)
Base.metadata.drop_all(bind=engine)
```

> **In production, you DON'T use `create_all()`.** You use **Alembic** for migrations (see the Alembic notes).

---

## 8. Column Types Reference

| Python Type | SQLAlchemy Type | SQL Result | Notes |
|-------------|----------------|------------|-------|
| `int` | `Integer` | `INT` | Standard integer |
| `int` | `BigInteger` | `BIGINT` | For very large numbers |
| `str` | `String(50)` | `VARCHAR(50)` | Fixed max length |
| `str` | `Text` | `TEXT` / `NVARCHAR(MAX)` | Unlimited length |
| `float` | `Float` | `FLOAT` | Floating point |
| `float` | `Numeric(10,2)` | `DECIMAL(10,2)` | Exact precision (money!) |
| `bool` | `Boolean` | `BIT` / `BOOLEAN` | True/False |
| `datetime` | `DateTime` | `DATETIME` | Date + time |
| `date` | `Date` | `DATE` | Date only |
| `time` | `Time` | `TIME` | Time only |
| `dict` | `JSON` | `NVARCHAR(MAX)` / `JSON` | JSON data |
| `bytes` | `LargeBinary` | `VARBINARY(MAX)` | Binary data |
| `uuid` | `Uuid` | `UNIQUEIDENTIFIER` | UUID / GUID |
| `enum` | `Enum(MyEnum)` | `VARCHAR` | Python enum stored as string |

### Column Options

```python
Column(String(100),
    primary_key=False,        # Is this the primary key?
    nullable=True,            # Can it be NULL? (default: True)
    unique=False,             # Must all values be unique?
    index=False,              # Create a database index?
    default="hello",          # Python-side default value
    server_default="hello",   # Database-side default (in SQL)
    onupdate=datetime.utcnow, # Auto-update on every UPDATE
)
```

### `default` vs `server_default`

```python
# default — Python generates the value before INSERT
created_at = Column(DateTime, default=datetime.utcnow)
# Python runs datetime.utcnow() and puts the value in the INSERT SQL

# server_default — Database generates the value
created_at = Column(DateTime, server_default=func.now())
# SQL: INSERT INTO ... DEFAULT VALUES → DB fills in GETDATE()

# Why does it matter?
# - `default`: Works everywhere, but value comes from Python's clock
# - `server_default`: Value comes from DB's clock (more reliable in distributed systems)
# - `server_default`: Shows up in CREATE TABLE SQL (Alembic sees it)
```

---

## 9. CRUD Operations

### Create (INSERT)

```python
# Single insert
user = User(name="Ritesh", email="ritesh@example.com")
session.add(user)
session.commit()

# After commit, the auto-generated id is available:
print(user.id)  # e.g., 1

# Bulk insert (faster for many records)
users = [
    User(name="Alice", email="alice@example.com"),
    User(name="Bob", email="bob@example.com"),
    User(name="Charlie", email="charlie@example.com"),
]
session.add_all(users)
session.commit()
```

### Read (SELECT)

```python
# Get by primary key (most efficient)
user = session.get(User, 1)                     # SELECT ... WHERE id = 1

# Get first match
user = session.query(User).filter(User.name == "Ritesh").first()

# Get ALL users
all_users = session.query(User).all()           # Returns a list

# Get one (raises if 0 or 2+ results)
user = session.query(User).filter(User.id == 1).one()

# Get one or None
user = session.query(User).filter(User.id == 999).one_or_none()

# Count
total = session.query(User).count()

# Check existence
exists = session.query(User).filter(User.email == "x@y.com").first() is not None
```

### Update (UPDATE)

```python
# Method 1: Fetch and modify (simple, common)
user = session.get(User, 1)
user.name = "Updated Name"
user.email = "new@example.com"
session.commit()
# SQLAlchemy detects the change and generates: UPDATE users SET name=?, email=? WHERE id=1

# Method 2: Bulk update (efficient, no fetch needed)
session.query(User).filter(User.is_active == False).update(
    {"is_active": True},
    synchronize_session="fetch"    # Keep session in sync
)
session.commit()
```

### Delete (DELETE)

```python
# Method 1: Fetch and delete
user = session.get(User, 1)
session.delete(user)
session.commit()

# Method 2: Bulk delete
session.query(User).filter(User.is_active == False).delete()
session.commit()
```

### The flush() vs commit() Distinction

```python
user = User(name="Ritesh")
session.add(user)

session.flush()
# SQL is SENT to the database (INSERT runs)
# user.id is now populated
# BUT the transaction is NOT committed — changes can still be rolled back

session.commit()
# Transaction is COMMITTED — changes are permanent
# Internally calls flush() first if you haven't already
```

**When to use `flush()`:** When you need auto-generated values (like `id`) before committing. For example, creating a parent and child in the same transaction:

```python
parent = Parent(name="Family")
session.add(parent)
session.flush()           # parent.id is now available

child = Child(name="Kid", parent_id=parent.id)
session.add(child)
session.commit()          # Both saved in one transaction
```

---

## 10. Querying — The Heart of SQLAlchemy

### Filter Conditions (WHERE clauses)

```python
from sqlalchemy import and_, or_, not_, func

# Equality
session.query(User).filter(User.name == "Ritesh")

# Not equal
session.query(User).filter(User.name != "Ritesh")

# Greater than / less than
session.query(User).filter(User.age > 25)
session.query(User).filter(User.age >= 25)
session.query(User).filter(User.age < 30)
session.query(User).filter(User.age.between(25, 30))

# LIKE (pattern matching)
session.query(User).filter(User.name.like("%rite%"))        # Case-sensitive
session.query(User).filter(User.name.ilike("%rite%"))       # Case-insensitive

# IN
session.query(User).filter(User.id.in_([1, 2, 3]))

# NOT IN
session.query(User).filter(User.id.not_in([1, 2, 3]))

# IS NULL / IS NOT NULL
session.query(User).filter(User.bio.is_(None))
session.query(User).filter(User.bio.is_not(None))

# AND (multiple filters)
session.query(User).filter(
    User.is_active == True,
    User.age > 25                   # Implicit AND
)
# OR explicit:
session.query(User).filter(
    and_(User.is_active == True, User.age > 25)
)

# OR
session.query(User).filter(
    or_(User.name == "Ritesh", User.name == "Alice")
)

# NOT
session.query(User).filter(not_(User.is_active))

# Chaining filters (each .filter() adds AND)
session.query(User) \
    .filter(User.is_active == True) \
    .filter(User.age > 25) \
    .filter(User.name.like("R%"))
```

### Ordering, Limiting, Offsetting

```python
# ORDER BY
session.query(User).order_by(User.name)              # ASC (default)
session.query(User).order_by(User.name.desc())        # DESC
session.query(User).order_by(User.name, User.age.desc())  # Multi-column

# LIMIT
session.query(User).limit(10)

# OFFSET (for pagination)
session.query(User).offset(20).limit(10)              # Skip 20, take 10

# Pagination helper
page = 3
per_page = 10
users = session.query(User) \
    .order_by(User.id) \
    .offset((page - 1) * per_page) \
    .limit(per_page) \
    .all()
```

### Aggregations (GROUP BY)

```python
from sqlalchemy import func

# COUNT
total = session.query(func.count(User.id)).scalar()

# SUM, AVG, MIN, MAX
avg_age = session.query(func.avg(User.age)).scalar()

# GROUP BY
from sqlalchemy import func
results = session.query(
    User.is_active,
    func.count(User.id).label("count")
).group_by(User.is_active).all()

# Result: [(True, 42), (False, 8)]
for is_active, count in results:
    print(f"Active={is_active}: {count} users")

# HAVING (filter on aggregates)
results = session.query(
    User.department,
    func.count(User.id).label("count")
).group_by(User.department) \
 .having(func.count(User.id) > 5) \
 .all()
```

### Selecting Specific Columns

```python
# Select only name and email (not full objects)
results = session.query(User.name, User.email).all()
# Returns list of tuples: [("Ritesh", "r@x.com"), ("Alice", "a@x.com")]

# Named tuples — access by attribute
for row in results:
    print(row.name, row.email)

# With labels
results = session.query(
    User.name,
    func.count(User.id).label("total")
).group_by(User.name).all()
```

### SQLAlchemy 2.0 Query Style (Modern)

```python
from sqlalchemy import select

# Old style (still works)
users = session.query(User).filter(User.name == "Ritesh").all()

# New 2.0 style (recommended for new code)
stmt = select(User).where(User.name == "Ritesh")
users = session.execute(stmt).scalars().all()

# The 2.0 style separates statement construction from execution
# This makes it easier to build queries dynamically
stmt = select(User)
if name_filter:
    stmt = stmt.where(User.name == name_filter)
if active_only:
    stmt = stmt.where(User.is_active == True)
result = session.execute(stmt).scalars().all()
```

---

## 11. Relationships — Connecting Tables

This is where ORM really shines. Instead of writing JOIN queries, you navigate between objects like Python attributes.

### One-to-Many (Most Common)

**Scenario:** One User has many Posts.

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(100))

    # This is NOT a database column — it's a Python-only attribute
    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = "posts"
    id = Column(Integer, primary_key=True)
    title = Column(String(200))
    content = Column(Text)

    # THIS is the actual database column (foreign key)
    author_id = Column(Integer, ForeignKey("users.id"), nullable=False)

    # Python-only reverse relationship
    author = relationship("User", back_populates="posts")
```

**Using it:**

```python
# Create a user with posts
user = User(name="Ritesh")
user.posts.append(Post(title="First Post", content="Hello!"))
user.posts.append(Post(title="Second Post", content="World!"))
session.add(user)
session.commit()

# Navigate the relationship
user = session.get(User, 1)
for post in user.posts:                  # ← Lazy loads posts automatically
    print(f"{post.title} by {post.author.name}")

# Reverse navigation
post = session.get(Post, 1)
print(post.author.name)                   # ← "Ritesh"
```

### How `back_populates` Works

```
User.posts ◄──── back_populates ────► Post.author

When you do:        user.posts.append(post)
SQLAlchemy also:    sets post.author = user  (automatically!)
```

If you only need one direction, use `backref` (shortcut):

```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    posts = relationship("Post", backref="author")
    # This auto-creates Post.author without defining it in Post class
```

### One-to-One

Same as one-to-many, but add `uselist=False`:

```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    profile = relationship("Profile", back_populates="user", uselist=False)

class Profile(Base):
    __tablename__ = "profiles"
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), unique=True)
    bio = Column(Text)
    user = relationship("User", back_populates="profile")

# Usage
user.profile.bio = "I love Python"      # ← single object, not a list
```

### Many-to-Many

**Scenario:** Students enroll in Courses. One student → many courses. One course → many students.

```python
# Association table (no Python class needed)
student_courses = Table(
    "student_courses",
    Base.metadata,
    Column("student_id", Integer, ForeignKey("students.id"), primary_key=True),
    Column("course_id", Integer, ForeignKey("courses.id"), primary_key=True),
)

class Student(Base):
    __tablename__ = "students"
    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    courses = relationship("Course", secondary=student_courses, back_populates="students")

class Course(Base):
    __tablename__ = "courses"
    id = Column(Integer, primary_key=True)
    title = Column(String(200))
    students = relationship("Student", secondary=student_courses, back_populates="courses")

# Usage
student = Student(name="Ritesh")
python_course = Course(title="Python 101")
student.courses.append(python_course)
session.add(student)
session.commit()

# Navigate both directions
print(student.courses)         # [<Course: Python 101>]
print(python_course.students)  # [<Student: Ritesh>]
```

### Lazy Loading vs Eager Loading

**Lazy loading** (default): Related objects are loaded only when you access them.

```python
user = session.get(User, 1)    # SQL: SELECT * FROM users WHERE id = 1
print(user.posts)               # SQL: SELECT * FROM posts WHERE author_id = 1
# Second query only runs when you access .posts
```

**Problem: N+1 Query Issue**

```python
users = session.query(User).all()     # 1 query: SELECT * FROM users (100 users)
for user in users:
    print(user.posts)                  # 100 queries! One per user!
# Total: 101 queries 😱
```

**Solution: Eager Loading**

```python
from sqlalchemy.orm import joinedload, selectinload

# Option 1: JOIN (one big query)
users = session.query(User).options(joinedload(User.posts)).all()
# SQL: SELECT * FROM users LEFT JOIN posts ON ...

# Option 2: Separate SELECT (two queries)
users = session.query(User).options(selectinload(User.posts)).all()
# SQL: SELECT * FROM users
# SQL: SELECT * FROM posts WHERE author_id IN (1, 2, 3, ...)
```

**When to use which:**
- `joinedload` → Small related data, one-to-one
- `selectinload` → One-to-many (avoids cartesian explosion)

### Cascade — What Happens When You Delete

```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    posts = relationship("Post", back_populates="author",
                         cascade="all, delete-orphan")
    # "all, delete-orphan" means:
    # - When you delete a user, all their posts are deleted too
    # - When you remove a post from user.posts, it's deleted from DB

# Without cascade, deleting a user with posts → foreign key error!
```

---

## 12. Enums in Models

Python enums map cleanly to database columns:

```python
import enum

class UserRole(enum.Enum):
    ADMIN = "admin"
    EDITOR = "editor"
    VIEWER = "viewer"

class OrderStatus(enum.Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    role = Column(Enum(UserRole), default=UserRole.VIEWER, nullable=False)

class Order(Base):
    __tablename__ = "orders"
    id = Column(Integer, primary_key=True)
    status = Column(Enum(OrderStatus), default=OrderStatus.PENDING)

# Usage
user = User(name="Ritesh", role=UserRole.ADMIN)
session.add(user)
session.commit()

# Querying with enums
admins = session.query(User).filter(User.role == UserRole.ADMIN).all()

# The DB stores the string value: "admin", "editor", etc.
```

---

## 13. Indexes and Constraints

```python
from sqlalchemy import UniqueConstraint, Index, CheckConstraint

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String(200), unique=True)        # Unique constraint
    username = Column(String(50), index=True)        # Simple index
    first_name = Column(String(50))
    last_name = Column(String(50))
    age = Column(Integer)

    __table_args__ = (
        # Composite unique constraint (both together must be unique)
        UniqueConstraint("first_name", "last_name", name="uq_user_fullname"),

        # Composite index
        Index("ix_user_name", "first_name", "last_name"),

        # Check constraint
        CheckConstraint("age >= 0 AND age <= 150", name="ck_user_age"),
    )
```

---

## 14. Events and Hooks

SQLAlchemy lets you run code before/after database operations:

```python
from sqlalchemy import event

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    updated_at = Column(DateTime)

# Before any UPDATE on User, set updated_at
@event.listens_for(User, "before_update")
def set_updated_at(mapper, connection, target):
    target.updated_at = datetime.utcnow()

# Before any INSERT on User, normalize name
@event.listens_for(User, "before_insert")
def normalize_name(mapper, connection, target):
    target.name = target.name.strip().title()

# After INSERT
@event.listens_for(User, "after_insert")
def after_user_created(mapper, connection, target):
    print(f"New user created: {target.name} (id={target.id})")
```

---

## 15. Async SQLAlchemy

For async frameworks like FastAPI, SQLAlchemy supports async sessions:

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy import select

# Async engine (note the +aiosqlite / +asyncpg driver)
engine = create_async_engine(
    "sqlite+aiosqlite:///./myapp.db",     # SQLite
    # "postgresql+asyncpg://user:pass@localhost/db",  # PostgreSQL
    echo=True,
)

# Async session factory
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

# Usage in an async function
async def get_user(user_id: int):
    async with async_session() as session:
        result = await session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

async def create_user(name: str, email: str):
    async with async_session() as session:
        async with session.begin():
            user = User(name=name, email=email)
            session.add(user)
        # Auto-commits on exit

# FastAPI dependency
async def get_db():
    async with async_session() as session:
        yield session

@app.get("/users/{id}")
async def read_user(id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == id))
    user = result.scalar_one_or_none()
    if not user:
        raise HTTPException(404)
    return user
```

> **Note:** With async, you MUST use `await session.execute()` and the 2.0 `select()` style. The old `session.query()` doesn't work with async.

### Async Relationship Access

```python
# Lazy loading doesn't work with async!
# This will ERROR:
user = await session.get(User, 1)
print(user.posts)  # ❌ MissingGreenlet error

# Solution 1: Eager loading
result = await session.execute(
    select(User).options(selectinload(User.posts)).where(User.id == 1)
)
user = result.scalar_one()
print(user.posts)  # ✅ Already loaded

# Solution 2: Awaitable attributes (SQLAlchemy 2.0+)
user = await session.get(User, 1)
await session.refresh(user, ["posts"])  # Explicitly load
print(user.posts)  # ✅
```

---

## 16. Common Patterns & Best Practices

### Mixin Classes (Reusable Columns)

```python
from datetime import datetime

class TimestampMixin:
    """Add created_at and updated_at to any model."""
    created_at = Column(DateTime, default=datetime.utcnow, nullable=False)
    updated_at = Column(DateTime, default=datetime.utcnow,
                        onupdate=datetime.utcnow, nullable=False)

class SoftDeleteMixin:
    """Soft delete instead of hard delete."""
    is_deleted = Column(Boolean, default=False)
    deleted_at = Column(DateTime, nullable=True)

class User(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    # Automatically has: created_at, updated_at, is_deleted, deleted_at
```

### Repository Pattern

```python
class UserRepository:
    def __init__(self, session: Session):
        self.session = session

    def get_by_id(self, user_id: int) -> User | None:
        return self.session.get(User, user_id)

    def get_by_email(self, email: str) -> User | None:
        return self.session.query(User).filter(User.email == email).first()

    def get_active_users(self) -> list[User]:
        return self.session.query(User).filter(User.is_active == True).all()

    def create(self, name: str, email: str) -> User:
        user = User(name=name, email=email)
        self.session.add(user)
        self.session.flush()   # Get the ID without committing
        return user

    def search(self, query: str) -> list[User]:
        return self.session.query(User).filter(
            or_(
                User.name.ilike(f"%{query}%"),
                User.email.ilike(f"%{query}%")
            )
        ).all()
```

### Upsert (Insert or Update)

```python
from sqlalchemy.dialects.postgresql import insert  # PostgreSQL
# from sqlalchemy.dialects.sqlite import insert     # SQLite

stmt = insert(User).values(email="r@x.com", name="Ritesh")
stmt = stmt.on_conflict_do_update(
    index_elements=["email"],     # Conflict on this column
    set_={"name": stmt.excluded.name}   # Update these columns
)
session.execute(stmt)
session.commit()
```

### Raw SQL When Needed

```python
from sqlalchemy import text

# Simple raw query
result = session.execute(text("SELECT * FROM users WHERE age > :age"), {"age": 25})
for row in result:
    print(row.name, row.email)

# Raw query returning ORM objects
users = session.query(User).from_statement(
    text("SELECT * FROM users WHERE name LIKE :pattern")
).params(pattern="%Ritesh%").all()
```

---

## 17. Debugging & Troubleshooting

### Enable SQL Logging

```python
# Method 1: In engine creation
engine = create_engine("sqlite:///mydb.db", echo=True)

# Method 2: Via Python logging
import logging
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
```

### Common Errors and Fixes

#### "DetachedInstanceError"
```python
# ❌ Error: accessing relationship after session is closed
user = session.query(User).first()
session.close()
print(user.posts)  # DetachedInstanceError!

# ✅ Fix: eager load before closing
user = session.query(User).options(joinedload(User.posts)).first()
session.close()
print(user.posts)  # Works!

# ✅ Fix: or set expire_on_commit=False
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)
```

#### "IntegrityError: UNIQUE constraint failed"
```python
# ❌ Trying to insert duplicate unique value
try:
    session.add(User(email="existing@email.com"))
    session.commit()
except IntegrityError:
    session.rollback()
    # Handle duplicate
```

#### "SAWarning: relationship X will copy column Y"
```python
# Usually means overlapping foreign keys
# Fix: add foreign_keys parameter to relationship
posts = relationship("Post", foreign_keys="[Post.author_id]")
```

### Inspecting Generated SQL Without Executing

```python
from sqlalchemy.dialects import sqlite  # or postgresql, mssql

stmt = select(User).where(User.name == "Ritesh")
print(stmt.compile(dialect=sqlite.dialect(), compile_kwargs={"literal_binds": True}))
# Output: SELECT users.id, users.name FROM users WHERE users.name = 'Ritesh'
```

---

## 18. Mental Model — How It All Fits Together

```
Your Python Code
    │
    ▼
┌──────────────────────────────────────────────────────────┐
│  MODEL (User class)           SCHEMA (Pydantic)          │
│  ┌──────────────────┐        ┌──────────────────┐        │
│  │ id: int (PK)     │        │ name: str        │        │
│  │ name: str         │◄──────│ email: str       │        │
│  │ email: str        │  map  │                   │        │
│  │ posts: [Post]     │        └──────────────────┘        │
│  └──────────────────┘                                     │
│           │                                                │
│     session.add(user)                                      │
│           │                                                │
│  ┌──────────────────┐                                     │
│  │    SESSION        │ ← tracks new/dirty/deleted objects │
│  │  (Unit of Work)   │                                    │
│  └──────────────────┘                                     │
│           │                                                │
│     session.commit()                                       │
│           │                                                │
│  ┌──────────────────┐                                     │
│  │     ENGINE        │ ← connection pool                  │
│  │  (Connection Pool)│                                    │
│  └──────────────────┘                                     │
│           │                                                │
│    SQL Statement      INSERT INTO users (name, email)      │
│           │            VALUES ('Ritesh', 'r@x.com')       │
│           ▼                                                │
│  ┌──────────────────┐                                     │
│  │    DATABASE       │                                     │
│  │   (SQL Server)    │                                     │
│  └──────────────────┘                                     │
└──────────────────────────────────────────────────────────┘
```

### Quick Decision Guide

| I want to... | Use... |
|--------------|--------|
| Define a table | `class MyModel(Base)` |
| Connect to DB | `create_engine(url)` |
| Run queries | `session.query(Model)` or `session.execute(select(Model))` |
| Insert data | `session.add(obj)` + `session.commit()` |
| Update data | Modify attributes + `session.commit()` |
| Delete data | `session.delete(obj)` + `session.commit()` |
| Join tables | `relationship()` + eager loading |
| Raw SQL | `session.execute(text("..."))` |
| Migrations | Use **Alembic** (see Alembic notes) |
| Async | `create_async_engine` + `AsyncSession` |

---

*These notes cover SQLAlchemy 2.0 concepts. The library is backward-compatible with 1.x style, so both approaches work. Start with the ORM, use `echo=True`, and read the generated SQL — that's the fastest way to learn.*
