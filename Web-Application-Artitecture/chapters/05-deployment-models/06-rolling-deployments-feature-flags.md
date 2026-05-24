# Rolling Deployments & Feature Flags

> **What you'll learn**: How to update application instances one-by-one without downtime (rolling deployments), and how to separate code deployment from feature releases using feature flags — giving you control over WHO sees WHAT, WHEN.

---

## Real-Life Analogy

### Rolling Deployment: Replacing Tires While Driving

Imagine you need to change all four tires on a bus — **while it's still carrying passengers**. Sounds impossible? Not if you do it one at a time:

1. The bus has 4 wheels. You lift wheel #1, replace the tire, put it back down. The bus is still rolling on 3 wheels.
2. Repeat for wheels #2, #3, #4.
3. Passengers never stopped moving. They barely noticed.

That's a **rolling deployment** — you update your servers one at a time while the others keep serving traffic.

### Feature Flags: Light Switches in Your Code

Imagine your house has every room wired and ready, but each room has a **light switch**. Even though the wiring for the new entertainment room is done, the switch is OFF. You can turn it ON for just yourself (testing), then for your family (beta users), then flip it ON for everyone (general availability).

**Feature flags** let you deploy code that's "dark" (hidden) until you flip the switch — no redeployment needed.

---

## Core Concept Explained Step-by-Step

### Rolling Deployment

```
10 instances running v1.0. Goal: update all to v2.0 with ZERO downtime.

Strategy: Update 2 at a time (maxUnavailable=2)

Step 1: Take 2 offline, update them
┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐
│v1.0││v1.0││v1.0││v1.0││v1.0││v1.0││v1.0││v1.0││ ⬆️ ││ ⬆️ │
│ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  ││upd ││upd │
└────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘
                                                  ↑ updating to v2.0

Step 2: Those 2 are now v2.0, take next 2 offline
┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐
│v1.0││v1.0││v1.0││v1.0││v1.0││v1.0││ ⬆️ ││ ⬆️ ││v2.0││v2.0│
│ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  ││upd ││upd ││ ✓  ││ ✓  │
└────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘

Step 3-5: Continue until all updated
┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐
│v2.0││v2.0││v2.0││v2.0││v2.0││v2.0││v2.0││v2.0││v2.0││v2.0│
│ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  ││ ✓  │
└────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘

DONE! All instances on v2.0. Zero downtime throughout.
At any point during the rollout, at least 8/10 instances were serving.
```

### Rolling Deployment Parameters

```
┌─────────────────────────────────────────────────────────────────────┐
│  KEY PARAMETERS:                                                    │
│                                                                     │
│  maxUnavailable: How many instances can be DOWN at once?            │
│  ├── "25%" → max 2 out of 10 can be updating simultaneously       │
│  ├── "1"   → only 1 at a time (safest, slowest)                   │
│  └── "50%" → half the fleet at once (fast, risky)                  │
│                                                                     │
│  maxSurge: How many EXTRA instances can exist temporarily?          │
│  ├── "25%" → spin up 2 extra v2.0 instances before removing v1.0  │
│  └── "0"   → no extra capacity (must remove before adding)         │
│                                                                     │
│  COMBINATIONS:                                                      │
│                                                                     │
│  Conservative (safest):                                             │
│    maxUnavailable=0, maxSurge=1                                    │
│    → Add 1 new v2 instance, then remove 1 v1 instance              │
│    → Capacity NEVER drops below 10. Slow but safe.                 │
│                                                                     │
│  Balanced (default Kubernetes):                                     │
│    maxUnavailable=25%, maxSurge=25%                                │
│    → Update 25% at a time. Brief mixed v1/v2.                      │
│                                                                     │
│  Aggressive (fast):                                                 │
│    maxUnavailable=50%, maxSurge=0                                  │
│    → Take half offline, update, then other half.                   │
│    → Capacity drops to 50%. Only if you have excess headroom.      │
└─────────────────────────────────────────────────────────────────────┘
```

### Feature Flags

