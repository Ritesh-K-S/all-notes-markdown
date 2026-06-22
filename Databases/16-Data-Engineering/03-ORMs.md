# 🔌 Chapter 5.3 — ORMs & Database Access Layers

> **Level:** 🟡 Intermediate
> **Time to Master:** ~4-5 hours
> **Prerequisites:** SQL Basics (Part 2A), any one programming language

---

## 🎯 What You'll Master

By the end of this chapter, you will:
- Understand **what ORMs do** and why they exist
- Know the **top ORMs** in every major language: Hibernate, Entity Framework, SQLAlchemy, Sequelize, Prisma, GORM, ActiveRecord
- Identify and **fix the N+1 query problem** — the #1 ORM performance killer
- Make informed decisions about **when to use an ORM vs raw SQL**
- Understand the **ORM spectrum** — from raw SQL to full ORM to query builders
- Write ORM code that **doesn't secretly destroy your database performance**

---

## 🧠 The Problem — Why Do ORMs Exist?

### The Impedance Mismatch

```
  Your APPLICATION thinks in OBJECTS:

  class User {                         class Order {
      int id;                              int id;
      String name;                         User customer;      ← Object reference
      String email;                        List<Item> items;   ← Collection
      List<Order> orders; ← Collection     DateTime createdAt;
  }                                    }

  Your DATABASE thinks in TABLES & ROWS:

  ┌───────────────────────────────┐    ┌──────────────────────────────────┐
  │ users                        │    │ orders                           │
  ├────┬────────┬────────────────┤    ├────┬─────────┬──────────────────┤
  │ id │ name   │ email          │    │ id │ user_id │ created_at       │
  ├────┼────────┼────────────────┤    ├────┼─────────┼──────────────────┤
  │ 1  │ Alice  │ alice@test.com │    │ 10 │ 1       │ 2026-06-01       │
  │ 2  │ Bob    │ bob@test.com   │    │ 11 │ 1       │ 2026-06-02       │
  └────┴────────┴────────────────┘    └────┴─────────┴──────────────────┘

  The GAP between objects and tables = "Object-Relational Impedance Mismatch"
  ORM = Object-Relational Mapping = Bridge between the two worlds
```

### Without an ORM (Raw SQL + Manual Mapping)

```java
// Java — Manual database access (the painful way)
public User getUserWithOrders(int userId) {
    Connection conn = DriverManager.getConnection(url, user, pass);

    // Query 1: Get user
    PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
    stmt.setInt(1, userId);
    ResultSet rs = stmt.executeQuery();

    User user = new User();
    if (rs.next()) {
        user.setId(rs.getInt("id"));
        user.setName(rs.getString("name"));
        user.setEmail(rs.getString("email"));
    }

    // Query 2: Get orders
    PreparedStatement stmt2 = conn.prepareStatement("SELECT * FROM orders WHERE user_id = ?");
    stmt2.setInt(1, userId);
    ResultSet rs2 = stmt2.executeQuery();

    List<Order> orders = new ArrayList<>();
    while (rs2.next()) {
        Order order = new Order();
        order.setId(rs2.getInt("id"));
        order.setTotal(rs2.getBigDecimal("total"));
        orders.add(order);
    }
    user.setOrders(orders);

    // Don't forget to close everything!
    rs2.close(); stmt2.close();
    rs.close(); stmt.close();
    conn.close();

    return user;  // 30+ lines for a simple query 😩
}
```

### With an ORM (Magic!)

```java
// Java + Hibernate — Same thing in 1 line
User user = session.get(User.class, userId);
// Hibernate generates the SQL, maps the result, handles connections
// user.getOrders() lazily loads orders when accessed
```

```python
# Python + SQLAlchemy — Same thing in 1 line
user = session.get(User, user_id)
# user.orders is automatically available
```

```csharp
// C# + Entity Framework — Same thing in 1 line
var user = context.Users.Include(u => u.Orders).FirstOrDefault(u => u.Id == userId);
```

---

## 🗺️ The Database Access Spectrum

