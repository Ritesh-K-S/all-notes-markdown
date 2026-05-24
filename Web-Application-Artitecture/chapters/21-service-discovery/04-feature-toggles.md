# Feature Toggles & Dynamic Configuration

> **What you'll learn**: How to turn features on and off without deploying new code, safely roll out changes to a subset of users, run A/B tests, implement kill switches, and manage the lifecycle of feature flags from creation to retirement.

---

## Real-Life Analogy

Imagine you're a restaurant owner testing a new dish — "Spicy Mango Tacos."

**Without feature toggles (old way):** You print a new menu with the dish, give it to ALL customers at once. If people hate it, you have to reprint all menus and remove it. Expensive, slow, risky.

**With feature toggles (smart way):**
1. You **add the dish to the menu but cover it with a sticky note** — it's in the kitchen, but no one can order it yet.
2. You **remove the sticky note for regulars only** — your loyal customers try it first and give feedback.
3. If they love it → remove sticky notes from ALL menus (gradual rollout).
4. If something goes wrong (supply issue, allergic reactions) → slap the sticky note back on instantly. No menu reprinting needed.

Feature toggles are those sticky notes — **the code is deployed but invisible until you flip a switch**.

---

## What Are Feature Toggles?

A **feature toggle** (also called feature flag, feature switch, or feature gate) is a technique that allows you to enable or disable a feature **at runtime** without deploying new code.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Feature Toggle in Action                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Code (always deployed):                                          │
│                                                                   │
│  if (featureFlags.isEnabled("new-checkout-flow")) {              │
│      return newCheckoutPage();     ← New experience              │
│  } else {                                                        │
│      return oldCheckoutPage();     ← Current experience          │
│  }                                                               │
│                                                                   │
│  Toggle State (changed without deployment):                       │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  new-checkout-flow:                                      │    │
│  │    enabled: true                                         │    │
│  │    rollout: 25%        ← Only 25% of users see it       │    │
│  │    targets:                                              │    │
│  │      - country: US     ← Only US users                  │    │
│  │      - plan: premium   ← Only premium users             │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Types of Feature Toggles

Not all feature toggles are the same. They serve different purposes and have different lifecycles:

```
┌────────────────────────────────────────────────────────────────────┐
│                    Types of Feature Toggles                          │
├─────────────────┬──────────────┬──────────────┬───────────────────┤
│     Type        │  Lifespan    │  Who Controls│  Example          │
├─────────────────┼──────────────┼──────────────┼───────────────────┤
│ Release Toggle  │ Days-Weeks   │ Dev/PM       │ New checkout flow │
│                 │              │              │ (hide until ready) │
├─────────────────┼──────────────┼──────────────┼───────────────────┤
│ Experiment      │ Weeks-Months │ Data/Product │ A/B test: blue vs │
│ Toggle          │              │              │ green button       │
├─────────────────┼──────────────┼──────────────┼───────────────────┤
│ Ops Toggle      │ Indefinite   │ Ops/SRE      │ Kill switch for   │
│ (Kill Switch)   │              │              │ recommendations   │
├─────────────────┼──────────────┼──────────────┼───────────────────┤
│ Permission      │ Indefinite   │ Business     │ Premium feature   │
│ Toggle          │              │              │ (paid users only) │
└─────────────────┴──────────────┴──────────────┴───────────────────┘
```

### 1. Release Toggles

**Purpose**: Decouple deployment from release. Code ships to production but stays hidden.

```
Sprint 1: Merge "new search" code (toggle OFF)
Sprint 2: Merge more search code (toggle OFF)
Sprint 3: QA tests internally (toggle ON for internal users)
Sprint 4: Release to 10% of users (toggle ON, 10%)
Sprint 5: Release to everyone (toggle ON, 100%)
Sprint 6: Remove toggle + old code (cleanup!)
```

### 2. Experiment Toggles (A/B Tests)

**Purpose**: Test variations to make data-driven decisions.