```
┌─────────────────────────────────────────────────────────────────────┐
│                     FEATURE FLAGS                                    │
│                                                                     │
│  CONCEPT: Code is deployed but FEATURES are controlled separately   │
│                                                                     │
│  Without Feature Flags:                                             │
│  ┌─────────────┐        ┌─────────────┐                           │
│  │  Deploy v1  │───────▶│  Deploy v2  │  ← Feature goes live      │
│  │ (no feature)│        │(with feature)│    AT deployment time     │
│  └─────────────┘        └─────────────┘                           │
│                                                                     │
│  With Feature Flags:                                                │
│  ┌─────────────┐        ┌─────────────┐      ┌────────────────┐  │
│  │  Deploy v1  │───────▶│  Deploy v2  │─────▶│ Enable Feature │  │
│  │ (no feature)│        │(feature OFF)│      │ (flip switch)  │  │
│  └─────────────┘        └─────────────┘      └────────────────┘  │
│                                                                     │
│  Feature is live when YOU decide, not when code is deployed!       │
│                                                                     │
│  Timeline:                                                          │
│  Mon: Code merged & deployed (flag OFF) → users see nothing        │
│  Wed: Flag ON for internal team → team tests in production         │
│  Fri: Flag ON for 10% of users → canary test                      │
│  Mon: Flag ON for 100% → general availability                     │
│  +2 weeks: Remove flag from code (cleanup)                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Types of Feature Flags

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  TYPE 1: Release Flag (short-lived)                                 │
│  Purpose: Control rollout of new feature                           │
│  Lifetime: Days to weeks                                           │
│  Example: "new_checkout_flow" → ON for 10% users                  │
│                                                                     │
│  TYPE 2: Ops Flag (kill switch)                                     │
│  Purpose: Disable feature if it causes problems                    │
│  Lifetime: Permanent (always in code)                              │
│  Example: "enable_recommendations" → OFF during peak load          │
│                                                                     │
│  TYPE 3: Experiment Flag (A/B test)                                 │
│  Purpose: Test two variations against each other                   │
│  Lifetime: Weeks to months                                         │
│  Example: "button_color" → "blue" for 50%, "green" for 50%       │
│                                                                     │
│  TYPE 4: Permission Flag (entitlement)                              │
│  Purpose: Gate features by plan/tier                               │
│  Lifetime: Permanent                                               │
│  Example: "advanced_analytics" → ON for Enterprise plan only       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Rolling Deployment: Graceful Shutdown

When an instance is being replaced, it must **finish current requests** before shutting down:

```
GRACEFUL SHUTDOWN SEQUENCE:

┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  1. Deployment controller: "Instance 3, you're being replaced"  │
│                                                                  │
│  2. Load balancer: Stop sending NEW requests to Instance 3       │
│     (Instance 3 is now "draining")                              │
│                                                                  │
│  3. Instance 3: Finish processing in-flight requests             │
│     Request A: ████████░░ (80% done → finish it)               │
│     Request B: ██░░░░░░░░ (20% done → finish it)               │
│                                                                  │
│  4. Wait up to 30 seconds for all requests to complete           │
│     (terminationGracePeriodSeconds)                             │
│                                                                  │
│  5. Instance 3: Shuts down cleanly                              │
│                                                                  │
│  6. New Instance 3 (v2.0): Starts up                            │
│                                                                  │
│  7. Health check passes → Load balancer routes to new instance   │
│                                                                  │
│  8. Move on to Instance 4...                                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

TIMELINE:
─────────────────────────────────────────────────────────────▶ time
│ Draining │ Shutdown │ Start v2 │ Health check │ Ready!  │
│  (30s)   │  (2s)    │  (10s)   │   (15s)      │ Traffic │
```

### Feature Flag: How It Works in Code

```
REQUEST PROCESSING WITH FEATURE FLAG:

┌────────────┐      ┌────────────────────┐      ┌──────────────┐
│   Client   │─────▶│   Application      │─────▶│ Flag Service │
│  (request) │      │                    │      │ (LaunchDarkly│
└────────────┘      │  if flag("new_ui") │      │  / Unleash)  │
                    │    → show new UI    │◀─────│              │
                    │  else               │      │ Returns:     │
                    │    → show old UI    │      │ true/false   │
                    └────────────────────┘      └──────────────┘