```
  ◄──── More Control                                    More Convenience ────►
  ◄──── More SQL Knowledge                              Less SQL Knowledge ────►

  ┌────────────┐  ┌────────────────┐  ┌─────────────┐  ┌───────────────┐
  │  Raw SQL   │  │ Query Builder  │  │ Lightweight  │  │   Full ORM    │
  │            │  │                │  │    ORM       │  │               │
  │ • JDBC     │  │ • Knex.js      │  │ • Dapper     │  │ • Hibernate   │
  │ • psycopg2 │  │ • Kysely       │  │ • MyBatis    │  │ • EF Core     │
  │ • pg (npm) │  │ • JOOQ         │  │ • MicroORM   │  │ • SQLAlchemy  │
  │ • database │  │ • SQLBuilder   │  │              │  │ • Prisma      │
  │   /sql (Go)│  │                │  │              │  │ • Django ORM  │
  │            │  │                │  │              │  │ • ActiveRecord│
  │            │  │                │  │              │  │ • Sequelize   │
  │            │  │                │  │              │  │ • TypeORM     │
  └────────────┘  └────────────────┘  └─────────────┘  └───────────────┘
       │                 │                   │                  │
  You write SQL     You build SQL         You write SQL     ORM writes SQL
  You map results   programmatically      ORM maps results  ORM maps results
                    Type-safe!            Minimal overhead   Full entity management
```

---

## 🛠️ ORM Deep Dives — Every Major Language

### 1. Hibernate / JPA — Java's Enterprise ORM ⭐🔥

> **Used by:** Every Fortune 500 Java shop
> **Concept:** JPA (Java Persistence API) is the standard, Hibernate is the #1 implementation

```java
// Entity Definition
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(length = 255)
    private String email;

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();

    @CreationTimestamp
    private LocalDateTime createdAt;

    // Getters, setters...
}

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(precision = 10, scale = 2)
    private BigDecimal total;

    @Enumerated(EnumType.STRING)
    private OrderStatus status = OrderStatus.PENDING;

    @CreationTimestamp
    private LocalDateTime createdAt;
}
```

```java
// CRUD Operations with Hibernate

// CREATE
User user = new User();
user.setUsername("alice");
user.setEmail("alice@example.com");
session.persist(user);                    // INSERT INTO users ...

// READ
User user = session.get(User.class, 1L);  // SELECT * FROM users WHERE id = 1

// UPDATE
user.setEmail("new@example.com");
session.merge(user);                      // UPDATE users SET email = ... WHERE id = 1
// Or: Hibernate auto-detects changes (dirty checking) and flushes at commit!

// DELETE
session.remove(user);                     // DELETE FROM users WHERE id = 1

// QUERY (JPQL — Java Persistence Query Language)
List<User> users = session.createQuery(
    "SELECT u FROM User u WHERE u.email LIKE :domain", User.class)
    .setParameter("domain", "%@gmail.com")
    .getResultList();
// Generates: SELECT u.* FROM users u WHERE u.email LIKE '%@gmail.com'

// QUERY (Criteria API — Type-safe, no strings)
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<User> query = cb.createQuery(User.class);
Root<User> root = query.from(User.class);
query.select(root).where(cb.equal(root.get("username"), "alice"));
List<User> result = session.createQuery(query).getResultList();
```

#### Hibernate Fetch Types — LAZY vs EAGER

```
  LAZY (Default for Collections — @OneToMany, @ManyToMany):
  ┌──────────────────────────────────────────────────────────┐
  │ User user = session.get(User.class, 1);                  │
  │ // SQL: SELECT * FROM users WHERE id = 1                 │
  │ // Orders NOT loaded yet!                                │
  │                                                          │
  │ user.getOrders();  // NOW it loads!                      │
  │ // SQL: SELECT * FROM orders WHERE user_id = 1           │
  └──────────────────────────────────────────────────────────┘
  ✅ Pro: Don't load data you don't need
  ❌ Con: LazyInitializationException if session is closed!

  EAGER (Default for Single — @ManyToOne, @OneToOne):
  ┌──────────────────────────────────────────────────────────┐
  │ User user = session.get(User.class, 1);                  │
  │ // SQL: SELECT u.*, o.* FROM users u                     │
  │ //      LEFT JOIN orders o ON u.id = o.user_id           │
  │ //      WHERE u.id = 1                                   │
  │ // EVERYTHING loaded immediately!                        │
  └──────────────────────────────────────────────────────────┘
  ✅ Pro: All data available immediately
  ❌ Con: Loads unnecessary data, can cause N+1!
```

