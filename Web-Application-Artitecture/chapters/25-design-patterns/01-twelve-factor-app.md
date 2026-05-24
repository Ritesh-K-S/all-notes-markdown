# The 12-Factor App Methodology

> **What you'll learn**: The 12-Factor App is a set of principles for building modern, cloud-native applications that are portable, scalable, and maintainable — and why every serious production app follows these rules (whether they know it or not).

---

## Real-Life Analogy

Imagine you're packing for a trip. A **bad packer** throws everything loose in a suitcase — charger cables tangled with clothes, liquids leaking on documents, and they can never find anything.

A **great packer** uses a system:
- Each item type goes in its own labeled bag (separation of concerns)
- Nothing spills into another compartment (isolation)
- You can swap one item without unpacking everything (loose coupling)
- The packing list is the same whether you're going to Delhi or Dubai (environment parity)

The 12-Factor App is that packing system — but for software. It tells you **how to organize your code, config, dependencies, and processes** so your app works perfectly whether it's on your laptop, a staging server, or a 500-node production cluster.

---

## Why Does This Exist?

In 2012, engineers at Heroku (a cloud platform) noticed that most app failures in production weren't caused by bad code — they were caused by **bad practices** around deployment, configuration, and environment management.

They distilled their experience into **12 principles** that make apps:
- Easy to deploy on any cloud
- Easy to scale up or down
- Easy to hand off to new developers
- Resilient to environment differences

```
Traditional App Problems           │  12-Factor Solutions
───────────────────────────────────┼──────────────────────────────────
Config hardcoded in code           │  Config from environment variables
"Works on my machine" syndrome     │  Dev/prod parity
Stateful servers that can't scale  │  Stateless processes
Coupled dependencies               │  Explicit dependency declaration
Manual deployment steps            │  Automated, repeatable deploys
```

---

## The 12 Factors — One by One

### Factor 1: Codebase — One Codebase, Many Deploys

> **One codebase tracked in version control, many deploys.**

```
                    ┌─────────────┐
                    │  Git Repo   │
                    │ (Codebase)  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │   Dev    │ │ Staging  │ │Production│
        │  Deploy  │ │  Deploy  │ │  Deploy  │
        └──────────┘ └──────────┘ └──────────┘
```

**Rule**: One app = one repository. Multiple apps = multiple repos. The SAME codebase is deployed to dev, staging, and production. The difference between environments is **configuration**, not code.

**Violation**: Having separate code repos for "dev version" and "prod version" of the same app.

---

### Factor 2: Dependencies — Explicitly Declare and Isolate

> **Never rely on the system having something pre-installed.**

Your app must **declare** all dependencies in a manifest file and **isolate** them so nothing leaks from the system.

| Language | Manifest File | Isolation Tool |
|----------|---------------|----------------|
| Python | `requirements.txt` / `pyproject.toml` | `venv` / `virtualenv` |
| Java | `pom.xml` / `build.gradle` | Maven/Gradle dependency resolution |
| Node.js | `package.json` | `node_modules` |
| Go | `go.mod` | Go modules |

**Violation**: "Just `apt-get install imagemagick` on the server before deploying" — this is an implicit, undeclared system dependency.

---

### Factor 3: Config — Store Config in the Environment

> **Configuration that varies between deploys should be stored in environment variables.**

```
┌─────────────────────────────────────────────────────────┐
│                        CODE                              │
│  (Same across all environments — never changes)         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    CONFIGURATION                         │
│  (Different per environment — DB URLs, API keys, ports) │
│  Stored in: ENV VARS, not in code!                      │
└─────────────────────────────────────────────────────────┘
```

**Good**:
```python
import os
database_url = os.environ["DATABASE_URL"]
```

**Bad**:
```python
database_url = "postgresql://user:password@prod-db:5432/myapp"  # NEVER DO THIS!
```

**Why?** If your config is in code, you need to change code to deploy to a different environment. With env vars, the same artifact works everywhere.

---

### Factor 4: Backing Services — Treat Them as Attached Resources

> **Treat databases, queues, email services, and caches as swappable resources.**

```
Your App ──▶ PostgreSQL (local)     ← Can swap to...
Your App ──▶ Amazon RDS (managed)   ← ...without code changes

Your App ──▶ Local Redis            ← Can swap to...
Your App ──▶ AWS ElastiCache        ← ...without code changes
```

Your app connects to backing services via **URLs/credentials stored in config** (Factor 3). You should be able to swap a local PostgreSQL for Amazon RDS just by changing an environment variable — no code changes needed.

---

### Factor 5: Build, Release, Run — Strictly Separate Build and Run Stages

> **Codebase goes through three stages to become a running process.**