EVALUATION CONTEXT:
┌────────────────────────────────────────────────────┐
│  Flag: "new_checkout_flow"                         │
│                                                    │
│  Rules:                                            │
│  1. If user.email ends with "@mycompany.com" → ON  │ (internal testing)
│  2. If user.id in [123, 456, 789] → ON            │ (specific beta users)
│  3. If user.country == "US" → ON for 10%          │ (gradual US rollout)
│  4. Default → OFF                                  │
│                                                    │
│  Evaluation for user #456 (beta list): ON ✓       │
│  Evaluation for random US user: 10% chance ON     │
│  Evaluation for user in India: OFF                │
└────────────────────────────────────────────────────┘
```

### Feature Flag Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  FEATURE FLAG SYSTEM                              │
│                                                                 │
│  ┌──────────────────────────────────────┐                       │
│  │      Flag Management Dashboard       │                       │
│  │   (LaunchDarkly / Unleash / Custom)  │                       │
│  │                                       │                       │
│  │  ┌─────────────────────────────────┐ │                       │
│  │  │ new_checkout:  ON (10% users)   │ │                       │
│  │  │ dark_mode:     ON (all users)   │ │                       │
│  │  │ ai_search:     OFF              │ │                       │
│  │  │ holiday_promo: ON (US only)     │ │                       │
│  │  └─────────────────────────────────┘ │                       │
│  └──────────────────┬───────────────────┘                       │
│                     │                                           │
│                     │ Push updates via SSE/WebSocket             │
│                     │ (or SDK polls every 30s)                  │
│                     ▼                                           │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐               │
│  │  App       │  │  App       │  │  App       │               │
│  │ Instance 1 │  │ Instance 2 │  │ Instance 3 │               │
│  │            │  │            │  │            │               │
│  │ SDK cache: │  │ SDK cache: │  │ SDK cache: │               │
│  │ new_checkout│  │ new_checkout│  │ new_checkout│               │
│  │  → 10%    │  │  → 10%    │  │  → 10%    │               │
│  └────────────┘  └────────────┘  └────────────┘               │
│                                                                 │
│  Flag changes propagate to ALL instances within seconds         │
│  (No redeployment needed!)                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python (Feature Flag with Unleash SDK)

```python
# app.py — Feature flags controlling feature visibility
from flask import Flask, jsonify, request
from UnleashClient import UnleashClient
import os

app = Flask(__name__)

# Initialize feature flag SDK
unleash = UnleashClient(
    url=os.environ["UNLEASH_URL"],     # e.g., "http://unleash-server:4242/api"
    app_name="my-backend",
    instance_id=os.environ.get("HOSTNAME", "local"),
)
unleash.initialize_client()

@app.route("/api/search")
def search():
    """Search endpoint with optional AI-powered search behind a flag."""
    query = request.args.get("q", "")
    user_id = get_current_user_id()
    
    # Check feature flag with user context
    context = {"userId": str(user_id), "properties": {"plan": get_user_plan(user_id)}}
    
    if unleash.is_enabled("ai_search", context):
        # New AI-powered search (only visible to flagged users)
        results = ai_search(query)
        return jsonify({"results": results, "search_type": "ai"})
    else:
        # Traditional search (everyone else)
        results = traditional_search(query)
        return jsonify({"results": results, "search_type": "standard"})

@app.route("/api/checkout", methods=["POST"])
def checkout():
    """Checkout with kill switch for peak traffic protection."""
    user_id = get_current_user_id()
    context = {"userId": str(user_id)}
    
    # Ops flag: disable recommendations during peak load
    show_recommendations = unleash.is_enabled("enable_recommendations", context)
    
    order = process_order(request.json)
    
    response = {"order_id": order.id, "status": "confirmed"}
    if show_recommendations:
        response["recommendations"] = get_recommendations(user_id)
    
    return jsonify(response)
```

### Java (Feature Flag with LaunchDarkly SDK)

```java
// FeatureFlagService.java — Centralized feature flag evaluation
@Service
public class FeatureFlagService {
    
    private final LDClient ldClient;
    
    public FeatureFlagService(@Value("${launchdarkly.sdk-key}") String sdkKey) {
        this.ldClient = new LDClient(sdkKey);
    }
    
    public boolean isEnabled(String flagKey, User user) {
        LDContext context = LDContext.builder(user.getId())
            .set("email", user.getEmail())
            .set("plan", user.getPlan())
            .set("country", user.getCountry())
            .build();
        return ldClient.boolVariation(flagKey, context, false);
    }
    
    public String getVariation(String flagKey, User user, String defaultValue) {
        LDContext context = LDContext.builder(user.getId()).build();
        return ldClient.stringVariation(flagKey, context, defaultValue);
    }
}

// SearchController.java — Using feature flags
@RestController
public class SearchController {
    
    @Autowired private FeatureFlagService flags;
    @Autowired private AISearchService aiSearch;
    @Autowired private TraditionalSearchService classicSearch;
    