```
┌────────────────────────────────────────────────────────┐
│           A/B Test: Checkout Button Color               │
├────────────────────────────────────────────────────────┤
│                                                          │
│  User arrives at checkout page                           │
│         │                                                │
│         ▼                                                │
│  ┌─────────────┐                                        │
│  │ Feature Flag │                                       │
│  │ Engine       │                                       │
│  └──────┬──────┘                                        │
│         │                                                │
│    ┌────┴────┐                                          │
│    ▼         ▼                                          │
│ Group A    Group B                                      │
│ (50%)      (50%)                                        │
│ [Blue]     [Green]                                      │
│ Button     Button                                       │
│    │         │                                          │
│    ▼         ▼                                          │
│ Track      Track                                        │
│ conversion conversion                                   │
│ rate       rate                                         │
│                                                          │
│ Result: Green → 12% more conversions → Ship green!      │
│                                                          │
└────────────────────────────────────────────────────────┘
```

### 3. Ops Toggles (Kill Switches)

**Purpose**: Quickly disable non-critical features during incidents.

```
Normal Operation:
  recommendations-engine: ON
  personalized-feed: ON  
  analytics-tracking: ON

During High Load / Incident:
  recommendations-engine: OFF  ← Saves 40% CPU
  personalized-feed: OFF       ← Reduces DB queries
  analytics-tracking: OFF      ← Frees network bandwidth

Result: Core functionality (checkout, payments) stays fast
```

### 4. Permission Toggles

**Purpose**: Gate features by user attributes (plan, role, geography).

```
Feature: "Advanced Analytics Dashboard"
  Rules:
    - plan == "enterprise" → ON
    - plan == "pro" AND country IN ["US", "UK"] → ON
    - role == "admin" → ON (for internal testing)
    - everyone else → OFF
```

---

## How Feature Toggles Work Internally

### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│              Feature Toggle System Architecture                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌────────────────┐                                               │
│  │  Admin UI /    │  Create/modify flags, set rules, targets     │
│  │  Dashboard     │                                               │
│  └───────┬────────┘                                               │
│          │                                                         │
│          ▼                                                         │
│  ┌────────────────┐                                               │
│  │  Flag Service  │  Central storage of all flag definitions      │
│  │  (LaunchDarkly │  • Flag name + description                    │
│  │   / Unleash /  │  • Targeting rules                           │
│  │   Flagsmith)   │  • Rollout percentage                         │
│  └───────┬────────┘  • Variants (A/B test groups)                │
│          │                                                         │
│          │ SDK fetches flags                                       │
│          │ (streaming or polling)                                  │
│          ▼                                                         │
│  ┌────────────────────────────────────────────────────────┐       │
│  │              Your Application                           │       │
│  │                                                          │       │
│  │  ┌──────────────┐     ┌──────────────┐                 │       │
│  │  │  Flag SDK    │     │ Local Cache  │                 │       │
│  │  │  (client)    │────▶│ (in-memory)  │                 │       │
│  │  └──────────────┘     └──────┬───────┘                 │       │
│  │                               │                          │       │
│  │         flag.isEnabled("x", user) → true/false          │       │
│  │                               │                          │       │
│  │  ┌──────────────────────────┼──────────────────────┐   │       │
│  │  │                          ▼                       │   │       │
│  │  │  if (enabled) {                                  │   │       │
│  │  │      // new code path                            │   │       │
│  │  │  } else {                                        │   │       │
│  │  │      // old code path                            │   │       │
│  │  │  }                                               │   │       │
│  │  └──────────────────────────────────────────────────┘   │       │
│  │                                                          │       │
│  └────────────────────────────────────────────────────────┘       │
│                                                                    │
│  Flag Evaluation (happens locally, no network call):               │
│  1. Get flag definition from local cache                           │
│  2. Evaluate targeting rules against user context                  │
│  3. If targeted → return flag variation                           │
│  4. If percentage rollout → hash(user_id) % 100 < percentage?    │
│  5. Return result (true/false or variant string)                  │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Percentage Rollout — How It Works

The magic of "enable for 25% of users" is done via **deterministic hashing**:

```
┌────────────────────────────────────────────────────────────────┐
│           Percentage Rollout Algorithm                           │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Goal: Enable "new-checkout" for 25% of users                   │
│                                                                  │
│  Algorithm:                                                      │
│    bucket = hash(flag_name + user_id) % 100                     │
│    enabled = bucket < rollout_percentage                         │
│                                                                  │
│  Example:                                                        │
│    hash("new-checkout" + "user_123") % 100 = 17  → 17 < 25 ✓  │
│    hash("new-checkout" + "user_456") % 100 = 73  → 73 < 25 ✗  │
│    hash("new-checkout" + "user_789") % 100 = 4   → 4 < 25  ✓  │
│                                                                  │
│  Key properties:                                                 │
│  • DETERMINISTIC: Same user always gets same result             │
│  • CONSISTENT: Increasing from 25% to 50% keeps original 25%   │
│  • NO STATE: No need to store per-user flag assignments         │
│  • UNIFORM: Good hash function → even distribution              │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

### Targeting Rules Engine

```
┌────────────────────────────────────────────────────────────────┐
│                  Rule Evaluation Order                           │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Flag: "new-recommendation-engine"                              │
│                                                                  │
│  Rules (evaluated top to bottom, first match wins):             │
│                                                                  │
│  1. IF user.email ENDS_WITH "@company.com"                      │
│     THEN → ENABLED (always on for internal users)               │
│                                                                  │
│  2. IF user.id IN [specific_beta_user_ids]                      │
│     THEN → ENABLED (specific beta testers)                      │
│                                                                  │
│  3. IF user.country == "IN" AND user.plan == "premium"          │
│     THEN → ENABLED (premium India users = early adopters)       │
│                                                                  │
│  4. IF user.created_after > "2024-01-01"                        │
│     THEN → 50% rollout (new users get it 50% of the time)      │
│                                                                  │
│  5. DEFAULT → DISABLED                                          │
│                                                                  │
│  Evaluation for user {id: "u_789", email: "test@gmail.com",    │
│                        country: "IN", plan: "premium"}:          │
│    Rule 1: email doesn't end with @company.com → skip          │
│    Rule 2: not in beta list → skip                              │
│    Rule 3: country=IN AND plan=premium → MATCH! → ENABLED      │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: Feature Toggle Implementation

```python
import hashlib
import json
import threading
import time
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional

@dataclass
class TargetingRule:
    """A single targeting rule for a feature flag."""
    attribute: str          # e.g., "country", "plan", "email"
    operator: str           # e.g., "equals", "in", "contains", "percentage"
    value: Any              # The value to compare against
    variation: bool = True  # What to return if rule matches

@dataclass
class FeatureFlag:
    """Definition of a feature flag."""
    name: str
    enabled: bool = False
    rollout_percentage: int = 100  # 0-100
    rules: List[TargetingRule] = field(default_factory=list)
    description: str = ""

class FeatureFlagClient:
    """Client-side feature flag evaluation engine."""
    
    def __init__(self, config_source, poll_interval=30):
        self._flags: Dict[str, FeatureFlag] = {}
        self._config_source = config_source
        self._lock = threading.Lock()
        
        # Load flags initially
        self._refresh_flags()
        
        # Start background polling for updates
        self._poller = threading.Thread(
            target=self._poll_loop, 
            args=(poll_interval,), 
            daemon=True
        )
        self._poller.start()
    
    def is_enabled(self, flag_name: str, user_context: Dict = None) -> bool:
        """
        Evaluate if a feature flag is enabled for a given user.
        This runs LOCALLY (no network call) against cached flag data.
        """
        with self._lock:
            flag = self._flags.get(flag_name)
        
        if not flag:
            return False  # Unknown flag = disabled (safe default)
        
        if not flag.enabled:
            return False  # Globally disabled
        
        # Evaluate targeting rules (first match wins)
        if flag.rules and user_context:
            for rule in flag.rules:
                if self._evaluate_rule(rule, user_context):
                    return rule.variation
        
        # No rules matched → apply percentage rollout
        if flag.rollout_percentage < 100:
            return self._in_rollout(flag_name, user_context, 
                                     flag.rollout_percentage)
        
        return True  # Enabled with 100% rollout
    
    def _evaluate_rule(self, rule: TargetingRule, context: Dict) -> bool:
        """Evaluate a single targeting rule against user context."""
        user_value = context.get(rule.attribute)
        if user_value is None:
            return False
        
        if rule.operator == "equals":
            return user_value == rule.value
        elif rule.operator == "in":
            return user_value in rule.value
        elif rule.operator == "contains":
            return rule.value in str(user_value)
        elif rule.operator == "ends_with":
            return str(user_value).endswith(rule.value)
        elif rule.operator == "greater_than":
            return user_value > rule.value
        
        return False
    
    def _in_rollout(self, flag_name: str, context: Dict, 
                    percentage: int) -> bool:
        """Deterministic percentage rollout using hashing."""
        # Use user_id for consistency; fallback to session_id
        user_id = context.get("user_id", context.get("session_id", ""))
        
        # Hash flag name + user ID for deterministic bucket assignment
        hash_input = f"{flag_name}:{user_id}"
        hash_value = hashlib.sha256(hash_input.encode()).hexdigest()
        
        # Convert first 8 hex chars to int, mod 100 for bucket
        bucket = int(hash_value[:8], 16) % 100
        
        return bucket < percentage
    
    def _refresh_flags(self):
        """Fetch latest flag definitions from the config source."""
        try:
            flags_data = self._config_source.get_all_flags()
            with self._lock:
                self._flags = flags_data
        except Exception as e:
            print(f"Failed to refresh flags: {e}")
            # Keep using cached flags — don't break the app!
    
    def _poll_loop(self, interval):
        """Background thread to poll for flag updates."""
        while True:
            time.sleep(interval)
            self._refresh_flags()


# --- Usage ---
# (Assuming config_source is set up to read from your flag service)

flags = FeatureFlagClient(config_source=consul_flag_source)

# Simple boolean check
if flags.is_enabled("new-checkout-flow"):
    render_new_checkout()
else:
    render_old_checkout()

# With user targeting
user = {
    "user_id": "u_12345",
    "email": "alice@company.com",
    "country": "US",
    "plan": "premium",
    "created_at": "2024-03-15"
}

if flags.is_enabled("ai-recommendations", user):
    show_ai_recommendations(user)
else:
    show_basic_recommendations(user)

# Kill switch pattern
if not flags.is_enabled("recommendations-engine", user):
    # Feature killed — return cached/default recommendations
    return get_cached_recommendations(user)
```