---

### 2. Entity Framework Core — C#/.NET's ORM ⭐

> **Used by:** Microsoft ecosystem, ASP.NET Core applications

```csharp
// Entity Definition
public class User
{
    public int Id { get; set; }

    [Required, MaxLength(50)]
    public string Username { get; set; }

    [MaxLength(255)]
    public string Email { get; set; }

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    // Navigation property
    public ICollection<Order> Orders { get; set; } = new List<Order>();
}

public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public decimal Total { get; set; }
    public string Status { get; set; } = "pending";
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    // Navigation property
    public User User { get; set; }
}

// DbContext
public class AppDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<User>()
            .HasIndex(u => u.Username)
            .IsUnique();

        modelBuilder.Entity<Order>()
            .HasOne(o => o.User)
            .WithMany(u => u.Orders)
            .HasForeignKey(o => o.UserId);
    }
}
```

```csharp
// CRUD Operations with EF Core

// CREATE
var user = new User { Username = "alice", Email = "alice@example.com" };
context.Users.Add(user);
await context.SaveChangesAsync();

// READ (with eager loading)
var user = await context.Users
    .Include(u => u.Orders)
    .FirstOrDefaultAsync(u => u.Id == 1);

// READ (with filtering)
var activeUsers = await context.Users
    .Where(u => u.Orders.Any(o => o.Status == "delivered"))
    .OrderBy(u => u.Username)
    .ToListAsync();

// UPDATE
user.Email = "new@example.com";
await context.SaveChangesAsync();  // EF detects the change automatically!

// DELETE
context.Users.Remove(user);
await context.SaveChangesAsync();

// LINQ Queries (type-safe, IntelliSense-powered!)
var topCustomers = await context.Users
    .Where(u => u.Orders.Count > 10)
    .Select(u => new {
        u.Username,
        TotalSpent = u.Orders.Sum(o => o.Total),
        OrderCount = u.Orders.Count
    })
    .OrderByDescending(x => x.TotalSpent)
    .Take(10)
    .ToListAsync();
// EF generates optimized SQL from this LINQ expression!
```

---

### 3. SQLAlchemy — Python's Swiss Army Knife ⭐

> **Used by:** Flask, FastAPI, data engineering, Airflow
> **Unique:** Has BOTH ORM mode AND Core mode (raw SQL builder)

```python
# Entity Definition (Declarative style — SQLAlchemy 2.0+)
from sqlalchemy import String, ForeignKey, Numeric
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from datetime import datetime

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, nullable=False)
    email: Mapped[str | None] = mapped_column(String(255))
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    orders: Mapped[list["Order"]] = relationship(back_populates="user", lazy="selectin")

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    total: Mapped[float] = mapped_column(Numeric(10, 2))
    status: Mapped[str] = mapped_column(String(20), default="pending")

    user: Mapped["User"] = relationship(back_populates="orders")
```

```python
# CRUD Operations with SQLAlchemy

from sqlalchemy import create_engine, select
from sqlalchemy.orm import Session

engine = create_engine("postgresql://user:pass@localhost/mydb")

# CREATE
with Session(engine) as session:
    user = User(username="alice", email="alice@example.com")
    session.add(user)
    session.commit()

# READ
with Session(engine) as session:
    # Simple get by primary key
    user = session.get(User, 1)

    # Query with filters
    stmt = select(User).where(User.email.like("%@gmail.com")).order_by(User.username)
    users = session.scalars(stmt).all()

    # Complex query with join
    stmt = (
        select(User.username, func.sum(Order.total).label("total_spent"))
        .join(Order)
        .group_by(User.username)
        .having(func.sum(Order.total) > 1000)
        .order_by(func.sum(Order.total).desc())
    )
    results = session.execute(stmt).all()

# UPDATE
with Session(engine) as session:
    user = session.get(User, 1)
    user.email = "new@example.com"      # SQLAlchemy tracks changes!
    session.commit()

# DELETE
with Session(engine) as session:
    user = session.get(User, 1)
    session.delete(user)
    session.commit()
```