```
┌──────────┐      ┌──────────┐      ┌──────────┐
│  BUILD   │─────▶│ RELEASE  │─────▶│   RUN    │
│          │      │          │      │          │
│ Compile  │      │ Build +  │      │ Execute  │
│ Install  │      │ Config = │      │ process  │
│ deps     │      │ Release  │      │ in env   │
└──────────┘      └──────────┘      └──────────┘
```

- **Build**: Convert code into an executable bundle (compile, install deps, bundle assets)
- **Release**: Combine build with config for a specific environment
- **Run**: Execute the release in the target environment

**Key rule**: You can NEVER change code at runtime. If you need a change, go through the full build → release → run pipeline again.

---

### Factor 6: Processes — Execute the App as Stateless Processes

> **Processes should be stateless and share-nothing.**

```
Request A ──▶ ┌──────────┐
              │ Server 1 │──▶ Response A
Request B ──▶ └──────────┘
                              
Request C ──▶ ┌──────────┐
              │ Server 2 │──▶ Response C
Request D ──▶ └──────────┘

Both servers are IDENTICAL. No local state.
Any request can go to ANY server.
```

**Stateless means**: Don't store user sessions, file uploads, or cache in the process memory. Use external services (Redis for sessions, S3 for files, Memcached for cache).

**Why?** If a server crashes or scales down, all its in-memory data is lost. Stateless processes can be killed and restarted without data loss.

---

### Factor 7: Port Binding — Export Services via Port Binding

> **Your app is completely self-contained and exposes itself by binding to a port.**

Your app doesn't rely on an external web server (like Apache or Tomcat being pre-installed). Instead, it embeds an HTTP server and listens on a port:

```python
# Python Flask - self-contained
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
```

```java
// Spring Boot - embedded Tomcat
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);  // Starts on port 8080
    }
}
```

The platform (Kubernetes, Heroku, etc.) routes traffic to your app's port. Your app doesn't care what's outside — it just listens.

---

### Factor 8: Concurrency — Scale Out via the Process Model

> **Scale by running more processes, not by making one process bigger.**

```
WRONG: One giant process using 64 GB RAM
─────────────────────────────────────────

RIGHT: Many small processes, each using 512 MB
┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
│ web │ │ web │ │ web │ │work │ │work │
│  1  │ │  2  │ │  3  │ │  1  │ │  2  │
└─────┘ └─────┘ └─────┘ └─────┘ └─────┘
  HTTP     HTTP    HTTP   Background jobs
```

Different work types get different process types:
- **Web processes**: Handle HTTP requests
- **Worker processes**: Handle background jobs
- **Clock processes**: Handle scheduled tasks

Each can scale independently. Need more web capacity? Add more web processes.

---

### Factor 9: Disposability — Fast Startup, Graceful Shutdown

> **Processes should start fast and shut down gracefully.**

```
START: App boots in < 5 seconds
──────────────────────────────

SHUTDOWN: 
1. Receive SIGTERM signal
2. Stop accepting new requests
3. Finish current requests (with timeout)
4. Close database connections
5. Exit cleanly
```

**Why fast startup?** Auto-scaling and deploys need to spin up new instances quickly.

**Why graceful shutdown?** You don't want to kill a process mid-transaction. Finish what you're doing, then die cleanly.

---

### Factor 10: Dev/Prod Parity — Keep Environments Similar

> **Keep development, staging, and production as similar as possible.**

| Gap | Traditional App | 12-Factor App |
|-----|----------------|---------------|
| **Time gap** | Weeks between deploys | Hours or minutes |
| **Personnel gap** | Developers write, ops deploy | Developers deploy their own code |
| **Tools gap** | SQLite in dev, PostgreSQL in prod | Same tools everywhere |

**Rule**: If you use PostgreSQL in production, use PostgreSQL in development. Don't use SQLite locally "because it's easier." Docker makes this trivial — run the same services locally that you run in prod.

---

### Factor 11: Logs — Treat Logs as Event Streams

> **Don't manage log files yourself. Write to stdout and let the platform handle it.**

```
Your App ──▶ stdout ──▶ Platform captures ──▶ Log aggregation
                                                (ELK, Splunk,
                                                 CloudWatch)

NOT THIS:
Your App ──▶ /var/log/app.log  ← Don't manage files!
```

Your app should never concern itself with routing or storing logs. Just write to `stdout`. The execution environment (Docker, Kubernetes, Heroku) will capture and route logs to the right place.

---

### Factor 12: Admin Processes — Run Admin Tasks as One-Off Processes

> **Run management tasks (migrations, scripts, console) as one-off processes in the same environment.**

```bash
# Database migration
kubectl exec -it myapp-pod -- python manage.py migrate

# Open a REPL/console
kubectl exec -it myapp-pod -- python manage.py shell

# Run a data fix script
kubectl exec -it myapp-pod -- python scripts/fix_data.py
```

