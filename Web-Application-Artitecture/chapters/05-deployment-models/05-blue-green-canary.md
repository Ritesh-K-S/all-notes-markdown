# Blue-Green & Canary Deployments — Zero Downtime Releases

> **What you'll learn**: How to deploy new versions of your application without any downtime — using Blue-Green deployments (instant switch) and Canary deployments (gradual rollout) — so users never notice you shipped new code.

---

## Real-Life Analogy

### Blue-Green: The Theater Stage Trick

Imagine a theater performing a play. Behind the main stage, there's a **second identical stage** hidden by a curtain. While the audience watches Act 1 on Stage A (Blue), stagehands are setting up Act 2 on Stage B (Green) — moving props, adjusting lights, doing sound checks.

When everything is ready and tested on Stage B, someone **pulls the curtain** — instantly switching the audience's view to the new stage. If something goes wrong with Act 2, they simply pull the curtain BACK to Stage A.

The audience (your users) never saw the transition. Zero downtime.

### Canary: The Coal Mine Canary

Miners used to bring a canary bird into coal mines. If there was toxic gas, the canary would die first (being more sensitive), giving miners time to escape.

In **Canary deployments**, you send a tiny portion of your traffic (1-5%) to the new version first. If the new version "dies" (errors, crashes, slow), only 1-5% of users are affected — and you can pull back instantly. The "canary" died, but your "miners" (95% of users) are safe.

---

## Core Concept Explained Step-by-Step

### The Old Way: Downtime Deployments

```
THE BAD OLD DAYS:

1. Take server offline         → Users see "503 Service Unavailable"
2. Upload new code             → Users still waiting...
3. Run database migrations     → Users STILL waiting... (5-15 min)
4. Restart application         → Users still getting errors
5. Verify it works             → Finally back online!

Total downtime: 5-30 minutes
User experience: 😱 "Is the site down? Should I try a competitor?"
```

### Blue-Green Deployment

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BLUE-GREEN DEPLOYMENT                             │
│                                                                     │
│  STEP 1: Current state (Blue is LIVE)                               │
│                                                                     │
│  Users ──▶ Load Balancer ──▶ 🔵 BLUE (v1.0) ← serving traffic     │
│                              🟢 GREEN (empty/old) ← idle            │
│                                                                     │
│  STEP 2: Deploy new version to GREEN                                │
│                                                                     │
│  Users ──▶ Load Balancer ──▶ 🔵 BLUE (v1.0) ← still serving!      │
│                              🟢 GREEN (v2.0) ← deploying, testing   │
│                                                                     │
│  STEP 3: Test GREEN thoroughly (smoke tests, health checks)         │
│                                                                     │
│  Users ──▶ Load Balancer ──▶ 🔵 BLUE (v1.0) ← still serving!      │
│                              🟢 GREEN (v2.0) ← tests pass ✓        │
│                                                                     │
│  STEP 4: SWITCH! Route all traffic to GREEN                         │
│                                                                     │
│  Users ──▶ Load Balancer ──▶ 🟢 GREEN (v2.0) ← NOW serving!       │
│                              🔵 BLUE (v1.0) ← idle (rollback ready)│
│                                                                     │
│  STEP 5 (if needed): ROLLBACK! Switch back to BLUE instantly        │
│                                                                     │
│  Users ──▶ Load Balancer ──▶ 🔵 BLUE (v1.0) ← back to serving!    │
│                              🟢 GREEN (v2.0) ← broken, will fix    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