#### SQLAlchemy Core vs ORM

```python
# CORE MODE — SQL builder without ORM objects (great for complex queries)
from sqlalchemy import text, select, insert

# Raw SQL (with parameterized queries — safe from SQL injection!)
with engine.connect() as conn:
    result = conn.execute(
        text("SELECT * FROM users WHERE email = :email"),
        {"email": "alice@example.com"}
    )

# Core expression builder (no ORM objects, just tables)
from sqlalchemy import Table, MetaData
metadata = MetaData()
users = Table("users", metadata, autoload_with=engine)

stmt = select(users.c.username, users.c.email).where(users.c.id == 1)
result = conn.execute(stmt)
```

---

### 4. Prisma — The Modern TypeScript ORM 🔥

> **Used by:** Next.js, NestJS, modern Node.js apps
> **Unique:** Schema-first, auto-generated type-safe client

```typescript
// prisma/schema.prisma (single source of truth)
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        Int      @id @default(autoincrement())
  username  String   @unique @db.VarChar(50)
  email     String?  @db.VarChar(255)
  createdAt DateTime @default(now())
  orders    Order[]
}

model Order {
  id        Int      @id @default(autoincrement())
  userId    Int
  total     Decimal  @db.Decimal(10, 2)
  status    String   @default("pending")
  createdAt DateTime @default(now())
  user      User     @relation(fields: [userId], references: [id])
}
```

```typescript
// CRUD Operations with Prisma (100% type-safe!)
import { PrismaClient } from '@prisma/client'
const prisma = new PrismaClient()

// CREATE
const user = await prisma.user.create({
  data: {
    username: 'alice',
    email: 'alice@example.com',
    orders: {
      create: [                          // Create user AND orders in one call!
        { total: 99.99, status: 'delivered' },
        { total: 49.99, status: 'pending' },
      ]
    }
  },
  include: { orders: true }             // Return with orders
})

// READ
const user = await prisma.user.findUnique({
  where: { id: 1 },
  include: { orders: true }
})

// Complex query
const topCustomers = await prisma.user.findMany({
  where: {
    orders: { some: { status: 'delivered' } }   // Users with delivered orders
  },
  include: {
    orders: {
      where: { status: 'delivered' },
      orderBy: { createdAt: 'desc' },
      take: 5                                   // Last 5 delivered orders
    },
    _count: { select: { orders: true } }        // Total order count
  },
  orderBy: { createdAt: 'desc' },
  take: 10
})

// UPDATE
const updated = await prisma.user.update({
  where: { id: 1 },
  data: { email: 'new@example.com' }
})

// UPSERT (create or update)
const user = await prisma.user.upsert({
  where: { username: 'alice' },
  create: { username: 'alice', email: 'alice@example.com' },
  update: { email: 'alice@example.com' }
})

// DELETE
await prisma.user.delete({ where: { id: 1 } })

// TRANSACTION
const [order, updatedUser] = await prisma.$transaction([
  prisma.order.create({ data: { userId: 1, total: 199.99 } }),
  prisma.user.update({ where: { id: 1 }, data: { email: 'vip@example.com' } })
])

// RAW SQL (when you need it)
const result = await prisma.$queryRaw`
  SELECT u.username, SUM(o.total) as total_spent
  FROM users u JOIN orders o ON u.id = o.user_id
  GROUP BY u.username
  HAVING SUM(o.total) > ${1000}
  ORDER BY total_spent DESC
`
```

---

### 5. Django ORM — Python Web's Favorite