### Java: Feature Toggles with Unleash SDK

```java
import io.getunleash.Unleash;
import io.getunleash.DefaultUnleash;
import io.getunleash.UnleashContext;
import io.getunleash.util.UnleashConfig;
import io.getunleash.strategy.Strategy;
import java.util.Map;
import java.util.Optional;

public class FeatureToggleService {
    
    private final Unleash unleash;
    
    public FeatureToggleService() {
        // Configure the Unleash SDK
        UnleashConfig config = UnleashConfig.builder()
            .appName("payment-service")
            .instanceId("payment-1")
            .unleashAPI("http://unleash-server:4242/api")
            .apiKey("your-api-key")
            .fetchTogglesInterval(10)   // Poll every 10 seconds
            .sendMetricsInterval(60)    // Report usage every 60s
            .build();
        
        this.unleash = new DefaultUnleash(config);
    }
    
    /**
     * Check if a feature is enabled for a specific user.
     */
    public boolean isEnabled(String featureName, UserContext user) {
        // Build Unleash context from user data
        UnleashContext context = UnleashContext.builder()
            .userId(user.getId())
            .sessionId(user.getSessionId())
            .remoteAddress(user.getIpAddress())
            .addProperty("country", user.getCountry())
            .addProperty("plan", user.getPlan())
            .addProperty("email", user.getEmail())
            .build();
        
        return unleash.isEnabled(featureName, context);
    }
    
    /**
     * Get a specific variant (for A/B tests).
     */
    public String getVariant(String featureName, UserContext user) {
        UnleashContext context = UnleashContext.builder()
            .userId(user.getId())
            .build();
        
        return unleash.getVariant(featureName, context)
            .map(v -> v.getName())
            .orElse("control");  // Default to control group
    }
}

// --- Usage in a Controller ---
@RestController
public class CheckoutController {
    
    private final FeatureToggleService features;
    
    @PostMapping("/api/v1/checkout")
    public ResponseEntity<?> checkout(
            @RequestBody CheckoutRequest request,
            @AuthenticationPrincipal UserContext user) {
        
        // Release toggle: new checkout flow
        if (features.isEnabled("new-checkout-flow", user)) {
            return newCheckoutFlow(request, user);
        }
        return legacyCheckoutFlow(request, user);
    }
    
    @GetMapping("/api/v1/recommendations")
    public ResponseEntity<?> recommendations(
            @AuthenticationPrincipal UserContext user) {
        
        // Ops toggle (kill switch): disable during high load
        if (!features.isEnabled("recommendations-engine", user)) {
            return ResponseEntity.ok(getCachedRecommendations(user));
        }
        
        // Experiment toggle: algorithm A vs B
        String variant = features.getVariant("recommendation-algo", user);
        
        switch (variant) {
            case "collaborative":
                return ResponseEntity.ok(collaborativeFiltering(user));
            case "content-based":
                return ResponseEntity.ok(contentBasedFiltering(user));
            default:  // "control"
                return ResponseEntity.ok(currentAlgorithm(user));
        }
    }
}

// --- Custom Strategy (Advanced) ---
// Create a custom targeting strategy for Unleash
public class GeoTargetingStrategy implements Strategy {
    
    @Override
    public String getName() {
        return "geoTargeting";
    }
    
    @Override
    public boolean isEnabled(Map<String, String> parameters, 
                             UnleashContext context) {
        String allowedCountries = parameters.get("countries");
        String userCountry = context.getProperties().get("country");
        
        if (allowedCountries == null || userCountry == null) {
            return false;
        }
        
        return List.of(allowedCountries.split(","))
            .contains(userCountry);
    }
}
```

