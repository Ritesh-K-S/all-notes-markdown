# Configuration Management — Spring Cloud Config, Consul KV

> **What you'll learn**: How to externalize, manage, and dynamically update configuration across hundreds of microservices — without redeploying, without downtime, and without losing your sanity.

---

## Real-Life Analogy

Imagine you run a chain of 200 coffee shops. Each shop needs to know:
- The price of a latte ($4.50)
- Whether to offer the seasonal pumpkin drink (yes/no)
- The WiFi password
- The maximum discount allowed (15%)

**Bad approach**: Print a manual for each shop. When prices change, print 200 new manuals and hand-deliver them. (This is hardcoding config in your app.)

**Better approach**: Put everything on a **shared digital menu board** that all shops access. When you change the latte price in the central system, all 200 shops see the update within seconds. (This is centralized configuration management.)

**Best approach**: The digital menu board can even update **different shops differently** — the airport location charges $6.00, the suburban one charges $4.00. And some experimental shops get the new drink. (This is per-environment + per-instance configuration with feature flags.)

---

## The Problem: Why Configuration Management Matters

In a microservices world, you might have:
- 50 services × 4 environments (dev, staging, prod-us, prod-eu) = **200 config combinations**
- Each service needs: database URLs, API keys, timeouts, feature flags, retry counts...
- Changes must happen **without redeployment**
- Different environments need different values
- Secrets must be handled securely

```
┌────────────────────────────────────────────────────────────────┐
│           The Configuration Nightmare (Without Management)      │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  payment-service/                                                │
│    application-dev.yml     ← Different DB for dev               │
│    application-staging.yml ← Different API keys                 │
│    application-prod.yml    ← Production secrets                 │
│                                                                  │
│  order-service/                                                  │
│    application-dev.yml     ← Duplicated config!                 │
│    application-staging.yml ← Easy to forget one                 │
│    application-prod.yml    ← Secrets in git?!                   │
│                                                                  │
│  user-service/                                                   │
│    ...50 more like this                                          │
│                                                                  │
│  Problems:                                                       │
│  ❌ Config scattered across repos                                │
│  ❌ Changing a shared value = updating 50 repos                  │
│  ❌ No audit trail (who changed what?)                           │
│  ❌ Secrets committed to version control                         │
│  ❌ Need to redeploy to change a timeout value                   │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

---

## The Solution: Externalized Configuration

```
┌─────────────────────────────────────────────────────────────────┐
│              Centralized Configuration Architecture              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│                    ┌──────────────────────┐                       │
│                    │   Config Server       │                       │
│                    │  (Spring Cloud Config │                       │
│                    │   / Consul KV         │                       │
│                    │   / AWS AppConfig)    │                       │
│                    └──────────┬───────────┘                       │
│                               │                                   │
│              ┌────────────────┼────────────────┐                  │
│              │                │                │                  │
│              ▼                ▼                ▼                  │
│     ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│     │ Payment Svc  │  │  Order Svc   │  │  User Svc    │        │
│     │              │  │              │  │              │        │
│     │ db.url=...   │  │ db.url=...   │  │ db.url=...   │        │
│     │ timeout=5s   │  │ timeout=10s  │  │ timeout=3s   │        │
│     │ retry=3      │  │ retry=5      │  │ retry=3      │        │
│     └──────────────┘  └──────────────┘  └──────────────┘        │
│                                                                   │
│  Benefits:                                                        │
│  ✅ Single source of truth                                        │
│  ✅ Change config without redeployment                            │
│  ✅ Environment-specific values                                   │
│  ✅ Audit trail (who changed what, when)                          │
│  ✅ Secrets never in source code                                  │
│  ✅ Dynamic refresh (change takes effect in seconds)              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Configuration Hierarchy

Most systems follow a **layered override** model:

```
Priority (highest wins):
═══════════════════════

┌─────────────────────────────────────┐  ← Highest Priority
│  Environment Variables               │     (runtime override)
├─────────────────────────────────────┤
│  Instance-Specific Config            │     (this specific pod)
├─────────────────────────────────────┤
│  Profile/Environment Config          │     (prod, staging, dev)
├─────────────────────────────────────┤
│  Service-Specific Config             │     (payment-service)
├─────────────────────────────────────┤
│  Shared/Common Config                │     (all services)
├─────────────────────────────────────┤
│  Default Values (in code)            │     (hardcoded defaults)
└─────────────────────────────────────┘  ← Lowest Priority

Example resolution for "db.pool.size" in payment-service (prod):

1. Check env var: DB_POOL_SIZE → not set
2. Check instance config → not set  
3. Check profile: payment-service-prod.yml → db.pool.size=50 ✓ FOUND!
4. (Would check payment-service.yml → db.pool.size=20)
5. (Would check application.yml → db.pool.size=10)
```

---

## How It Works Internally

### Pattern 1: Pull-Based (Service Fetches Config)

```
┌──────────────┐                     ┌──────────────┐
│   Service    │──── GET /config ───▶│ Config Server│
│              │◀── JSON response ───│              │
│              │                     │              │
│  Polls every │                     │  Backed by:  │
│  30 seconds  │                     │  • Git repo  │
│  (or on      │                     │  • Database  │
│   demand)    │                     │  • Consul KV │
└──────────────┘                     └──────────────┘
```

**Used by:** Spring Cloud Config Server

### Pattern 2: Push-Based (Config Pushes to Services)

```
┌──────────────┐    ┌──────────────┐     ┌──────────┐
│ Config Store │───▶│ Message Bus  │────▶│ Service  │
│              │    │ (Kafka/      │     │ (watches │
│ Change event │    │  RabbitMQ)   │     │  for     │
│              │    │              │     │  events) │
└──────────────┘    └──────────────┘     └──────────┘
```

**Used by:** Consul Watch, etcd Watch, AWS AppConfig

### Pattern 3: Sidecar / Agent-Based

```
┌───────────────────────────────────────┐
│              Pod / VM                   │
│                                         │
│  ┌─────────────┐    ┌──────────────┐  │
│  │  Service    │◄───│  Config      │  │
│  │  (reads     │    │  Agent       │  │
│  │   local     │    │  (sidecar)   │  │
│  │   file)     │    │              │  │
│  └─────────────┘    └──────┬───────┘  │
│                             │           │
└─────────────────────────────┼───────────┘
                              │
                              ▼
                     ┌──────────────┐
                     │ Config Server│
                     └──────────────┘
```

**Used by:** Consul Agent, Vault Agent, Envoy xDS

---

## Deep Dive: Spring Cloud Config Server

Spring Cloud Config is the most popular configuration management solution in the Java ecosystem.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              Spring Cloud Config Architecture                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────┐         ┌────────────────────┐            │
│  │   Git Repository │         │  Config Server     │            │
│  │                  │◄────────│  (Spring Boot App) │            │
│  │  /payment-       │ clone   │                    │            │
│  │   service.yml    │         │  Endpoints:        │            │
│  │  /payment-       │         │  /{app}/{profile}  │            │
│  │   service-       │         │  /{app}/{profile}/ │            │
│  │   prod.yml       │         │   {label}          │            │
│  │  /order-         │         └─────────┬──────────┘            │
│  │   service.yml    │                   │                        │
│  └──────────────────┘                   │                        │
│                              ┌──────────┼──────────┐             │
│                              │          │          │             │
│                              ▼          ▼          ▼             │
│                     ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│                     │ Payment  │ │  Order   │ │  User    │      │
│                     │ Service  │ │  Service │ │  Service │      │
│                     │          │ │          │ │          │      │
│                     │@Value    │ │@Value    │ │@Value    │      │
│                     │@Refresh  │ │@Refresh  │ │@Refresh  │      │
│                     └──────────┘ └──────────┘ └──────────┘      │
│                                                                   │
│  Refresh Flow (without restart):                                  │
│  1. Push change to Git                                            │
│  2. POST /actuator/bus-refresh                                    │
│  3. Spring Cloud Bus sends event to all services                  │
│  4. Services re-fetch config from Config Server                   │
│  5. @RefreshScope beans are recreated with new values             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### File Naming Convention