```python
# models.py
from django.db import models

class User(models.Model):
    username = models.CharField(max_length=50, unique=True)
    email = models.EmailField(blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'users'

class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='orders')
    total = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20, default='pending')
    created_at = models.DateTimeField(auto_now_add=True)

# CRUD
user = User.objects.create(username='alice', email='alice@example.com')
user = User.objects.get(id=1)
users = User.objects.filter(email__endswith='@gmail.com').order_by('username')
user.email = 'new@example.com'; user.save()
user.delete()

# Complex query
from django.db.models import Sum, Count
top_customers = (
    User.objects
    .annotate(total_spent=Sum('orders__total'), order_count=Count('orders'))
    .filter(order_count__gt=10)
    .order_by('-total_spent')[:10]
)

# Prefetch to avoid N+1
users = User.objects.prefetch_related('orders').all()
```

---

### 6. Quick Reference — Other ORMs

| ORM | Language | Style | Key Feature |
|-----|----------|-------|-------------|
| **ActiveRecord** | Ruby | Convention over config | `User.where(active: true).order(:name)` |
| **Sequelize** | Node.js | Traditional ORM | Mature, full-featured, callbacks |
| **TypeORM** | TypeScript | Decorator-based | Similar to Hibernate for TS |
| **GORM** | Go | Struct tags | `db.Where("age > ?", 18).Find(&users)` |
| **Diesel** | Rust | Compile-time checked | Zero-cost abstractions |
| **Eloquent** | PHP/Laravel | ActiveRecord pattern | Beautiful syntax, Laravel-native |
| **Ecto** | Elixir | Changesets | Functional, composable queries |
| **Exposed** | Kotlin | DSL + DAO | Type-safe SQL DSL |
| **JOOQ** | Java | SQL-first | Generates type-safe Java from schema |
| **Drizzle** | TypeScript | SQL-like TS | Lightweight, SQL-centric |
| **Knex.js** | Node.js | Query builder | Not ORM — just SQL builder |

---

## 🐛 The N+1 Query Problem — ORM's #1 Performance Killer

### What Is N+1?

```
  Scenario: Display 100 users with their latest order

  ❌ N+1 Problem (101 queries!):

  Query 1:  SELECT * FROM users LIMIT 100;              ← 1 query for users

  Then FOR EACH user (100 times):
  Query 2:  SELECT * FROM orders WHERE user_id = 1;     ← Query for user 1's orders
  Query 3:  SELECT * FROM orders WHERE user_id = 2;     ← Query for user 2's orders
  Query 4:  SELECT * FROM orders WHERE user_id = 3;     ← Query for user 3's orders
  ...
  Query 101: SELECT * FROM orders WHERE user_id = 100;  ← Query for user 100's orders

  Total: 1 + 100 = 101 queries! 🐌
  Each query = network round-trip to DB = ~1-5ms
  101 queries × 3ms = 303ms minimum latency!

  ✅ Optimized (2 queries):

  Query 1:  SELECT * FROM users LIMIT 100;
  Query 2:  SELECT * FROM orders WHERE user_id IN (1,2,3,...,100);

  Total: 2 queries! ⚡
  2 queries × 3ms = 6ms. That's 50x faster!
```

### N+1 Visualized

```
  ❌ N+1 (Lazy loading without optimization):

  App ──Query 1──► DB    (fetch users)
  App ◄──Result──── DB

  App ──Query 2──► DB    (fetch orders for user 1)
  App ◄──Result──── DB

  App ──Query 3──► DB    (fetch orders for user 2)
  App ◄──Result──── DB

  ... 98 more round trips ...

  Total round trips: 101 🐌


  ✅ Eager Loading / Batch Loading:

  App ──Query 1──► DB    (fetch users)
  App ◄──Result──── DB

  App ──Query 2──► DB    (fetch ALL orders for these users)
  App ◄──Result──── DB

  Total round trips: 2 ⚡
```

### How to Fix N+1 in Every ORM

```java
// HIBERNATE — Eager fetch with JOIN FETCH
// ❌ N+1
List<User> users = session.createQuery("FROM User", User.class).getResultList();
for (User u : users) {
    u.getOrders().size();  // Each call = separate query!
}

// ✅ Fixed — JOIN FETCH
List<User> users = session.createQuery(
    "SELECT u FROM User u JOIN FETCH u.orders", User.class
).getResultList();

// ✅ Fixed — @BatchSize annotation
@OneToMany(mappedBy = "user")
@BatchSize(size = 100)  // Load orders in batches of 100 user IDs
private List<Order> orders;

// ✅ Fixed — Entity Graph
@EntityGraph(attributePaths = {"orders"})
List<User> findAll();
```

