# Multi-Tenancy Architecture

> **What you'll learn**: How to design systems that serve multiple customers (tenants) from shared infrastructure — covering data isolation models, tenant-aware routing, performance isolation, and the architecture patterns behind SaaS platforms like Salesforce, Slack, Shopify, and AWS itself.

---

## Real-Life Analogy

Think of three types of housing:

1. **Detached house** (single-family home) — Each family has their own building, yard, utilities. Complete privacy, but expensive. → **Single-tenant (dedicated infrastructure per customer)**

2. **Apartment building** — Multiple families share the building, plumbing, and electricity. Each has a locked private unit. Efficient, but one neighbor's water leak might affect you. → **Multi-tenant with shared infrastructure**

3. **Shared hostel room** — Multiple people share the same room, bathroom, and kitchen. Cheapest, but zero privacy. → **Shared everything (don't do this!)**

Multi-tenancy in software is like the apartment building: **shared infrastructure, private data**. The goal is to give every tenant the experience of having their own system while actually sharing resources efficiently.

---

## What is Multi-Tenancy?

**Multi-tenancy** means a single instance of software serves multiple customers (tenants). Each tenant's data is isolated — they can't see each other's data.

```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTI-TENANT SYSTEM                            │
│                                                                   │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐                   │
│  │ Tenant A  │  │ Tenant B  │  │ Tenant C  │  ← Different      │
│  │ (Acme Co) │  │ (Beta Inc)│  │ (Gamma Ltd)│    customers      │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘                   │
│        │               │               │                         │
│        └───────────────┼───────────────┘                         │
│                        │                                         │
│                        ▼                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              SHARED APPLICATION LAYER                     │    │
│  │         (Same code serves all tenants)                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                        │                                         │
│                        ▼                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              SHARED INFRASTRUCTURE                        │    │
│  │    (Servers, databases, caches — with isolation)          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  vs. SINGLE-TENANT:                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│  │ App + DB │  │ App + DB │  │ App + DB │  ← Separate           │
│  │ for A    │  │ for B    │  │ for C    │    everything         │
│  └──────────┘  └──────────┘  └──────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Multi-Tenancy Data Isolation Models

### Model 1: Shared Database, Shared Schema (tenant_id column)

All tenants share the same tables. Each row has a `tenant_id` column.

```
┌─────────────────────────────────────────────────────────────┐
│  SHARED DATABASE, SHARED SCHEMA                              │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Single Database                          │    │
│  │                                                       │    │
│  │  orders table:                                        │    │
│  │  ┌───────────┬──────────┬────────┬─────────┐        │    │
│  │  │ tenant_id │ order_id │ amount │ status  │        │    │
│  │  ├───────────┼──────────┼────────┼─────────┤        │    │
│  │  │ acme      │ ord_001  │ 99.99  │ shipped │        │    │
│  │  │ acme      │ ord_002  │ 149.99 │ pending │        │    │
│  │  │ beta      │ ord_003  │ 75.00  │ shipped │        │    │
│  │  │ gamma     │ ord_004  │ 200.00 │ pending │        │    │
│  │  └───────────┴──────────┴────────┴─────────┘        │    │
│  │                                                       │    │
│  │  EVERY query must include: WHERE tenant_id = ?        │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
│  ✅ Cheapest: One DB for all tenants                         │
│  ✅ Simple: No infra per tenant                              │
│  ❌ Risk: Forgetting tenant_id filter = data leak!          │
│  ❌ Noisy neighbor: One tenant's heavy query slows everyone │
│  ❌ Scale limit: One DB for all tenants                     │
└─────────────────────────────────────────────────────────────┘
```

### Model 2: Shared Database, Separate Schemas

Each tenant gets their own schema (namespace) within the same database.

```
┌─────────────────────────────────────────────────────────────┐
│  SHARED DATABASE, SEPARATE SCHEMAS                           │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Single Database                          │    │
│  │                                                       │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │    │
│  │  │ schema:acme │  │ schema:beta │  │schema:gamma│  │    │
│  │  │             │  │             │  │            │  │    │
│  │  │ orders      │  │ orders      │  │ orders     │  │    │
│  │  │ users       │  │ users       │  │ users      │  │    │
│  │  │ products    │  │ products    │  │ products   │  │    │
│  │  └─────────────┘  └─────────────┘  └────────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
│  ✅ Better isolation than shared schema                      │
│  ✅ Easy migrations per tenant                               │
│  ❌ Still shared DB resources (CPU, memory, I/O)            │
│  ❌ Schema management complexity (1000 tenants = 1000 schemas)│
└─────────────────────────────────────────────────────────────┘
```

### Model 3: Separate Databases per Tenant

Each tenant gets their own database instance.

```
┌─────────────────────────────────────────────────────────────┐
│  SEPARATE DATABASES PER TENANT                               │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  DB: acme    │  │  DB: beta    │  │  DB: gamma   │      │
│  │              │  │              │  │              │      │
│  │  orders      │  │  orders      │  │  orders      │      │
│  │  users       │  │  users       │  │  users       │      │
│  │  products    │  │  products    │  │  products    │      │
│  │              │  │              │  │              │      │
│  │  Server A    │  │  Server B    │  │  Server C    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                               │
│  ✅ Strongest isolation (no data leaks possible)             │
│  ✅ Independent scaling per tenant                           │
│  ✅ Easy compliance (tenant data in specific region)         │
│  ❌ Most expensive (N databases = N × cost)                  │
│  ❌ Operational overhead (managing thousands of DBs)         │
│  ❌ Cross-tenant analytics harder                            │
└─────────────────────────────────────────────────────────────┘
```

### Comparison Table:

```
┌──────────────────┬───────────────┬──────────────────┬────────────────┐
│                  │ Shared Schema │ Separate Schemas │ Separate DBs   │
│                  │ (tenant_id)   │ (per-tenant)     │ (per-tenant)   │
├──────────────────┼───────────────┼──────────────────┼────────────────┤
│ Data Isolation   │ ⚠️ Low        │ 🟡 Medium        │ ✅ High         │
│ Cost             │ ✅ Lowest      │ 🟡 Medium        │ ❌ Highest      │
│ Scalability      │ ❌ Limited     │ 🟡 Moderate      │ ✅ Best         │
│ Ops Complexity   │ ✅ Simple      │ 🟡 Moderate      │ ❌ Complex      │
│ Noisy Neighbor   │ ❌ High risk   │ ⚠️ Medium risk   │ ✅ No risk      │
│ Schema Migration │ ✅ Once        │ ❌ Per tenant     │ ❌ Per tenant   │
│ Compliance       │ ❌ Hard        │ 🟡 Possible      │ ✅ Easy         │
│ Best for         │ Startup/MVP   │ Growing SaaS     │ Enterprise SaaS│
│ Tenant count     │ 10K+ tenants  │ 100-1000 tenants │ 10-100 tenants │
└──────────────────┴───────────────┴──────────────────┴────────────────┘
```

---

## Hybrid Model — The Best of Both Worlds

Most production systems use a **hybrid approach**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    HYBRID MULTI-TENANCY                           │
│                                                                   │
│  ┌───────────────────────────────────────┐                      │
│  │         SMALL TENANTS (Free/Basic)     │                      │
│  │                                         │                      │
│  │  Shared DB with tenant_id column        │                      │
│  │  100s of tenants per database           │                      │
│  │  Cost-effective, acceptable isolation   │                      │
│  └───────────────────────────────────────┘                      │
│                                                                   │
│  ┌───────────────────────────────────────┐                      │
│  │         MEDIUM TENANTS (Pro Plan)       │                      │
│  │                                         │                      │
│  │  Shared DB, separate schema             │                      │
│  │  Better isolation, some resource sharing │                      │
│  └───────────────────────────────────────┘                      │
│                                                                   │
│  ┌───────────────────────────────────────┐                      │
│  │         LARGE TENANTS (Enterprise)      │                      │
│  │                                         │                      │
│  │  Dedicated database (maybe dedicated    │                      │
│  │  compute too)                           │                      │
│  │  Full isolation, custom SLAs            │                      │
│  └───────────────────────────────────────┘                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tenant-Aware Request Routing

How does the system know which tenant a request belongs to?

```
┌─────────────────────────────────────────────────────────────────┐
│              TENANT IDENTIFICATION METHODS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. SUBDOMAIN-BASED                                              │
│     acme.myapp.com → tenant = "acme"                            │
│     beta.myapp.com → tenant = "beta"                            │
│                                                                   │
│  2. PATH-BASED                                                   │
│     myapp.com/acme/dashboard → tenant = "acme"                  │
│     myapp.com/beta/dashboard → tenant = "beta"                  │
│                                                                   │
│  3. HEADER-BASED                                                 │
│     X-Tenant-ID: acme                                           │
│     (Common for APIs)                                            │
│                                                                   │
│  4. JWT CLAIM                                                    │
│     Token payload: { "tenant_id": "acme", "user_id": "123" }   │
│     (Most secure — can't be spoofed by client)                  │
│                                                                   │
│  5. CUSTOM DOMAIN                                                │
│     acme-crm.com → maps to tenant "acme"                       │
│     (Requires domain → tenant lookup table)                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Request Flow:

```
Request: GET https://acme.myapp.com/api/orders
    │
    ▼
┌───────────────────────┐
│  Tenant Resolution     │  Extract tenant from subdomain
│  tenant_id = "acme"    │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Authentication        │  Verify user belongs to this tenant
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Tenant Context Set    │  ThreadLocal/Context stores tenant_id
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Route to Tenant's DB  │  Lookup: acme → shard-3.db.internal
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Execute Query         │  SELECT * FROM orders 
│                         │  WHERE tenant_id = 'acme'
└───────────────────────┘  (auto-added by framework)
```

---

## How It Works Internally — Automatic Tenant Filtering

The most dangerous bug in multi-tenancy: **forgetting the tenant filter**. One missing `WHERE tenant_id = ?` and you leak data across tenants.

Solution: **Automatic tenant filtering at the framework level**.

### Row-Level Security (PostgreSQL):

```sql
-- Create policy that automatically filters by tenant
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant'));

-- Before any query, set the tenant context:
SET app.current_tenant = 'acme';

-- Now ALL queries are automatically filtered:
SELECT * FROM orders;
-- Postgres internally adds: WHERE tenant_id = 'acme'
-- Even if developer forgets! 🛡️
```

---

## Code Examples

### Python: Multi-Tenant Flask Application

```python
from flask import Flask, request, g
from functools import wraps
import psycopg2
from contextlib import contextmanager

app = Flask(__name__)

# ─── Tenant resolution middleware ─────────────────────────
@app.before_request
def resolve_tenant():
    """Extract tenant from subdomain or header"""
    # Method 1: From subdomain (acme.myapp.com)
    host = request.host.split(".")[0]
    if host in ("www", "myapp", "localhost"):
        # Method 2: From header (for API clients)
        tenant_id = request.headers.get("X-Tenant-ID")
    else:
        tenant_id = host
    
    if not tenant_id:
        return {"error": "Tenant not identified"}, 400
    
    # Validate tenant exists
    tenant = get_tenant_config(tenant_id)
    if not tenant:
        return {"error": "Unknown tenant"}, 404
    
    # Store in request context
    g.tenant_id = tenant_id
    g.tenant_config = tenant


# ─── Database connection with tenant context ──────────────
TENANT_DB_MAP = {
    "acme": "postgresql://user:pass@shard-1.db/app",
    "beta": "postgresql://user:pass@shard-1.db/app",
    "gamma": "postgresql://user:pass@shard-2.db/app",
    # Enterprise tenants get dedicated DBs
    "bigcorp": "postgresql://user:pass@bigcorp-dedicated.db/app",
}

@contextmanager
def tenant_db_connection():
    """Get DB connection for current tenant with RLS"""
    db_url = TENANT_DB_MAP.get(g.tenant_id)
    conn = psycopg2.connect(db_url)
    
    # Set Row-Level Security context
    cur = conn.cursor()
    cur.execute("SET app.current_tenant = %s", (g.tenant_id,))
    
    try:
        yield conn
    finally:
        conn.close()


# ─── Tenant-aware endpoint ────────────────────────────────
@app.route("/api/orders")
def list_orders():
    with tenant_db_connection() as conn:
        cur = conn.cursor()
        # No need for WHERE tenant_id = ? — RLS handles it!
        cur.execute("""
            SELECT id, total, status, created_at 
            FROM orders 
            ORDER BY created_at DESC 
            LIMIT 50
        """)
        orders = cur.fetchall()
    
    return {"data": [format_order(o) for o in orders]}


# ─── Tenant-scoped cache keys ─────────────────────────────
import redis
cache = redis.Redis()

def get_cached_orders(tenant_id):
    """Cache keys are namespaced by tenant"""
    cache_key = f"tenant:{tenant_id}:orders:recent"
    return cache.get(cache_key)

def set_cached_orders(tenant_id, data, ttl=300):
    cache_key = f"tenant:{tenant_id}:orders:recent"
    cache.setex(cache_key, ttl, data)
```

### Java: Multi-Tenant Spring Boot Application

```java
// ─── Tenant Context (ThreadLocal) ────────────────────────
public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();
    
    public static void setTenantId(String tenantId) {
        CURRENT_TENANT.set(tenantId);
    }
    
    public static String getTenantId() {
        return CURRENT_TENANT.get();
    }
    
    public static void clear() {
        CURRENT_TENANT.remove();
    }
}

// ─── Tenant Resolution Filter ────────────────────────────
@Component
public class TenantFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
            HttpServletResponse response, FilterChain chain) 
            throws ServletException, IOException {
        
        String tenantId = resolveTenant(request);
        
        if (tenantId == null) {
            response.sendError(400, "Tenant not identified");
            return;
        }
        
        try {
            TenantContext.setTenantId(tenantId);
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();  // CRITICAL: prevent tenant leaks!
        }
    }
    
    private String resolveTenant(HttpServletRequest request) {
        // Try subdomain first
        String host = request.getServerName();
        String subdomain = host.split("\\.")[0];
        if (!subdomain.equals("www") && !subdomain.equals("api")) {
            return subdomain;
        }
        // Fall back to header
        return request.getHeader("X-Tenant-ID");
    }
}

// ─── Dynamic DataSource Routing ──────────────────────────
public class TenantAwareDataSource extends AbstractRoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.getTenantId();
    }
}

// Configuration
@Configuration
public class DataSourceConfig {
    
    @Bean
    public DataSource dataSource() {
        TenantAwareDataSource routingDs = new TenantAwareDataSource();
        
        Map<Object, Object> dataSources = new HashMap<>();
        dataSources.put("acme", createDataSource("jdbc:postgresql://shard-1/app"));
        dataSources.put("beta", createDataSource("jdbc:postgresql://shard-1/app"));
        dataSources.put("bigcorp", createDataSource("jdbc:postgresql://bigcorp-dedicated/app"));
        
        routingDs.setTargetDataSources(dataSources);
        routingDs.setDefaultTargetDataSource(dataSources.get("acme"));
        return routingDs;
    }
}

// ─── JPA Entity with Tenant Filter ──────────────────────
@Entity
@Table(name = "orders")
@FilterDef(name = "tenantFilter", 
    parameters = @ParamDef(name = "tenantId", type = String.class))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
public class Order {
    @Id
    private String id;
    
    @Column(name = "tenant_id")
    private String tenantId;
    
    private BigDecimal total;
    private String status;
}

// ─── Auto-enable tenant filter on every session ─────────
@Component
public class TenantHibernateFilter implements HibernateFilterConfigurer {
    
    @Override
    public void configureFilters(Session session) {
        String tenantId = TenantContext.getTenantId();
        if (tenantId != null) {
            session.enableFilter("tenantFilter")
                .setParameter("tenantId", tenantId);
        }
    }
}
```

---

## Performance Isolation — Preventing Noisy Neighbors

```
┌─────────────────────────────────────────────────────────────────┐
│             NOISY NEIGHBOR PROBLEM                                │
│                                                                   │
│  Tenant A: Normal usage (100 req/s)                              │
│  Tenant B: Normal usage (50 req/s)                               │
│  Tenant C: Running massive export (10,000 req/s) 🔥             │
│                                                                   │
│  Without isolation:                                               │
│    Tenant C consumes all CPU → A and B experience timeouts!      │
│                                                                   │
│  SOLUTIONS:                                                       │
│                                                                   │
│  1. Rate Limiting per Tenant                                     │
│     ┌─────────┬──────────────┐                                   │
│     │ Tenant  │ Rate Limit    │                                   │
│     ├─────────┼──────────────┤                                   │
│     │ Free    │ 100 req/min  │                                   │
│     │ Pro     │ 1000 req/min │                                   │
│     │ Ent.    │ 10000 req/min│                                   │
│     └─────────┴──────────────┘                                   │
│                                                                   │
│  2. Resource Quotas (Kubernetes)                                 │
│     Each tenant's workload gets CPU/memory limits                │
│                                                                   │
│  3. Queue-Based Isolation                                        │
│     Each tenant has its own job queue with processing limits     │
│                                                                   │
│  4. Database Connection Pooling per Tenant                       │
│     Tenant A: max 10 connections                                 │
│     Tenant B: max 10 connections                                 │
│     Tenant C: max 10 connections (can't hog all connections)     │
│                                                                   │
│  5. Dedicated Resources for Large Tenants                        │
│     Move "noisy" tenants to dedicated infrastructure             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tenant Onboarding Flow

```
New Tenant Signs Up
        │
        ▼
┌─────────────────────┐
│  Create tenant record│
│  in control plane DB │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Provision resources │
│  (based on plan)     │
│                       │
│  Free: Just add      │
│    tenant_id to      │
│    shared DB         │
│                       │
│  Enterprise:         │
│    Create new DB     │
│    Create DNS entry  │
│    Set up custom     │
│    domain            │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Run migrations/     │
│  seed data           │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Configure routing   │
│  (subdomain → infra) │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Tenant ready!       │
│  acme.myapp.com ✅   │
└─────────────────────┘
```

---

## Real-World Example

### Salesforce — The Multi-Tenancy Pioneer

Salesforce serves 150,000+ organizations from shared infrastructure:
- **Model**: Shared database, shared schema with `org_id` column
- **Isolation**: Every query automatically filtered by `org_id`
- **Custom fields**: Stored in universal `data_1` through `data_500` columns (metadata-driven)
- **Scale**: Single DB instance can serve thousands of tenants
- **Big tenants**: Automatically moved to larger pod groups

### Slack — Hybrid Multi-Tenancy

- **Small workspaces** (free): Shared infrastructure, shared database
- **Enterprise Grid**: Dedicated infrastructure, separate databases
- **Data residency**: EU customers' data stays in EU region
- **Isolation**: Rate limiting per workspace to prevent noisy neighbors

### Shopify — Pod Architecture

Shopify groups tenants into "pods":
- Each pod is an independent, complete deployment
- New merchants assigned to pod with most capacity
- Large merchants (like Kylie Cosmetics) get dedicated pods
- If a pod has issues, only those merchants are affected

```
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│ Pod 1  │  │ Pod 2  │  │ Pod 3  │  │ Pod 4  │
│        │  │        │  │        │  │        │
│ 5000   │  │ 5000   │  │ 1      │  │ 4500   │
│ small  │  │ small  │  │ HUGE   │  │ small  │
│ shops  │  │ shops  │  │ shop   │  │ shops  │
└────────┘  └────────┘  └────────┘  └────────┘
```

### AWS — Multi-Tenancy at Infrastructure Level

AWS itself is the ultimate multi-tenant system:
- EC2 runs multiple customer VMs on shared physical hardware
- RDS runs multiple customer databases on shared clusters
- Isolation via hypervisors, network VPCs, and IAM
- "Noisy neighbor" prevention via dedicated instances (at higher cost)

---

## Tenant Data Migration and Offboarding

```
TENANT OFFBOARDING:
─────────────────────────────────────────
1. Export tenant's data (GDPR compliance)
2. Deactivate tenant access
3. Grace period (30 days)
4. Delete all tenant data:
   - Database records
   - File storage (S3 objects)
   - Cache entries
   - Log entries (or redact)
   - Backups (after retention period)
5. Release resources

TENANT MIGRATION (moving between tiers):
─────────────────────────────────────────
Free → Pro:
  1. Tenant stays on shared DB
  2. Increase rate limits
  3. Enable premium features

Pro → Enterprise:
  1. Provision dedicated DB
  2. Migrate data from shared DB to dedicated
  3. Update routing rules
  4. Verify data integrity
  5. Switch DNS
  6. Delete from shared DB
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Missing tenant filter in a query | DATA LEAK between tenants | Use RLS, ORM-level filters, or framework middleware |
| Shared cache without tenant prefix | Tenant A sees Tenant B's cached data | Always prefix: `tenant:{id}:key` |
| No rate limiting per tenant | One tenant's spike kills everyone | Implement per-tenant rate limits and quotas |
| Same encryption key for all tenants | Compromising one key exposes all data | Per-tenant encryption keys (envelope encryption) |
| Hard-coded to one isolation model | Can't upgrade tenants to dedicated infra | Design routing layer to support multiple models |
| Global admin endpoints without tenant scope | Admin accidentally sees all tenant data | Enforce tenant context even for internal tools |
| Not testing tenant isolation | Bugs leak data silently | Automated tests that verify cross-tenant access is blocked |
| Storing tenant config in code | Can't onboard new tenants without deploy | Store tenant config in database or config service |

---

## When to Use / When NOT to Use

### Use Multi-Tenancy When:
- Building a SaaS product serving multiple organizations
- Cost efficiency matters (can't afford N servers for N customers)
- Features are mostly identical across tenants (customization via config)
- You need rapid tenant provisioning (signup → ready in minutes)
- Operating many instances would be operationally infeasible

### Use Single-Tenancy When:
- Regulatory requirements demand complete isolation (banking, healthcare)
- Tenants have vastly different resource requirements
- Data sovereignty laws require physical data separation
- You have very few, very large customers (consulting firms)
- Security requirements are extreme (government, defense)
- Each customer needs heavy customization (different code per tenant)

### Hybrid (Most Common):
- Start multi-tenant for smaller customers
- Offer dedicated infrastructure for enterprise customers
- Gradually move large tenants to dedicated resources as they grow

---

## Key Takeaways

1. **Multi-tenancy is a spectrum** — From fully shared (tenant_id column) to fully isolated (dedicated servers). Most production systems use a hybrid approach.

2. **Data isolation is non-negotiable** — One tenant must NEVER see another's data. Use database-level enforcement (RLS, schema separation) — don't rely only on application code.

3. **The noisy neighbor problem is real** — One tenant's spike can destroy everyone's experience. Rate limiting, resource quotas, and connection pooling are essential.

4. **Tenant identification must be foolproof** — Extract tenant from JWT claims (secure), not just URL parameters (spoofable). Validate at every layer.

5. **Design for migration between tiers** — A tenant that starts on shared infrastructure should be movable to dedicated infrastructure without data loss or downtime.

6. **Cache and queue keys must be tenant-scoped** — Every shared resource (Redis, RabbitMQ, S3 buckets) needs tenant namespacing to prevent data leaks.

7. **Start simple, evolve** — Begin with shared schema + tenant_id for your first 100 customers. Add schema separation or dedicated DBs as you get enterprise customers with specific requirements.

---

## What's Next?

Congratulations! You've completed **Part 25: Design Patterns & Best Practices for Large Systems**. These patterns form the foundation of how planet-scale applications are built and operated.

Continue to **Part 26: Cost Optimization & Capacity Planning** to learn how to right-size your infrastructure and manage cloud costs effectively.

See: [../26-cost-optimization/01-server-sizing.md](../26-cost-optimization/01-server-sizing.md)