KEY INSIGHT: The switch is INSTANT (just changing where the load balancer points).
Users experience ZERO downtime.
```

### Canary Deployment

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CANARY DEPLOYMENT                                │
│                                                                     │
│  STEP 1: All traffic goes to v1 (current version)                   │
│                                                                     │
│  Users (100%) ──▶ Load Balancer ──▶ v1.0 (10 instances)           │
│                                                                     │
│  STEP 2: Deploy v2 to ONE instance (the "canary")                   │
│                                                                     │
│  Users (95%) ───▶ Load Balancer ──┬──▶ v1.0 (9 instances)         │
│  Users (5%) ────▶                 └──▶ v2.0 (1 instance) ← canary │
│                                                                     │
│  STEP 3: Monitor canary for 15-30 minutes                           │
│    ✓ Error rate same as v1?                                         │
│    ✓ Latency same as v1?                                            │
│    ✓ No crashes?                                                    │
│    ✓ Business metrics normal? (conversions, signups)                │
│                                                                     │
│  STEP 4a: Canary is HEALTHY → gradually increase                    │
│                                                                     │
│  Users (75%) ───▶ Load Balancer ──┬──▶ v1.0 (7 instances)         │
│  Users (25%) ───▶                 └──▶ v2.0 (3 instances)          │
│                                                                     │
│  STEP 4b: Canary is SICK → rollback immediately                     │
│                                                                     │
│  Users (100%) ──▶ Load Balancer ──▶ v1.0 (10 instances)           │
│                                    v2.0 REMOVED ✗                   │
│                                                                     │
│  STEP 5: If healthy at each stage → 50% → 75% → 100%               │
│                                                                     │
│  Users (100%) ──▶ Load Balancer ──▶ v2.0 (10 instances) ← DONE!   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Comparison: Blue-Green vs Canary

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Feature           │ Blue-Green         │ Canary                    │
│  ──────────────────┼────────────────────┼──────────────────────     │
│  Switch speed      │ Instant (all at    │ Gradual (1%→10%→50%→100%) │
│                    │ once)              │                           │
│  Risk              │ Medium (all users  │ Low (only % affected)     │
│                    │ get new version    │                           │
│                    │ at once)           │                           │
│  Rollback speed    │ Instant (switch    │ Fast (route back to v1)   │
│                    │ back to blue)      │                           │
│  Infrastructure    │ 2x resources       │ 1x + small canary         │
│                    │ (both envs running)│                           │
│  Testing           │ Tested BEFORE      │ Tested WITH real traffic  │
│                    │ switch             │                           │
│  Best for          │ Critical releases, │ Feature releases,         │
│                    │ database changes   │ uncertain changes          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Blue-Green: The DNS/Load Balancer Switch

```
METHOD 1: DNS Switch (slower — DNS caching delay)

Before:  api.myapp.com → 10.0.1.100 (Blue servers)
After:   api.myapp.com → 10.0.2.100 (Green servers)

Problem: DNS TTL means some clients use old IP for minutes!


METHOD 2: Load Balancer Target Switch (instant — preferred!)

Before:
┌──────────┐     ┌───────────────┐     ┌──────────────────────┐
│  Client  │────▶│    ALB        │────▶│  Target Group: BLUE  │
└──────────┘     │               │     │  10.0.1.10           │
                 │ api.myapp.com │     │  10.0.1.11           │
                 └───────────────┘     │  10.0.1.12           │
                                       └──────────────────────┘

After (one API call to switch):
┌──────────┐     ┌───────────────┐     ┌──────────────────────┐
│  Client  │────▶│    ALB        │────▶│ Target Group: GREEN  │
└──────────┘     │               │     │  10.0.2.10           │
                 │ api.myapp.com │     │  10.0.2.11           │
                 └───────────────┘     │  10.0.2.12           │
                                       └──────────────────────┘

Switch is ONE API call: update the ALB listener rule.
All new requests go to Green. In-flight Blue requests finish naturally.
```

### Canary: Weighted Routing

```
HOW LOAD BALANCER SPLITS TRAFFIC:

┌─────────────────────────────────────────────────────────┐
│            Load Balancer (Weighted Routing)              │
│                                                         │
│  Rule: /api/* →                                         │
│    95% weight → Target Group A (v1.0, 9 instances)     │
│     5% weight → Target Group B (v2.0, 1 instance)      │
│                                                         │
│  Implementation:                                        │
│  For every 100 requests:                               │
│    Request 1-95   → v1.0                               │
│    Request 96-100 → v2.0 (canary)                      │
└─────────────────────────────────────────────────────────┘

MONITORING THE CANARY:

┌──────────────────────────────────────────────────────────────┐
│  Metric              │ v1.0 (stable) │ v2.0 (canary) │ OK?  │
│  ────────────────────┼───────────────┼───────────────┼──── │
│  Error rate          │    0.1%       │    0.1%       │  ✓  │
│  P99 latency         │    120ms      │    125ms      │  ✓  │
│  P50 latency         │    45ms       │    44ms       │  ✓  │
│  5xx responses       │    2          │    0          │  ✓  │
│  CPU usage           │    45%        │    47%        │  ✓  │
│                      │               │               │      │
│  Verdict: CANARY IS HEALTHY → Increase to 25%              │
└──────────────────────────────────────────────────────────────┘

IF CANARY IS UNHEALTHY:
┌──────────────────────────────────────────────────────────────┐
│  Metric              │ v1.0 (stable) │ v2.0 (canary) │ OK?  │
│  ────────────────────┼───────────────┼───────────────┼──── │
│  Error rate          │    0.1%       │    5.2%       │  ✗! │
│  P99 latency         │    120ms      │    800ms      │  ✗! │
│                      │               │               │      │
│  Verdict: CANARY IS SICK → ROLLBACK! Remove v2.0 instantly  │
└──────────────────────────────────────────────────────────────┘
```

### Handling Database Migrations in Blue-Green

```
THE TRICKY PART: Database schema changes

WRONG approach:
┌──────┐                       ┌──────┐
│ Blue │──▶ DB (schema v1) ←──│Green │
│ v1.0 │                       │ v2.0 │ needs schema v2!
└──────┘                       └──────┘

If Green needs a NEW column → you change the DB → Blue breaks!


CORRECT approach: "Expand and Contract" migrations

Step 1: EXPAND (add new column, keep old one working)
┌──────┐                       ┌──────┐
│ Blue │──▶ DB (schema v1+v2) ←──│Green │
│ v1.0 │  (reads old column)   │ v2.0 │  (reads new column)
└──────┘                       └──────┘
Both versions work with the current schema!

Step 2: Switch traffic to Green (v2.0)

Step 3: CONTRACT (remove old column after Blue is gone)
- Only do this AFTER Blue is decommissioned
- Now DB is clean with only v2 schema

Timeline:
Day 1: Add new column (both versions work)  ← EXPAND
Day 2: Deploy Green with v2 code
Day 3: Switch traffic Blue → Green
Day 7: Remove old column                    ← CONTRACT (safe now)
```

---

## Code Examples

### Python (Blue-Green Deployment Script)

```python
# deploy_blue_green.py — Automated Blue-Green deployment on AWS
import boto3
import time
import requests

elbv2 = boto3.client('elbv2')
ecs = boto3.client('ecs')

LISTENER_ARN = "arn:aws:elasticloadbalancing:...:listener/..."
BLUE_TARGET_GROUP = "arn:aws:elasticloadbalancing:...:targetgroup/blue/..."
GREEN_TARGET_GROUP = "arn:aws:elasticloadbalancing:...:targetgroup/green/..."
GREEN_HEALTH_URL = "http://green-internal.myapp.com/health"

def deploy_blue_green(new_version: str):
    """Deploy new version using Blue-Green strategy."""
    
    # Step 1: Deploy new version to GREEN environment
    print(f"Deploying {new_version} to GREEN...")
    update_ecs_service("green-service", new_version)
    
    # Step 2: Wait for GREEN to be healthy
    print("Waiting for GREEN instances to be healthy...")
    wait_for_healthy(GREEN_TARGET_GROUP, timeout=300)
    
    # Step 3: Run smoke tests against GREEN
    print("Running smoke tests on GREEN...")
    if not run_smoke_tests(GREEN_HEALTH_URL):
        print("SMOKE TESTS FAILED! Aborting deployment.")
        return False
    
    # Step 4: SWITCH traffic from BLUE to GREEN
    print("Switching traffic BLUE → GREEN...")
    elbv2.modify_listener(
        ListenerArn=LISTENER_ARN,
        DefaultActions=[{
            'Type': 'forward',
            'TargetGroupArn': GREEN_TARGET_GROUP  # All traffic to GREEN!
        }]
    )
    
    print(f"✅ Deployment complete! v{new_version} is now LIVE on GREEN.")
    print("   BLUE is idle and ready for instant rollback.")
    return True

def rollback():
    """Instant rollback: switch traffic back to BLUE."""
    print("⚠️  ROLLING BACK! Switching traffic GREEN → BLUE...")
    elbv2.modify_listener(
        ListenerArn=LISTENER_ARN,
        DefaultActions=[{
            'Type': 'forward',
            'TargetGroupArn': BLUE_TARGET_GROUP  # Back to BLUE!
        }]
    )
    print("✅ Rollback complete. BLUE is serving traffic again.")

def run_smoke_tests(health_url: str) -> bool:
    """Verify the new version is working before switching."""
    try:
        resp = requests.get(health_url, timeout=5)
        return resp.status_code == 200
    except Exception:
        return False

def wait_for_healthy(target_group_arn: str, timeout: int):
    """Wait until all targets in the group are healthy."""
    start = time.time()
    while time.time() - start < timeout:
        response = elbv2.describe_target_health(TargetGroupArn=target_group_arn)
        healths = [t['TargetHealth']['State'] for t in response['TargetHealthDescriptions']]
        if all(h == 'healthy' for h in healths) and len(healths) > 0:
            return
        time.sleep(10)
    raise TimeoutError("GREEN targets did not become healthy in time!")
```

### Java (Canary Deployment with Spring Cloud Gateway)

```java
// CanaryRoutingConfig.java — Route % of traffic to canary version
@Configuration
public class CanaryRoutingConfig {
    
    @Value("${canary.weight:0}")  // 0-100, controlled externally
    private int canaryWeight;
    
    @Bean
    public RouteLocator canaryRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            // Canary route: send canaryWeight% to new version
            .route("canary", r -> r
                .path("/api/**")
                .and()
                .weight("backend", canaryWeight)
                .uri("http://backend-v2:8080"))  // New version
            
            // Stable route: send (100 - canaryWeight)% to current version
            .route("stable", r -> r
                .path("/api/**")
                .and()
                .weight("backend", 100 - canaryWeight)
                .uri("http://backend-v1:8080"))  // Current version
            
            .build();
    }
}

// CanaryController.java — Endpoint to adjust canary percentage
@RestController
@RequestMapping("/admin/canary")
public class CanaryController {
    
    @Autowired
    private ConfigurableEnvironment environment;
    
    @PostMapping("/weight")
    public ResponseEntity<String> setCanaryWeight(@RequestParam int weight) {
        if (weight < 0 || weight > 100) {
            return ResponseEntity.badRequest().body("Weight must be 0-100");
        }
        // Update canary weight dynamically
        System.setProperty("canary.weight", String.valueOf(weight));
        return ResponseEntity.ok("Canary weight set to " + weight + "%");
    }
}
```

### Nginx Canary Configuration

```nginx
# nginx.conf — Canary deployment with weighted upstream
http {
    # Split traffic using a map on a cookie or random
    split_clients "${remote_addr}${request_uri}" $backend_version {
        5%    canary;      # 5% to new version
        *     stable;      # 95% to stable version
    }

    upstream stable {
        server 10.0.1.10:8080;
        server 10.0.1.11:8080;
        server 10.0.1.12:8080;
    }

    upstream canary {
        server 10.0.2.10:8080;  # New version instance(s)
    }

    server {
        listen 80;
        
        location /api/ {
            # Route based on split_clients result
            proxy_pass http://$backend_version;
            
            # Add header so we can trace which version served the request
            add_header X-Served-By $backend_version;
            proxy_set_header Host $host;
        }
    }
}
```

---

## Infrastructure Example

### AWS Blue-Green with ECS

```
┌─────────────────────────────────────────────────────────────────────┐
│                  AWS BLUE-GREEN WITH ECS                             │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │              Application Load Balancer                   │       │
│  │                                                         │       │
│  │  Listener :443 ─── Rule: Forward to ──┐                │       │
│  └─────────────────────────────────────────┼───────────────┘       │
│                                            │                        │
│              ┌─────────────────────────────┼────────┐               │
│              │                             ▼        │               │
│              │                    ┌───────────────┐ │               │
│              │                    │ Target Group  │ │               │
│              │                    │   (GREEN)     │ │               │
│              │                    │               │ │               │
│              │                    │  Task 1 (v2)  │ │               │
│              │                    │  Task 2 (v2)  │ │               │
│              │                    │  Task 3 (v2)  │ │               │
│              │                    └───────────────┘ │               │
│              │                                      │               │
│              │     (idle)         ┌───────────────┐ │               │
│              │                    │ Target Group  │ │               │
│              │                    │   (BLUE)      │ │               │
│              │                    │               │ │               │
│              │                    │  Task 1 (v1)  │ │               │
│              │                    │  Task 2 (v1)  │ │               │
│              │                    │  Task 3 (v1)  │ │               │
│              │                    └───────────────┘ │               │
│              │                                      │               │
│              │     ECS Cluster                      │               │
│              └──────────────────────────────────────┘               │
│                                                                     │
│  Switch = Change ALB Listener Rule: BLUE → GREEN (1 API call!)     │
│  Rollback = Change it back: GREEN → BLUE (1 API call!)             │
└─────────────────────────────────────────────────────────────────────┘
```

### Kubernetes Canary with Istio

```yaml
# VirtualService — Istio canary routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend-routing
spec:
  hosts:
  - backend-service
  http:
  - route:
    - destination:
        host: backend-service
        subset: stable    # v1.0
      weight: 95          # 95% of traffic
    - destination:
        host: backend-service
        subset: canary    # v2.0
      weight: 5           # 5% of traffic (the canary!)
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend-versions
spec:
  host: backend-service
  subsets:
  - name: stable
    labels:
      version: v1.0
  - name: canary
    labels:
      version: v2.0
```

### Automated Canary with Argo Rollouts

```yaml
# rollout.yaml — Argo Rollouts automatic canary with analysis
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: backend-app
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 5       # Step 1: 5% traffic to canary
      - pause:
          duration: 5m     # Wait 5 min, check metrics
      - analysis:
          templates:
          - templateName: success-rate  # Auto-check error rate
      - setWeight: 25      # Step 2: 25% traffic
      - pause:
          duration: 5m
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 50      # Step 3: 50% traffic
      - pause:
          duration: 10m
      - setWeight: 100     # Step 4: Full rollout!
      
      # If any analysis fails → automatic rollback!
      rollbackWindow:
        revisions: 1
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2.0.0
---
# AnalysisTemplate — Define success criteria
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 60s
    successCondition: result[0] >= 0.99  # 99%+ success rate required
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status=~"2.."}[5m])) /
          sum(rate(http_requests_total[5m]))
```

---

## Real-World Example

### Amazon — Blue-Green at Planet Scale

```
Amazon deploys code every 11.7 seconds (average across all teams):

Their approach:
1. Each service has a "deployment pipeline"
2. New code goes to a staging environment (Green)
3. Automated tests run (integration, load, canary)
4. Traffic shifts gradually: 1 box → 1 AZ → all AZs → all regions
5. At any point, automatic rollback if metrics degrade

Pipeline stages:
┌─────┐   ┌──────┐   ┌────────┐   ┌───────┐   ┌───────────┐
│Build│──▶│ Test │──▶│ Canary │──▶│1 Zone │──▶│All Zones  │
│     │   │(unit)│   │ (1 box)│   │       │   │(full prod)│
└─────┘   └──────┘   └────────┘   └───────┘   └───────────┘
                         │              │            │
                    If errors ↑    If errors ↑   If errors ↑
                    ROLLBACK       ROLLBACK       ROLLBACK
```

### Facebook — Canary at 3 Billion Users

```
Facebook's deployment process:

1. Code lands in main branch
2. Deployed to internal employees first (dogfooding)
3. Deployed to 1% of real users (canary)
4. Metrics monitored for 24 hours
5. If healthy: expand to 10% → 50% → 100%
6. Total rollout: 3-5 days for non-urgent changes

They track:
- Error rates
- Performance (P50, P95, P99 latency)
- Engagement metrics (are users interacting less?)
- Crash rates (mobile apps)

If ANY metric regresses by >0.1% → automatic rollback
```

---

## Common Mistakes / Pitfalls

### 1. Not Running Smoke Tests Before Switching
❌ **Mistake**: Switch to Green immediately after deploy — Green is broken but you didn't test it.
✅ **Fix**: Always run health checks + smoke tests on the new environment BEFORE routing traffic.

### 2. Forgetting About Database Compatibility
❌ **Mistake**: Green requires a DB column that doesn't exist yet → Green crashes on switch.
✅ **Fix**: Use "expand and contract" migrations. DB changes must be backward-compatible.

### 3. Canary Without Proper Monitoring
❌ **Mistake**: Deploy canary but don't compare metrics → canary has 10x errors and nobody notices for hours.
✅ **Fix**: Set up automated metric comparison (v1 error rate vs v2 error rate) with automatic rollback.

### 4. Blue-Green with Shared State
❌ **Mistake**: Blue and Green share an in-memory cache → switching causes cache invalidation → thundering herd on DB.
✅ **Fix**: Warm up Green's cache before switching, or use external shared cache (Redis).

### 5. Not Testing Rollback
❌ **Mistake**: "We can rollback!" — but never actually tested it. On failure day, rollback script has a bug.
✅ **Fix**: Test rollback as part of every deployment. Switch to Green, then roll back to Blue, then switch again.

---

## When to Use / When NOT to Use

### Blue-Green

| ✅ Use When | ❌ Avoid When |
|---|---|
| Need instant rollback capability | Budget is tight (need 2x infrastructure) |
| Database migrations are involved | Application state is hard to maintain across versions |
| Compliance requires pre-production testing | Sessions/connections must persist across switch |
| Deployments are infrequent but critical | Long-running background jobs that can't be interrupted |

### Canary

| ✅ Use When | ❌ Avoid When |
|---|---|
| High-traffic services where bugs affect millions | Changes need immediate 100% rollout (security patches) |
| Feature changes with uncertain impact | Not enough traffic to get statistical significance |
| Need gradual confidence building | Database schema changes (can't serve 2 schemas) |
| Want automated progressive delivery | Team lacks monitoring infrastructure |

---

## Decision Flowchart

```
┌─────────────────────────────────────────────────────────┐
│         WHICH DEPLOYMENT STRATEGY?                       │
│                                                         │
│  Is the change risky/uncertain?                         │
│  │                                                     │
│  ├── YES: Do you have enough traffic for canary?       │
│  │   ├── YES → CANARY (gradual rollout, 5%→25%→100%)  │
│  │   └── NO → BLUE-GREEN (test then switch)           │
│  │                                                     │
│  └── NO (routine, well-tested):                        │
│      ├── Need instant rollback? → BLUE-GREEN           │
│      └── Simple update? → ROLLING DEPLOYMENT (Ch 5.6)  │
│                                                         │
│  Database schema change? → BLUE-GREEN + expand/contract │
│  Security hotfix? → ROLLING (immediate, all instances)  │
└─────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

- **Blue-Green** maintains two identical environments — deploy to the idle one, test it, then switch all traffic instantly. Rollback = switch back.
- **Canary** sends a small percentage (1-5%) of real traffic to the new version first, monitors metrics, and gradually increases if healthy.
- **Zero downtime** is the goal — users should never know you deployed new code.
- **Automated rollback** is critical — if error rates or latency increase beyond thresholds, roll back automatically without human intervention.
- **Database migrations must be backward-compatible** — both old and new code must work with the same database schema during the transition.
- **Monitor canary vs stable side-by-side** — compare error rates, latency, and business metrics between the two versions.
- **Amazon deploys every 11.7 seconds** using these techniques — it's not just for big companies, it's standard practice.

---

## What's Next?

Blue-Green and Canary are specific strategies, but what about updating 50 instances one at a time? And how do you control WHO sees new features without redeploying? That's **Chapter 5.6: Rolling Deployments & Feature Flags** — the most common deployment approach for large fleets of servers.