```
Git repository structure:
═════════════════════════

config-repo/
├── application.yml              ← Shared by ALL services
├── application-prod.yml         ← Shared, production profile
├── payment-service.yml          ← Payment service defaults
├── payment-service-prod.yml     ← Payment service + production
├── payment-service-dev.yml      ← Payment service + development
├── order-service.yml            ← Order service defaults
└── order-service-prod.yml       ← Order service + production

Resolution order for payment-service in prod:
  1. payment-service-prod.yml    (most specific)
  2. payment-service.yml         (service defaults)
  3. application-prod.yml        (shared prod)
  4. application.yml             (shared defaults)
```

---

## Deep Dive: Consul KV for Configuration

Consul's built-in Key-Value store is widely used for configuration:

```
┌────────────────────────────────────────────────────────────────┐
│                  Consul KV Configuration                         │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Key Structure:                                                  │
│                                                                  │
│  config/                                                         │
│  ├── global/                      ← All services                │
│  │   ├── db.host = "db.prod.internal"                           │
│  │   ├── db.port = "5432"                                       │
│  │   └── log.level = "INFO"                                     │
│  ├── payment-service/             ← Service-specific            │
│  │   ├── stripe.timeout = "5000"                                │
│  │   ├── retry.max = "3"                                        │
│  │   └── pool.size = "50"                                       │
│  └── payment-service/prod/        ← Service + environment       │
│      ├── stripe.api.url = "https://api.stripe.com"              │
│      └── pool.size = "100"        ← Overrides parent            │
│                                                                  │
│  Watch mechanism:                                                │
│  • Services watch their config prefix                            │
│  • On any change → callback fires                               │
│  • Application reloads affected config                           │
│  • No restart needed!                                            │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python: Configuration Management with Consul KV

```python
import consul
import json
import threading
import os

class ConfigManager:
    """Centralized config management using Consul KV store."""
    
    def __init__(self, service_name, environment="prod",
                 consul_host="localhost", consul_port=8500):
        self.consul = consul.Consul(host=consul_host, port=consul_port)
        self.service_name = service_name
        self.environment = environment
        self._config = {}
        self._watchers = []
        
        # Load config on initialization
        self._load_config()
    
    def _load_config(self):
        """Load config with proper override hierarchy."""
        config = {}
        
        # 1. Load global defaults
        config.update(self._get_prefix("config/global/"))
        
        # 2. Load service-specific defaults
        config.update(self._get_prefix(f"config/{self.service_name}/"))
        
        # 3. Load environment-specific overrides (highest priority)
        config.update(self._get_prefix(
            f"config/{self.service_name}/{self.environment}/"
        ))
        
        # 4. Environment variables override everything
        for key in config:
            env_key = key.upper().replace(".", "_")
            if env_key in os.environ:
                config[key] = os.environ[env_key]
        
        self._config = config
        print(f"Loaded {len(config)} config values")
    
    def _get_prefix(self, prefix):
        """Get all KV pairs under a prefix."""
        _, data = self.consul.kv.get(prefix, recurse=True)
        result = {}
        if data:
            for item in data:
                key = item["Key"].replace(prefix, "")
                if key:  # Skip the prefix itself
                    value = item["Value"].decode() if item["Value"] else ""
                    result[key] = value
        return result
    
    def get(self, key, default=None):
        """Get a config value."""
        return self._config.get(key, default)
    
    def get_int(self, key, default=0):
        """Get a config value as integer."""
        return int(self._config.get(key, default))
    
    def get_bool(self, key, default=False):
        """Get a config value as boolean."""
        val = self._config.get(key, str(default)).lower()
        return val in ("true", "1", "yes")
    
    def watch_for_changes(self, callback):
        """Watch Consul KV for config changes (blocking query)."""
        prefix = f"config/{self.service_name}/"
        
        def _watch_loop():
            index = None
            while True:
                try:
                    # Blocking query — returns when data changes
                    index, data = self.consul.kv.get(
                        prefix, recurse=True, index=index, wait="5m"
                    )
                    # Reload config
                    old_config = self._config.copy()
                    self._load_config()
                    
                    # Notify about changes
                    for key in self._config:
                        if self._config[key] != old_config.get(key):
                            callback(key, old_config.get(key), 
                                    self._config[key])
                except Exception as e:
                    print(f"Config watch error: {e}")
                    import time
                    time.sleep(5)
        
        thread = threading.Thread(target=_watch_loop, daemon=True)
        thread.start()
    
    def set(self, key, value):
        """Update a config value (for admin tools)."""
        full_key = f"config/{self.service_name}/{self.environment}/{key}"
        self.consul.kv.put(full_key, str(value))