### Python: Kill Switch Pattern for Graceful Degradation

```python
from functools import wraps
import logging

logger = logging.getLogger(__name__)

def feature_gate(flag_name, fallback=None):
    """
    Decorator that gates a function behind a feature flag.
    If flag is disabled, returns fallback value or calls fallback function.
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Get user context from kwargs or first arg
            user_ctx = kwargs.get('user_context', {})
            
            if feature_flags.is_enabled(flag_name, user_ctx):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    logger.error(f"Feature {flag_name} failed: {e}")
                    # Auto-degrade on error
                    if callable(fallback):
                        return fallback(*args, **kwargs)
                    return fallback
            else:
                # Feature is disabled
                logger.info(f"Feature {flag_name} is disabled")
                if callable(fallback):
                    return fallback(*args, **kwargs)
                return fallback
        return wrapper
    return decorator


# --- Usage ---

def get_basic_recommendations(user_id, **kwargs):
    """Fallback: return cached/static recommendations."""
    return {"recommendations": get_from_cache(user_id)}

@feature_gate("ai-recommendations", fallback=get_basic_recommendations)
def get_personalized_recommendations(user_id, user_context=None):
    """Primary: expensive ML-based recommendations."""
    return ml_engine.predict(user_id)

# Called like normal — flag evaluation is transparent:
recs = get_personalized_recommendations("u_123", user_context=user)
```

---

## Infrastructure: Feature Flag Tools

### Self-Hosted: Unleash (Docker Compose)

```yaml
version: '3.8'
services:
  # Unleash Server (flag management UI + API)
  unleash:
    image: unleashorg/unleash-server:latest
    ports:
      - "4242:4242"
    environment:
      DATABASE_URL: postgres://unleash:password@postgres:5432/unleash
      DATABASE_SSL: "false"
    depends_on:
      postgres:
        condition: service_healthy

  # PostgreSQL for Unleash storage
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: unleash
      POSTGRES_USER: unleash
      POSTGRES_PASSWORD: password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U unleash"]
      interval: 5s
    volumes:
      - unleash-data:/var/lib/postgresql/data

volumes:
  unleash-data:
```

### Consul KV as Feature Flag Store

```python
# Store flags in Consul KV
import consul

c = consul.Consul()

# Create a feature flag
flag_definition = {
    "name": "new-checkout-flow",
    "enabled": True,
    "rollout_percentage": 25,
    "rules": [
        {
            "attribute": "email",
            "operator": "ends_with",
            "value": "@company.com",
            "variation": True  # Always on for internal
        }
    ],
    "description": "New React-based checkout page",
    "owner": "checkout-team",
    "created_at": "2024-01-15"
}

c.kv.put(
    "feature-flags/new-checkout-flow",
    json.dumps(flag_definition)
)

# Watch for changes (used by SDK)
index = None
while True:
    index, data = c.kv.get("feature-flags/", recurse=True, index=index)
    # Process updated flags...
```