    @GetMapping("/api/search")
    public ResponseEntity<SearchResults> search(
            @RequestParam String q,
            @AuthenticationPrincipal User user) {
        
        // Feature flag controls which search engine is used
        if (flags.isEnabled("ai_search", user)) {
            return ResponseEntity.ok(aiSearch.search(q));
        } else {
            return ResponseEntity.ok(classicSearch.search(q));
        }
    }
}
```

### Python (Rolling Deployment Script)

```python
# rolling_deploy.py — Manual rolling deployment to EC2 instances
import boto3
import time
import requests

ec2 = boto3.client('ec2')
elb = boto3.client('elbv2')

TARGET_GROUP_ARN = "arn:aws:elasticloadbalancing:..."
INSTANCES = ["i-001", "i-002", "i-003", "i-004", "i-005"]
BATCH_SIZE = 2  # Update 2 at a time

def rolling_deploy(new_ami: str):
    """Deploy new version 2 instances at a time."""
    
    for i in range(0, len(INSTANCES), BATCH_SIZE):
        batch = INSTANCES[i:i + BATCH_SIZE]
        print(f"\n--- Deploying batch: {batch} ---")
        
        # Step 1: Deregister from load balancer (stop traffic)
        for instance_id in batch:
            deregister_from_lb(instance_id)
        
        # Step 2: Wait for connections to drain (30s)
        print("Draining connections (30s)...")
        time.sleep(30)
        
        # Step 3: Update each instance
        for instance_id in batch:
            update_instance(instance_id, new_ami)
        
        # Step 4: Wait for instances to be healthy
        for instance_id in batch:
            wait_for_healthy(instance_id)
        
        # Step 5: Re-register with load balancer
        for instance_id in batch:
            register_to_lb(instance_id)
        
        # Step 6: Verify health through LB
        print("Verifying health through load balancer...")
        time.sleep(15)
        
        # Step 7: Check for errors before continuing
        if check_error_rate() > 0.01:  # >1% error rate
            print("⚠️  ERROR RATE TOO HIGH! Stopping deployment.")
            print(f"   Remaining instances still on old version: {INSTANCES[i+BATCH_SIZE:]}")
            return False
        
        print(f"✅ Batch {batch} deployed successfully!")
    
    print("\n✅ Rolling deployment complete! All instances updated.")
    return True
```

---

## Infrastructure Example

### Kubernetes Rolling Update

```yaml
# deployment.yaml — Kubernetes rolling update configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2    # At most 2 pods can be down during update
      maxSurge: 2          # At most 2 extra pods can exist temporarily
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      terminationGracePeriodSeconds: 30  # Allow 30s for graceful shutdown
      containers:
      - name: app
        image: myapp:v2.0.0    # ← Change this to trigger rolling update
        ports:
        - containerPort: 8080
        readinessProbe:        # Only send traffic when ready
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:         # Restart if unhealthy
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 10"]  # Allow LB to deregister
```

```
KUBERNETES ROLLING UPDATE TIMELINE:

kubectl set image deployment/backend-app app=myapp:v2.0.0

Time 0s:    10 pods (v1) running
Time 5s:    8 pods (v1) + 2 pods (v2 starting)     [maxUnavailable=2]
Time 20s:   8 pods (v1) + 2 pods (v2 ready ✓)
Time 25s:   6 pods (v1) + 4 pods (v2)              [next batch]
Time 40s:   6 pods (v1) + 4 pods (v2 ready ✓)
...
Time 100s:  0 pods (v1) + 10 pods (v2 ready ✓)    [COMPLETE]
```

### Feature Flag with Unleash (Self-Hosted)

```yaml
# docker-compose.yml — Self-hosted feature flag system
version: '3.8'

services:
  unleash:
    image: unleashorg/unleash-server:latest
    ports:
      - "4242:4242"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/unleash
      - DATABASE_SSL=false
    depends_on:
      - db
  
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: unleash
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - unleash_data:/var/lib/postgresql/data

  # Your application instances connect to Unleash
  app1:
    build: ./app
    environment:
      - UNLEASH_URL=http://unleash:4242/api
      - UNLEASH_APP_NAME=my-backend
      - UNLEASH_API_TOKEN=${UNLEASH_TOKEN}
  
  app2:
    build: ./app
    environment:
      - UNLEASH_URL=http://unleash:4242/api
      - UNLEASH_APP_NAME=my-backend
      - UNLEASH_API_TOKEN=${UNLEASH_TOKEN}

volumes:
  unleash_data:
```

### GitHub Actions: Rolling Deploy Pipeline

```yaml
# .github/workflows/rolling-deploy.yml
name: Rolling Deployment

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: |
          docker build -t myapp:${{ github.sha }} .
          docker push myregistry/myapp:${{ github.sha }}
      
      - name: Rolling deploy to Kubernetes
        run: |
          kubectl set image deployment/backend-app \
            app=myregistry/myapp:${{ github.sha }}
          
          # Wait for rollout to complete (with timeout)
          kubectl rollout status deployment/backend-app \
            --timeout=300s
      
      - name: Verify deployment health
        run: |
          # Check error rate after deployment
          ERROR_RATE=$(curl -s http://prometheus:9090/api/v1/query \
            --data-urlencode 'query=rate(http_errors_total[5m])' | jq '.data.result[0].value[1]')
          
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Error rate too high! Rolling back..."
            kubectl rollout undo deployment/backend-app
            exit 1
          fi
      
      - name: Rollback on failure
        if: failure()
        run: kubectl rollout undo deployment/backend-app
```

---

## Real-World Example

### Netflix — Rolling + Feature Flags Together

```
Netflix's Deployment Strategy:

1. Code merged → Rolling deployment to all instances
   (New code is deployed, but new FEATURES are behind flags)

2. Feature flags control visibility:
   ┌─────────────────────────────────────────────────────────┐
   │  "new_recommendation_algo":                              │
   │    - Netflix employees: ON (dogfooding)                  │
   │    - 1% of US users: ON (canary/experiment)             │
   │    - Everyone else: OFF                                  │
   │                                                         │
   │  After 1 week of healthy metrics:                       │
   │    - 10% of US users: ON                                │
   │                                                         │
   │  After 2 weeks:                                          │
   │    - 100% of all users: ON                              │
   │    - Old code path: deleted in next sprint              │
   └─────────────────────────────────────────────────────────┘

3. Kill switches for incidents:
   "If recommendation service is overloaded, 
    disable_recommendations = ON → show generic list instead"
```

### Facebook/Meta — Gatekeeper System

```
Facebook's internal feature flag system ("Gatekeeper"):

- 10,000+ active feature flags at any time
- Every engineer uses flags for every feature
- Typical lifecycle:

  Week 1: Code deployed, flag OFF
  Week 1: Flag ON for author only (self-testing)
  Week 2: Flag ON for team (5 people)
  Week 2: Flag ON for company (50,000 employees)
  Week 3: Flag ON for 1% external users
  Week 4: Flag ON for 50% external users  
  Week 5: Flag ON for 100% (launched!)
  Week 7: Flag removed from code (cleanup)

Stats:
- ~500 flags created per week
- ~400 flags cleaned up per week
- Average flag lifetime: 2-4 weeks
```

### Uber — Feature Flag for City Launches

```
Uber uses feature flags to control city-by-city launches:

"uber_pool" feature flag:
├── San Francisco: ON (launched first)
├── New York: ON (launched second)
├── London: ON
├── Mumbai: OFF (not yet launched)
├── Tokyo: OFF (regulatory approval pending)
└── Default: OFF

When Mumbai is ready:
1. Flip "uber_pool" → ON for Mumbai
2. No code change needed!
3. No deployment needed!
4. If problems arise → flip back to OFF instantly
```

---

## Common Mistakes / Pitfalls

### Rolling Deployment Pitfalls

#### 1. No Readiness Probes
❌ **Mistake**: New instance gets traffic before it's fully started → 502 errors during deploy.
✅ **Fix**: Configure readiness probes. Only route traffic after the app passes health checks.

#### 2. Breaking API Changes
❌ **Mistake**: v2 removes an API field that v1 clients expect → errors during mixed v1/v2 window.
✅ **Fix**: Maintain backward compatibility. Add new fields first, deprecate old ones later.

```
DURING ROLLING UPDATE, BOTH VERSIONS EXIST:
Client ──▶ LB ──┬──▶ v1.0 (returns {name, email})
                └──▶ v2.0 (returns {name, email, avatar})  ← ADD is safe
                                                                REMOVE is NOT safe!