# --- Usage ---
config = ConfigManager("payment-service", environment="prod")

# Read config values
db_host = config.get("db.host", "localhost")
timeout = config.get_int("stripe.timeout", 5000)
debug = config.get_bool("debug.enabled", False)

# Watch for changes
def on_config_change(key, old_value, new_value):
    print(f"Config changed: {key} = {old_value} → {new_value}")
    # React to specific changes
    if key == "log.level":
        logging.getLogger().setLevel(new_value)

config.watch_for_changes(on_config_change)
```

### Java: Spring Cloud Config Client

```java
// --- Config Server (separate Spring Boot app) ---
// application.yml for the Config Server:
// server:
//   port: 8888
// spring:
//   cloud:
//     config:
//       server:
//         git:
//           uri: https://github.com/company/config-repo
//           default-label: main
//           search-paths: '{application}'

// Main class for Config Server
@SpringBootApplication
@EnableConfigServer  // This annotation makes it a Config Server
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}

// --- Config Client (your microservice) ---
// bootstrap.yml (loaded before application.yml):
// spring:
//   application:
//     name: payment-service
//   cloud:
//     config:
//       uri: http://config-server:8888
//       fail-fast: true
//       retry:
//         max-attempts: 5
//         initial-interval: 1000

// Using config values in your service
@RestController
@RefreshScope  // Beans in this scope are recreated on /actuator/refresh
public class PaymentController {
    
    // Injected from config server (payment-service.yml)
    @Value("${stripe.timeout:5000}")
    private int stripeTimeout;
    
    @Value("${payment.max.retries:3}")
    private int maxRetries;
    
    @Value("${payment.enabled:true}")
    private boolean paymentEnabled;
    
    @GetMapping("/api/v1/config")
    public Map<String, Object> getConfig() {
        // Useful for debugging — shows current config
        return Map.of(
            "stripeTimeout", stripeTimeout,
            "maxRetries", maxRetries,
            "paymentEnabled", paymentEnabled
        );
    }
    
    @PostMapping("/api/v1/charge")
    public ResponseEntity<?> charge(@RequestBody ChargeRequest request) {
        if (!paymentEnabled) {
            return ResponseEntity.status(503)
                .body("Payments temporarily disabled");
        }
        // Use stripeTimeout, maxRetries from config...
        return ResponseEntity.ok(processPayment(request));
    }
}

// --- Dynamic refresh with Spring Cloud Bus ---
// When config changes in Git:
// 1. Webhook calls: POST /actuator/bus-refresh
// 2. Message sent to all services via RabbitMQ/Kafka
// 3. All @RefreshScope beans recreate with new values
// 4. No restart needed!

@Configuration
public class DynamicConfig {
    
    @Bean
    @RefreshScope  // This bean is recreated when config refreshes
    public StripeClient stripeClient(
            @Value("${stripe.api.key}") String apiKey,
            @Value("${stripe.timeout:5000}") int timeout) {
        return StripeClient.builder()
            .apiKey(apiKey)
            .timeout(Duration.ofMillis(timeout))
            .build();
    }
}
```

### Java: Consul KV Config with Spring Cloud Consul

```java
// bootstrap.yml
// spring:
//   application:
//     name: payment-service
//   cloud:
//     consul:
//       host: consul.service.internal
//       port: 8500
//       config:
//         enabled: true
//         format: YAML           # Store YAML in Consul KV
//         prefix: config         # KV prefix
//         default-context: application  # shared config
//         profile-separator: ':'
//         watch:
//           enabled: true        # Auto-detect changes
//           delay: 1000          # Check every 1s