```csharp
// ENTITY FRAMEWORK — Include()
// ❌ N+1
var users = context.Users.ToList();
foreach (var u in users) {
    var orders = u.Orders;  // Lazy load = separate query per user!
}

// ✅ Fixed — Eager loading with Include
var users = context.Users.Include(u => u.Orders).ToList();

// ✅ Fixed — Explicit loading
foreach (var u in users) {
    context.Entry(u).Collection(u => u.Orders).Load();
}

// ✅ Fixed — Split query (avoids cartesian explosion)
var users = context.Users
    .Include(u => u.Orders)
    .AsSplitQuery()          // Separate queries but batched
    .ToList();
```

```python
# SQLALCHEMY — joinedload / selectinload
# ❌ N+1
users = session.scalars(select(User)).all()
for u in users:
    print(u.orders)  # Each access = separate query!

# ✅ Fixed — Joined load (single JOIN query)
from sqlalchemy.orm import joinedload
users = session.scalars(
    select(User).options(joinedload(User.orders))
).unique().all()

# ✅ Fixed — Select-in load (2 queries: users + orders)
from sqlalchemy.orm import selectinload
users = session.scalars(
    select(User).options(selectinload(User.orders))
).all()

# ✅ Fixed — Subquery load
from sqlalchemy.orm import subqueryload
users = session.scalars(
    select(User).options(subqueryload(User.orders))
).all()
```

```typescript
// PRISMA — Include
// ❌ N+1
const users = await prisma.user.findMany()
for (const u of users) {
    const orders = await prisma.order.findMany({ where: { userId: u.id } })
    // Separate query per user!
}

// ✅ Fixed — Include
const users = await prisma.user.findMany({
    include: { orders: true }
})
// Prisma generates 2 optimized queries automatically
```

```python
# DJANGO — select_related / prefetch_related
# ❌ N+1
users = User.objects.all()
for u in users:
    print(u.orders.all())  # Separate query per user!

# ✅ Fixed — prefetch_related (for *-to-many)
users = User.objects.prefetch_related('orders').all()

# ✅ Fixed — select_related (for *-to-one, uses JOIN)
orders = Order.objects.select_related('user').all()
```

---

## ⚖️ ORM vs Raw SQL — When to Use What

```
  Use an ORM when:
  ┌──────────────────────────────────────────────────────────────┐
  │  ✅ Standard CRUD operations (80% of most apps)              │
  │  ✅ Rapid prototyping / MVP development                      │
  │  ✅ Type safety and IDE autocomplete matter                  │
  │  ✅ Multiple database support needed                         │
  │  ✅ Team has varying SQL skill levels                        │
  │  ✅ Schema migrations are integrated                         │
  │  ✅ Security (ORMs prevent SQL injection by default)         │
  └──────────────────────────────────────────────────────────────┘

  Use Raw SQL when:
  ┌──────────────────────────────────────────────────────────────┐
  │  ✅ Complex analytical queries (multi-join aggregations)      │
  │  ✅ Performance-critical paths (ORM overhead matters)         │
  │  ✅ Database-specific features (window functions, CTEs, etc.) │
  │  ✅ Bulk operations (INSERT millions of rows)                │
  │  ✅ Stored procedures / DB functions                         │
  │  ✅ Query plan optimization (you need exact SQL control)     │
  │  ✅ Data engineering / ETL workloads                          │
  └──────────────────────────────────────────────────────────────┘

  💡 Best Practice: Use ORM for 80% of queries, drop to raw SQL for the 20%
     that need performance or complexity. ALL major ORMs support raw SQL.
```

### Performance Comparison