### AWS AppConfig for Feature Flags

```json
// AWS AppConfig Feature Flag configuration profile
{
  "version": "1",
  "flags": {
    "new-checkout-flow": {
      "name": "New Checkout Flow",
      "description": "React-based checkout experience",
      "attributes": {
        "rollout_percentage": {
          "constraints": {
            "type": "number",
            "minimum": 0,
            "maximum": 100
          }
        }
      }
    },
    "ai-recommendations": {
      "name": "AI Recommendations",
      "description": "ML-powered product recommendations"
    }
  },
  "values": {
    "new-checkout-flow": {
      "enabled": true,
      "rollout_percentage": 25
    },
    "ai-recommendations": {
      "enabled": true
    }
  }
}
```

---

## Feature Toggle Lifecycle

Every feature toggle should follow a clear lifecycle. **Stale toggles are tech debt!**

```
┌────────────────────────────────────────────────────────────────┐
│              Feature Toggle Lifecycle                            │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. CREATE                                                       │
│     │  • Define flag name, owner, description                   │
│     │  • Set initial state (OFF)                                │
│     │  • Set expected expiry date                               │
│     ▼                                                            │
│  2. DEVELOP                                                      │
│     │  • Add toggle check in code                               │
│     │  • Deploy (flag still OFF in prod)                        │
│     │  • Test with flag ON in dev/staging                       │
│     ▼                                                            │
│  3. ROLL OUT                                                     │
│     │  • Enable for internal users (dogfooding)                 │
│     │  • Enable for 5% → 25% → 50% → 100%                     │
│     │  • Monitor metrics at each stage                          │
│     ▼                                                            │
│  4. FULL RELEASE                                                 │
│     │  • Flag at 100% for sufficient time                       │
│     │  • Metrics confirm success                                │
│     ▼                                                            │
│  5. CLEANUP ← Most teams forget this!                           │
│     │  • Remove toggle from code                                │
│     │  • Remove old code path                                   │
│     │  • Delete flag from system                                │
│     │  • Deploy cleaned-up code                                 │
│     ▼                                                            │
│  DONE                                                            │
│                                                                  │
│  ⚠️  WARNING: Flags that live forever become "toggle debt"       │
│  Set alerts for flags older than their expected expiry!          │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

---

## Real-World Examples

### Facebook — Gatekeeper

- **Scale**: Thousands of feature flags controlling billions of user experiences
- **System**: Custom-built "Gatekeeper" system
- **Key feature**: Every code change ships behind a flag. Engineers never push directly to users.
- **Rollout**: New features start at 0%, slowly ramped to 100% over days/weeks
- **Targeting**: Can target by country, OS, app version, employee status, specific user IDs
- **Speed**: Flag changes propagate globally in < 10 seconds

### Netflix — Feature Flags for Everything

- **Approach**: Every service has feature flags for critical code paths
- **Kill switches**: Non-essential features (recommendations, personalization, previews) can be disabled instantly during incidents
- **Example**: During AWS outages, Netflix disables non-critical features to keep streaming working
- **Scale**: Evaluated millions of times per second across all services

### GitHub — Feature Flags (Scientist + Flipper)

- **Scientist**: A/B testing library that runs both old and new code paths simultaneously, compares results, and reports mismatches
- **Flipper**: Ruby gem for feature flags with per-actor, group, and percentage targeting
- **Practice**: All major features ship behind flags. GitHub.com deploys 100+ times/day safely
- **Staff shipping**: New features enabled for GitHub employees first → then beta users → then everyone

### Uber — UCDP (Uber Client Dynamic Platform)

- **Challenge**: Manage features across mobile apps (Android, iOS) where "deployment" = app store release
- **Solution**: Server-side feature flags control which features mobile apps show
- **Kill switches**: Can disable a buggy feature in the mobile app without requiring users to update
- **Geo-targeting**: Different features in different cities (regulations vary)

---

## Comparison: Feature Flag Tools

| Tool | Type | Targeting | A/B Testing | Price | Best For |
|------|------|-----------|-------------|-------|----------|
| **LaunchDarkly** | SaaS | Advanced | Yes | $$$ | Enterprise, any language |
| **Unleash** | Self-hosted/Cloud | Good | Basic | Free/$ | Teams wanting control |
| **Flagsmith** | Self-hosted/Cloud | Good | Yes | Free/$ | Open-source friendly |
| **Split.io** | SaaS | Advanced | Yes | $$ | Data-driven teams |
| **AWS AppConfig** | Managed | Basic | No | $ | AWS-native apps |
| **ConfigCat** | SaaS | Good | No | $ | Small-medium teams |
| **Consul KV** | Self-hosted | DIY | DIY | Free | Already using Consul |
| **Custom** | DIY | Custom | Custom | Dev time | Specific requirements |

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | How to Fix |
|---------|-------------|------------|
| **Never cleaning up flags** | Hundreds of stale toggles = unmaintainable code | Set expiry dates, alert on old flags, require cleanup tickets |
| **Testing only with flag ON** | Old code path rots, breaks when flag OFF | Test BOTH paths in CI/CD |
| **No fallback when flag service is down** | App crashes if can't evaluate flags | SDK caches flags locally; use safe defaults |
| **Too many nested flags** | `if A && B && !C` = impossible to reason about | Keep flags independent; avoid interactions |
| **Flags in hot code paths without caching** | Performance degradation from network calls | Evaluate flags from in-memory cache only |
| **No ownership for flags** | Nobody knows what a flag does or if it's safe to remove | Require owner, description, and expiry on creation |
| **Using flags for permanent permissions** | Feature flag systems aren't access control systems | Use proper RBAC for permissions (Chapter 14) |
| **Percentage rollout without stickiness** | User sees feature, refreshes, it's gone | Use deterministic hashing on user_id |

---

## When to Use / When NOT to Use

### Use Feature Toggles When:
- ✅ You want to **deploy frequently** but release features gradually
- ✅ You need **kill switches** for graceful degradation during incidents
- ✅ You're running **A/B tests** to make data-driven decisions
- ✅ You want to **decouple deployment from release** (trunk-based development)
- ✅ You need **canary releases** for risky features
- ✅ Mobile apps where you **can't force updates**

### Don't Use Feature Toggles When:
- ❌ The change is **trivial or low-risk** (just deploy it)
- ❌ You'll **never turn it off** (it's not a toggle, it's just code)
- ❌ You don't have a plan to **clean up the flag** (it becomes debt)
- ❌ The feature is a **one-time migration** (use a deployment strategy instead)
- ❌ You're using them as a **substitute for proper testing**

---

## Key Takeaways

1. **Feature toggles decouple deployment from release** — ship code anytime, release to users when ready. This is the foundation of continuous delivery.

2. **Four types serve different purposes**: Release toggles (short-lived), experiment toggles (A/B tests), ops toggles (kill switches), and permission toggles (access control).

3. **Percentage rollouts use deterministic hashing** — `hash(flag + user_id) % 100 < percentage`. Same user always gets the same experience, no state storage needed.

4. **Always evaluate flags locally** — fetch flag definitions from the server, cache them, and evaluate in-memory. Never make a network call per flag check.

5. **Kill switches are non-negotiable** — every non-critical feature in production should be behind a kill switch for graceful degradation during incidents.

6. **Clean up stale flags religiously** — feature flags that live forever become unmaintainable "toggle spaghetti." Set expiry dates and enforce cleanup.

7. **Test both code paths** — if you only test with the flag ON, the old path rots and will break when you need it most (during an incident when you flip the flag OFF).

---

## What's Next?

This concludes **Part 21: Service Discovery, Configuration & Coordination**. You now understand how services find each other (discovery), where they store their settings (configuration), and how to control feature rollout (toggles).

Next up is **Part 22: Storage & File Systems** — where we'll go beyond databases to explore object storage (S3), distributed file systems, and how companies handle massive amounts of unstructured data like images, videos, and logs at planet scale.