// Consul KV structure:
// config/application/data     → shared YAML config
// config/payment-service/data → service-specific YAML
// config/payment-service:prod/data → service + profile

// Your YAML stored in Consul KV at config/payment-service/data:
// stripe:
//   timeout: 5000
//   max-retries: 3
// payment:
//   enabled: true
//   max-amount: 10000

@ConfigurationProperties(prefix = "stripe")
@RefreshScope
public class StripeConfig {
    private int timeout = 5000;
    private int maxRetries = 3;
    
    // Getters and setters...
    // When Consul KV changes, this bean auto-refreshes!
}
```

---

## Infrastructure Examples

### Kubernetes ConfigMaps and Secrets

Kubernetes has built-in configuration management:

```yaml
# ConfigMap — non-sensitive configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-service-config
  namespace: production
data:
  application.yml: |
    stripe:
      timeout: 5000
      max-retries: 3
    payment:
      enabled: true
      max-amount: 10000
    logging:
      level: INFO

---
# Secret — sensitive configuration (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: payment-service-secrets
  namespace: production
type: Opaque
data:
  stripe-api-key: c2tfdGVzdF9hYmMxMjM=  # base64 encoded
  db-password: cGFzc3dvcmQxMjM=

---
# Deployment using ConfigMap and Secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
        - name: payment-service
          image: payment-service:2.3.1
          # Mount as environment variables
          envFrom:
            - configMapRef:
                name: payment-service-config
            - secretRef:
                name: payment-service-secrets
          # Or mount as files
          volumeMounts:
            - name: config-volume
              mountPath: /app/config
              readOnly: true
      volumes:
        - name: config-volume
          configMap:
            name: payment-service-config
```

### AWS AppConfig (Managed Service)

```
┌──────────────────────────────────────────────────────────────┐
│                  AWS AppConfig Architecture                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────┐                                           │
│  │ AppConfig      │  Stores configurations:                   │
│  │ (AWS Service)  │  • Feature flags                          │
│  │                │  • Operational config                     │
│  │  Applications: │  • Allow lists                            │
│  │  ├─ payment    │                                           │
│  │  ├─ order      │  Deployment strategies:                   │
│  │  └─ user       │  • AllAtOnce                              │
│  │                │  • Linear (10% every 10 min)             │
│  │  Environments: │  • Canary (10%, wait, then 100%)         │
│  │  ├─ dev        │                                           │
│  │  ├─ staging    │  Validators:                              │
│  │  └─ prod       │  • JSON Schema                           │
│  └────────┬───────┘  • Lambda function                       │
│           │                                                    │
│           ▼                                                    │
│  ┌────────────────┐     ┌────────────────┐                   │
│  │ AppConfig      │     │ Your Service   │                   │
│  │ Agent          │────▶│ (reads local   │                   │
│  │ (localhost:    │     │  port 2772)    │                   │
│  │  2772)         │     │                │                   │
│  └────────────────┘     └────────────────┘                   │
│                                                                │
│  Agent caches config locally, polls AppConfig every N seconds  │
│  Your app just reads from localhost — fast + resilient         │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

---

## Configuration Change Patterns

### Pattern 1: Restart on Change

```
Config changes → New deployment with new config → Old pods terminate

Pros: Simple, guaranteed consistency
Cons: Downtime risk, slow, wasteful
```

### Pattern 2: Signal-Based Reload

```
Config changes → Signal (SIGHUP / API call) → App reloads config

Pros: No restart, fast
Cons: App must handle reload logic, potential inconsistency window
```

### Pattern 3: Watch + Hot Reload

```
Config changes → Watch triggers → App atomically swaps config

Pros: Real-time, no downtime, no restart
Cons: Complex to implement correctly, thread safety issues
```

### Pattern 4: Gradual Rollout

```
Config changes → Deploy to 10% of instances → Monitor → Roll to 100%

Pros: Safe, reversible, catches bad config
Cons: Inconsistency during rollout (some instances have old config)
```

---

## Real-World Examples