Admin processes run against the SAME release (same code + config) as your regular app. They are NOT separate tools or scripts that live outside your codebase.

---

## How It Works Internally — Putting All 12 Together

Here's what a fully 12-factor app looks like in practice:

```
┌─────────────────────────────────────────────────────────────────┐
│                     GIT REPOSITORY                               │
│                                                                   │
│  app/                    ← Application code                      │
│  requirements.txt        ← Factor 2: Declared dependencies       │
│  Dockerfile              ← Factor 5: Build stage                 │
│  Procfile                ← Factor 8: Process types               │
│  .env.example            ← Factor 3: Config template             │
│  migrations/             ← Factor 12: Admin processes            │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼ CI/CD Pipeline (Factor 5: Build)
┌─────────────────────────────────────────────────────────────────┐
│  Docker Image (immutable artifact)                               │
│  + Environment Variables (Factor 3)                              │
│  = RELEASE                                                       │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼ Kubernetes / Cloud Platform (Factor 5: Run)
┌─────────────────────────────────────────────────────────────────┐
│  ┌─────┐ ┌─────┐ ┌─────┐    ← Factor 6: Stateless              │
│  │Pod 1│ │Pod 2│ │Pod 3│    ← Factor 8: Scale out               │
│  └──┬──┘ └──┬──┘ └──┬──┘    ← Factor 9: Disposable             │
│     │       │       │                                            │
│     └───────┴───────┘                                            │
│             │                                                    │
│     ┌───────┴───────┐                                            │
│     ▼               ▼                                            │
│  ┌──────┐     ┌──────────┐   ← Factor 4: Backing services       │
│  │Redis │     │PostgreSQL│                                       │
│  └──────┘     └──────────┘                                       │
│                                                                   │
│  stdout ──▶ CloudWatch          ← Factor 11: Log streams         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: A 12-Factor App Structure

```python
# app.py - Factor 7: Port binding, Factor 3: Config from env
import os
from flask import Flask, jsonify
import redis
import psycopg2

app = Flask(__name__)

# Factor 3: All config from environment variables
DATABASE_URL = os.environ["DATABASE_URL"]
REDIS_URL = os.environ["REDIS_URL"]
PORT = int(os.environ.get("PORT", 5000))

# Factor 4: Backing services as attached resources
cache = redis.from_url(REDIS_URL)

def get_db_connection():
    return psycopg2.connect(DATABASE_URL)

@app.route("/health")
def health():
    return jsonify({"status": "ok"})

@app.route("/users/<int:user_id>")
def get_user(user_id):
    # Factor 11: Log to stdout
    print(f"Fetching user {user_id}")
    
    # Check cache first
    cached = cache.get(f"user:{user_id}")
    if cached:
        return jsonify({"user": cached.decode(), "source": "cache"})
    
    # Factor 4: Database is an attached resource
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("SELECT name FROM users WHERE id = %s", (user_id,))
    user = cur.fetchone()
    conn.close()
    
    if user:
        cache.setex(f"user:{user_id}", 300, user[0])
        return jsonify({"user": user[0], "source": "db"})
    return jsonify({"error": "not found"}), 404

# Factor 7: Self-contained, binds to a port
if __name__ == "__main__":
    app.run(host="0.0.0.0", port=PORT)
```

### Java: A 12-Factor Spring Boot App

```java
// Application.java - Factor 7: Embedded server
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// application.yml - Factor 3: Config references env vars
// spring:
//   datasource:
//     url: ${DATABASE_URL}
//   redis:
//     host: ${REDIS_HOST}
//     port: ${REDIS_PORT:6379}
// server:
//   port: ${PORT:8080}

// UserController.java
@RestController
@Slf4j
public class UserController {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @GetMapping("/users/{id}")
    public ResponseEntity<?> getUser(@PathVariable Long id) {
        // Factor 11: Log to stdout (SLF4J → stdout)
        log.info("Fetching user {}", id);
        
        // Factor 4: Redis as an attached backing service
        String cached = redisTemplate.opsForValue().get("user:" + id);
        if (cached != null) {
            return ResponseEntity.ok(Map.of("user", cached, "source", "cache"));
        }
        
        return userRepository.findById(id)
            .map(user -> {
                redisTemplate.opsForValue().set("user:" + id, user.getName(), 
                    Duration.ofMinutes(5));
                return ResponseEntity.ok(Map.of("user", user.getName(), "source", "db"));
            })
            .orElse(ResponseEntity.notFound().build());
    }
}
```

### Dockerfile (Factor 5: Build Stage)

```dockerfile
# Factor 2: Explicit dependencies installed in isolation
FROM python:3.11-slim

WORKDIR /app

# Factor 2: Declare dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app code
COPY . .

# Factor 7: Expose port
EXPOSE 5000