```
  Operation: Insert 100,000 rows

  ┌──────────────────────┬──────────┬───────────────────────────────┐
  │ Method               │ Time     │ Notes                         │
  ├──────────────────────┼──────────┼───────────────────────────────┤
  │ Raw SQL (bulk)       │ 0.8s     │ Single INSERT with VALUES     │
  │ ORM (batch commit)   │ 3.2s     │ session.add_all() + commit    │
  │ ORM (one-by-one)     │ 45s      │ commit after each insert 🐌   │
  │ Raw COPY (PG)        │ 0.3s     │ COPY FROM stdin               │
  └──────────────────────┴──────────┴───────────────────────────────┘

  Operation: Complex JOIN with aggregation

  ┌──────────────────────┬──────────┬───────────────────────────────┐
  │ Method               │ Time     │ Notes                         │
  ├──────────────────────┼──────────┼───────────────────────────────┤
  │ Raw SQL              │ 12ms     │ Optimized, index-aware        │
  │ ORM (well-written)   │ 15ms     │ ORM generates good SQL        │
  │ ORM (naive, N+1)     │ 850ms    │ 100+ queries! 🐌              │
  └──────────────────────┴──────────┴───────────────────────────────┘
```

---

## 🛡️ ORM Security — SQL Injection Prevention

```python
# ❌ VULNERABLE — String concatenation (NEVER DO THIS!)
username = request.args.get('username')
query = f"SELECT * FROM users WHERE username = '{username}'"
# Attacker sends: ' OR '1'='1' --
# Becomes: SELECT * FROM users WHERE username = '' OR '1'='1' --'
# Returns ALL users! 💀

# ✅ SAFE — Parameterized query (ORM does this automatically)
user = session.query(User).filter(User.username == username).first()
# ORM generates: SELECT * FROM users WHERE username = $1
# Parameter: ['attacker_input']
# SQL injection IMPOSSIBLE! ✅

# ✅ SAFE — Even raw SQL can be parameterized
result = session.execute(
    text("SELECT * FROM users WHERE username = :name"),
    {"name": username}
)
```

---

## 🧪 ORM Best Practices Checklist

```
  ✅ Always check for N+1 queries (use SQL logging in dev!)
  ✅ Use eager loading for known access patterns
  ✅ Use lazy loading for rarely accessed relationships
  ✅ Use batch operations for bulk inserts/updates
  ✅ Enable SQL logging in development to see generated queries
  ✅ Use transactions explicitly for multi-step operations
  ✅ Index foreign key columns (ORMs don't always do this!)
  ✅ Use connection pooling (Chapter 5.4)
  ✅ Profile ORM queries in production (Chapter 5.5)
  ✅ Drop to raw SQL for complex analytics queries
  ✅ NEVER concatenate user input into queries
  ✅ Keep entity/model classes simple — no business logic
```

### Enable SQL Logging (See What Your ORM Actually Does!)

```python
# SQLAlchemy
engine = create_engine("postgresql://...", echo=True)
# Every SQL query will be printed to console!

# Django
LOGGING = {
    'loggers': {
        'django.db.backends': {
            'level': 'DEBUG',
        }
    }
}
```

```java
// Hibernate — application.properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

```typescript
// Prisma
const prisma = new PrismaClient({ log: ['query', 'info', 'warn', 'error'] })
```

---

## 🏆 Key Takeaways

| Concept | Remember This |
|---------|---------------|
| **ORM** | Maps objects to tables — bridge between app and DB |
| **N+1 Problem** | Use eager loading / prefetch — NEVER loop + query |
| **80/20 Rule** | ORM for CRUD, raw SQL for complex queries |
| **Hibernate** | Java enterprise standard, JPA spec |
| **EF Core** | .NET standard, LINQ integration |
| **SQLAlchemy** | Python's powerhouse (Core + ORM modes) |
| **Prisma** | TypeScript's modern, type-safe ORM |
| **Django ORM** | Python web's go-to, select/prefetch_related |
| **SQL Logging** | Always enable in dev — see what queries run |
| **Security** | ORMs prevent SQL injection by default — USE parameterized queries |
| **Batch Operations** | Never insert one-by-one — batch for bulk operations |

---

> 🚀 **Next Up:** [Chapter 5.4 — Connection Pooling & Database Proxies](./04-Connection-Pooling.md) — Learn how to manage hundreds of database connections efficiently with PgBouncer, ProxySQL, and HikariCP.

---

*Last Updated: June 2026*