### Netflix — Archaius

- **Custom-built** dynamic configuration library
- **Layered**: Defaults → Environment → Dynamic (highest priority)
- **Backed by**: A poll-based system that checks for changes every 30-60 seconds
- **Used for**: Timeouts, circuit breaker thresholds, feature flags, A/B test parameters
- **Scale**: Manages config for 600+ microservices

### Uber — Config System

- **Challenge**: 4000+ microservices across multiple regions
- **Solution**: Custom config service with:
  - Git as source of truth
  - Push-based updates (< 10 second propagation)
  - Rollback in seconds
  - Per-city overrides (different config for NYC vs SF)

### Kubernetes ConfigMaps at Google

- **How Google runs**: Config stored in their internal equivalent of ConfigMaps
- **Canary configs**: New config values are tested on a small percentage of pods first
- **Binary configs**: Configuration is compiled into the binary for performance

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | How to Fix |
|---------|-------------|------------|
| **Secrets in config files/Git** | Security breach waiting to happen | Use Vault, AWS Secrets Manager, or K8s Secrets |
| **No default values** | Service crashes if config is missing | Always code defaults: `@Value("${timeout:5000}")` |
| **No validation** | Bad config takes down services | Validate on load: check types, ranges, required fields |
| **Big-bang config rollout** | Bad config affects all instances at once | Use gradual rollout (canary config deployment) |
| **Not testing config changes** | Typo in prod config = outage | Test in staging first, use validators |
| **Tight coupling to config source** | Can't switch from Consul to etcd | Abstract config access behind an interface |
| **No audit trail** | "Who changed this and when??" | Use Git-backed config, or enable audit logs |
| **Ignoring config drift** | Instances have different configs | Periodically verify all instances have same config |

---

## When to Use / When NOT to Use

### Use Centralized Config Management When:
- ✅ You have **5+ microservices**
- ✅ You deploy to **multiple environments** (dev/staging/prod)
- ✅ You need to change config **without redeployment**
- ✅ You need an **audit trail** of configuration changes
- ✅ Multiple teams manage their own service configs
- ✅ You need **dynamic tuning** (timeouts, pool sizes) in production

### Don't Need It When:
- ❌ You have a **single monolith** (just use env vars)
- ❌ Config **never changes** after deployment
- ❌ You have **1-2 services** (simple env vars or files suffice)
- ❌ You're in **early development** (premature optimization)

### Which Tool to Choose:

| Tool | Best For |
|------|----------|
| **Spring Cloud Config** | Java/Spring ecosystem, Git-backed config |
| **Consul KV** | Multi-language, need both discovery + config |
| **etcd** | Already using Kubernetes, need strong consistency |
| **AWS AppConfig** | AWS-native, want managed service + safe deployment |
| **K8s ConfigMaps** | Simple key-value, Kubernetes-native |
| **HashiCorp Vault** | Secrets specifically (combine with above for non-secrets) |

---

## Key Takeaways

1. **Externalize ALL configuration** — never hardcode environment-specific values in your code. Use a central config store.

2. **Layer your config** — defaults → service-specific → environment-specific → instance-specific. Each layer overrides the one below.

3. **Dynamic refresh is critical** — the ability to change config without restarting services is what separates toy systems from production systems.

4. **Separate secrets from config** — use a dedicated secrets manager (Vault, AWS Secrets Manager) for sensitive values. Don't mix them with regular config.

5. **Validate config on load** — a typo in a config value shouldn't crash your service. Validate types, ranges, and required fields.

6. **Gradual rollout for config changes** — treat config changes like code deployments. Roll out to a subset first, then expand.

7. **Audit everything** — you MUST know who changed what config value, when, and why. Git-backed config or audit logs are essential.

---

## What's Next?

Configuration management gives you the ability to change values dynamically. But one of the most powerful uses of dynamic configuration is **feature toggles** — turning features on/off without deploying new code. In the next chapter, [04-feature-toggles.md](./04-feature-toggles.md), we'll explore **Feature Toggles & Dynamic Configuration** — how to safely roll out features, run A/B tests, and implement kill switches.