# Factor 9: Fast startup
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "app:app"]
```

### Procfile (Factor 8: Process Types)

```
web: gunicorn --bind 0.0.0.0:$PORT --workers 4 app:app
worker: celery -A tasks worker --loglevel=info
clock: celery -A tasks beat --loglevel=info
```

---

## Infrastructure Example — Kubernetes Deployment

```yaml
# Factor 6: Stateless processes
# Factor 8: Scale via replicas
# Factor 3: Config via env vars
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3  # Factor 8: Horizontal scaling
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v1.2.3  # Factor 5: Immutable release
        ports:
        - containerPort: 5000  # Factor 7: Port binding
        env:
        - name: DATABASE_URL  # Factor 3: Config from env
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: myapp-config
              key: redis-url
        readinessProbe:  # Factor 9: Fast startup detection
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
        lifecycle:
          preStop:  # Factor 9: Graceful shutdown
            exec:
              command: ["sh", "-c", "sleep 5"]
```

---

## Real-World Example

### Heroku (The Creators)

Heroku literally invented this methodology based on hundreds of thousands of app deployments:
- Every app is a git repo (Factor 1)
- `Procfile` defines process types (Factor 8)
- Config is set via `heroku config:set KEY=value` (Factor 3)
- Dynos (containers) are stateless and disposable (Factor 6, 9)
- Logs stream to `heroku logs --tail` (Factor 11)

### Netflix

Netflix follows every factor:
- **Factor 6**: All services are stateless — state lives in Cassandra, EVCache, S3
- **Factor 8**: Services scale to hundreds of instances per region
- **Factor 9**: New instances boot in under 60 seconds
- **Factor 10**: They test in production environments (Chaos Engineering)
- **Factor 4**: Backing services (databases, caches) are all configured via service discovery + env config

### Spotify

Spotify's "Backstage" platform enforces 12-factor principles:
- Every microservice has a `catalog-info.yaml` (equivalent of declaring dependencies)
- All services log to stdout → captured by their logging pipeline
- Config is injected via Kubernetes secrets and configmaps

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Storing secrets in code/git | Security breach + can't change without deploy | Use env vars or secrets manager (Vault, AWS Secrets Manager) |
| Storing sessions in-memory | Can't scale horizontally; session lost on restart | Use Redis or database for session storage |
| Writing logs to files | Files fill disk; lost when container dies | Write to stdout; let platform aggregate |
| Different DB in dev vs prod | "Works on my machine" bugs | Use Docker Compose to run same services locally |
| Long startup times (>30s) | Auto-scaling is slow; deploys take forever | Reduce initialization work; lazy-load where possible |
| Hardcoding port numbers | Can't run multiple instances locally | Use `PORT` env var |
| Manual deployment steps | Human error; not repeatable | Automate everything in CI/CD pipeline |

---

## When to Use / When NOT to Use

### Use 12-Factor When:
- Building cloud-native applications
- Deploying to containers (Docker/Kubernetes)
- Building microservices
- Using CI/CD pipelines
- Working with multiple environments (dev/staging/prod)
- Your app needs to scale horizontally
- You're working in a team (need reproducibility)

### May Not Fully Apply When:
- Building a simple script or CLI tool
- Embedded systems or IoT firmware
- Desktop applications
- One-off data processing jobs (though many factors still apply)
- Legacy applications you can't refactor (apply factors incrementally)

> **Tip**: Even for small projects, following factors 1, 2, 3, 5, and 11 costs nothing and saves tremendous pain later.

---

## Key Takeaways

1. **12-Factor is a checklist, not a framework** — it's a set of principles that work with ANY language, framework, or platform.

2. **Factor 3 (Config) is the most violated** — hardcoded config is the #1 source of deployment failures. Always use environment variables.

3. **Factor 6 (Stateless) enables scaling** — you cannot horizontally scale an app that stores state in memory. Push state to external services.

4. **Factor 10 (Dev/Prod Parity) prevents surprises** — use Docker Compose to run the exact same databases and services locally that you use in production.

5. **Factor 9 (Disposability) enables resilience** — if your app can start in 5 seconds and shutdown gracefully, auto-scaling and self-healing become trivial.

6. **These principles compound** — following all 12 factors together creates an app that is dramatically easier to deploy, scale, debug, and maintain than one following only a few.

7. **Modern tools enforce these patterns** — Kubernetes, Docker, and cloud platforms naturally push you toward 12-factor apps. Fighting these principles makes everything harder.

---

## What's Next?

Next, we'll explore **Domain-Driven Design (DDD)** — a methodology for structuring your application's business logic in a way that scales with complexity. While 12-Factor tells you how to deploy and operate your app, DDD tells you how to organize the code inside it.

See: [02-domain-driven-design.md](./02-domain-driven-design.md)