```

#### 3. No Rollback Strategy
❌ **Mistake**: v2 has a subtle bug discovered after full rollout. No way to go back quickly.
✅ **Fix**: Always keep the previous image/artifact ready. `kubectl rollout undo` should work.

### Feature Flag Pitfalls

#### 4. Flag Debt (Never Cleaning Up)
❌ **Mistake**: 500 stale flags in code. Nobody knows which are still needed. Code is unreadable.
✅ **Fix**: Set expiration dates on flags. Track flag age. Remove flags within 2-4 weeks of full rollout.

```python
# BAD: Flag from 2 years ago, feature is fully launched
if feature_flag("new_checkout_2022"):  # This is ALWAYS true now!
    checkout_v2()
else:
    checkout_v1()  # DEAD CODE!

# GOOD: After full rollout, clean up
checkout_v2()  # Just call it directly. Remove the flag.
```

#### 5. Testing Only the Flag-ON Path
❌ **Mistake**: Tests only verify the feature with flag ON. Flag OFF path has a bug nobody catches.
✅ **Fix**: Test BOTH paths. Both flag=ON and flag=OFF must work correctly.

#### 6. Performance Impact of Flag Checks
❌ **Mistake**: Flag evaluated on every request by making HTTP call to flag server → adds 10ms per request.
✅ **Fix**: Use SDKs with local caching. Flag values cached in-memory, updated via streaming/polling.

---

## When to Use / When NOT to Use

### Rolling Deployment

| ✅ Use When | ❌ Avoid When |
|---|---|
| Standard, routine deployments | Database schema changes (use Blue-Green) |
| Large fleet (10-1000 instances) | All-or-nothing changes (breaking API changes) |
| Need zero downtime | Need instant rollback (Blue-Green is faster) |
| Kubernetes or ECS managed workloads | Stateful applications (need careful draining) |
| Frequent deployments (daily/weekly) | Very small fleet (2-3 instances, Blue-Green simpler) |

### Feature Flags

| ✅ Use When | ❌ Avoid When |
|---|---|
| Gradual feature rollout | Simple bug fixes (just deploy them) |
| A/B testing / experiments | Features that can't be partially enabled |
| Kill switches for incidents | Team is too small to manage flags |
| Decoupling deploy from release | Adding excessive complexity to simple apps |
| Dark launches (testing in prod) | Infrastructure changes (can't flag a DB migration) |

---

## Combined Strategy: The Modern Approach

```
THE BEST PRACTICE: Rolling Deploy + Feature Flags

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Code merged (Mon AM)                                              │
│       │                                                             │
│       ▼                                                             │
│  CI/CD Pipeline → Build → Test → Rolling Deploy (all instances)    │
│       │                    (new code deployed, features OFF)        │
│       ▼                                                             │
│  Feature Flag Dashboard:                                            │
│       │                                                             │
│       ├─ Mon PM: Flag ON for team (internal testing)               │
│       ├─ Tue: Flag ON for 5% of users (canary)                     │
│       ├─ Wed: Flag ON for 25%                                      │
│       ├─ Thu: Flag ON for 100% (general availability! 🎉)          │
│       │                                                             │
│       ├─ IF errors spike at any %: Flag OFF (instant rollback!)    │
│       │   (no redeployment needed!)                                │
│       │                                                             │
│       └─ +2 weeks: Remove flag from code (cleanup commit)          │
│                                                                     │
│  RESULT: Deployment risk ≈ ZERO                                    │
│  - Code is always deployed (rolling)                               │
│  - Features are controlled independently (flags)                   │
│  - Rollback is instant (flip flag OFF)                             │
│  - No downtime ever                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

- **Rolling deployment** updates instances one-by-one (or in batches), maintaining capacity throughout — the default strategy for Kubernetes and most orchestrators.
- **Graceful shutdown** is essential — instances must finish in-flight requests before stopping (terminationGracePeriodSeconds).
- **Feature flags separate deployment from release** — code can be deployed "dark" and enabled later without redeploying.
- **Feature flags enable instant rollback** — if something breaks, flip the flag OFF in seconds. No redeployment.
- **Combine both** for maximum safety: rolling deploy gets code to servers, feature flags control who sees what.
- **Clean up stale flags** — flag debt is real. Remove flags within weeks of full rollout or they become unmaintainable.
- **Netflix, Facebook, and Uber all use this combination** — it's how they ship features daily to billions of users with near-zero risk.

---

## What's Next?

You now know HOW to deploy code safely. But what about the infrastructure itself — the servers, networks, databases, load balancers? How do you create and manage all of that reliably and repeatably? That's **Chapter 5.7: Infrastructure as Code (Terraform, CloudFormation)** — where your entire infrastructure is defined in version-controlled files.
