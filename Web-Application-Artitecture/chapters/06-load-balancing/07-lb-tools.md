# Load Balancing Tools — Nginx, HAProxy, AWS ALB/NLB, Envoy

> **What you'll learn**: A hands-on comparison of the four most important load balancing tools in production today — their architectures, configuration styles, performance characteristics, and the specific scenarios where each shines. By the end, you'll know exactly which tool to reach for in any situation.

---

## Real-Life Analogy — Four Types of Traffic Controllers

| Tool | Analogy |
|------|---------|
| **Nginx** | A Swiss Army knife traffic cop — directs traffic, serves files, handles SSL, and more. Great at everything, master of web traffic |
| **HAProxy** | A Formula 1 pit crew — does ONE thing (proxy traffic) with extreme speed and precision. Pure load balancer, nothing else |
| **AWS ALB/NLB** | A self-driving car system — managed by someone else, zero maintenance, scales automatically, but you can't pop the hood |
| **Envoy** | A smart traffic drone swarm — modern, programmable, designed for microservices. Talks to every other drone for coordination |

---

## Core Concept Explained Step-by-Step

### Tool 1: Nginx

```
┌─────────────────────────────────────────────────────────────────┐
│                          NGINX                                   │
│                                                                 │
│  TYPE: Web server + Reverse proxy + Load balancer               │
│  LAYER: Layer 7 (HTTP/HTTPS) + basic Layer 4 (TCP/UDP streams) │
│  LICENSE: Open source (Nginx OSS) + Commercial (Nginx Plus)     │
│  WRITTEN IN: C                                                  │
│  CREATED: 2004, by Igor Sysoev                                  │
│                                                                 │
│  WHAT IT DOES:                                                  │
│  ├── Serves static files (HTML, CSS, JS, images)               │
│  ├── Reverse proxy for backend servers                          │
│  ├── Load balancer (HTTP and TCP/UDP)                           │
│  ├── SSL/TLS termination                                        │
│  ├── Caching proxy                                              │
│  ├── Rate limiting                                              │
│  ├── gzip compression                                           │
│  └── WebSocket proxying                                         │
│                                                                 │
│  ARCHITECTURE: Event-driven, non-blocking I/O                   │
│  ├── Master process (manages workers)                           │
│  ├── Worker processes (handle connections)                      │
│  └── Each worker handles thousands of connections concurrently  │
│                                                                 │
│  PERFORMANCE:                                                   │
│  ├── Handles 10,000+ concurrent connections per worker          │
│  ├── Very low memory footprint (~2.5MB per 10K connections)     │
│  └── Can serve 100K+ requests/second on modest hardware         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Nginx Configuration — Full Load Balancer Setup

```nginx
# /etc/nginx/nginx.conf — Production load balancer configuration

worker_processes auto;  # One worker per CPU core
events {
    worker_connections 4096;  # Max connections per worker
    use epoll;               # High-performance event model (Linux)
}

http {
    # Logging
    log_format lb_format '$remote_addr → $upstream_addr [$status] ${request_time}s';
    access_log /var/log/nginx/lb_access.log lb_format;

    # --- Backend server pools ---
    upstream api_servers {
        least_conn;                        # Algorithm: least connections
        server 10.0.1.1:8080 weight=3;    # 3x traffic (powerful server)
        server 10.0.1.2:8080 weight=2;    # 2x traffic
        server 10.0.1.3:8080 weight=1;    # 1x traffic
        server 10.0.1.4:8080 backup;      # Only used if others are down
        
        keepalive 32;  # Keep 32 connections alive to backends
    }
    
    upstream static_servers {
        server 10.0.2.1:80;
        server 10.0.2.2:80;
    }

    # --- SSL termination + routing ---
    server {
        listen 443 ssl http2;
        server_name api.myapp.com;
        
        ssl_certificate     /etc/ssl/certs/myapp.pem;
        ssl_certificate_key /etc/ssl/private/myapp.key;
        ssl_protocols       TLSv1.2 TLSv1.3;
        
        # API routes → api_servers pool
        location /api/ {
            proxy_pass http://api_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_read_timeout 30s;
            
            # Retry on failure
            proxy_next_upstream error timeout http_502 http_503;
            proxy_next_upstream_tries 2;
        }
        
        # Static files → static_servers pool
        location /static/ {
            proxy_pass http://static_servers;
            expires 30d;
            add_header Cache-Control "public, immutable";
        }
        
        # Rate limiting
        limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
        location /api/expensive {
            limit_req zone=api_limit burst=20;
            proxy_pass http://api_servers;
        }
    }
    
    # HTTP → HTTPS redirect
    server {
        listen 80;
        return 301 https://$host$request_uri;
    }
}
```

---

### Tool 2: HAProxy

```
┌─────────────────────────────────────────────────────────────────┐
│                          HAPROXY                                  │
│                                                                 │
│  TYPE: Pure load balancer / proxy (NOT a web server)            │
│  LAYER: Layer 4 (TCP) + Layer 7 (HTTP)                          │
│  LICENSE: Open source (Community) + Commercial (Enterprise)     │
│  WRITTEN IN: C                                                  │
│  CREATED: 2000, by Willy Tarreau                                │
│                                                                 │
│  WHAT IT DOES:                                                  │
│  ├── TCP and HTTP load balancing                                │
│  ├── SSL/TLS termination and pass-through                       │
│  ├── Advanced health checking                                   │
│  ├── Connection queuing and rate limiting                       │
│  ├── Circuit breaking                                           │
│  ├── Stick tables (advanced session tracking)                   │
│  └── Detailed real-time statistics dashboard                    │
│                                                                 │
│  DOES NOT DO:                                                   │
│  ├── Serve static files (it's NOT a web server!)               │
│  ├── Application hosting                                        │
│  └── Caching                                                    │
│                                                                 │
│  ARCHITECTURE: Multi-threaded event-driven                      │
│  ├── Single process, multiple threads                           │
│  ├── Zero-copy forwarding where possible                        │
│  └── Optimized for maximum throughput                           │
│                                                                 │
│  PERFORMANCE:                                                   │
│  ├── Handles 2 million+ concurrent connections                  │
│  ├── Up to 100 Gbps throughput                                  │
│  ├── Sub-millisecond latency addition                           │
│  └── Industry standard for high-performance load balancing      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### HAProxy Configuration — Full Production Setup

```haproxy
# /etc/haproxy/haproxy.cfg — Production configuration

global
    maxconn 100000
    nbthread 4                    # Use 4 CPU threads
    ssl-default-bind-ciphers ECDHE-ECDSA-AES256-GCM-SHA384
    stats socket /var/run/haproxy.sock mode 660

defaults
    mode http
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    option httplog
    option dontlognull
    option http-server-close     # Enable HTTP keep-alive
    retries 3

# --- Frontend (what clients connect to) ---
frontend https_front
    bind *:443 ssl crt /etc/ssl/myapp.pem
    bind *:80
    
    # Redirect HTTP → HTTPS
    http-request redirect scheme https unless { ssl_fc }
    
    # Route based on URL path (L7 intelligence)
    acl is_api path_beg /api
    acl is_static path_beg /static
    acl is_websocket hdr(Upgrade) -i WebSocket
    
    use_backend api_backend if is_api
    use_backend static_backend if is_static
    use_backend ws_backend if is_websocket
    default_backend web_backend

# --- Backend pools ---
backend api_backend
    balance leastconn
    option httpchk GET /health HTTP/1.1\r\nHost:\ api.myapp.com
    
    # Health check: every 5s, 3 failures = down, 2 successes = up
    default-server inter 5s fall 3 rise 2
    
    server api1 10.0.1.1:8080 check weight 100
    server api2 10.0.1.2:8080 check weight 100
    server api3 10.0.1.3:8080 check weight 50   # Smaller instance
    server api4 10.0.1.4:8080 check backup      # Backup only

backend web_backend
    balance roundrobin
    option httpchk GET /health
    cookie SERVERID insert indirect nocache
    
    server web1 10.0.2.1:8080 check cookie s1
    server web2 10.0.2.2:8080 check cookie s2

backend static_backend
    balance uri                  # Same URL → same server (cache-friendly)
    server cdn1 10.0.3.1:80 check
    server cdn2 10.0.3.2:80 check

backend ws_backend
    balance source               # Same client → same server (WebSocket)
    timeout server 3600s         # Long timeout for WebSocket
    server ws1 10.0.4.1:8080 check
    server ws2 10.0.4.2:8080 check

# --- Stats dashboard ---
listen stats
    bind *:9000
    stats enable
    stats uri /stats
    stats refresh 5s
    stats auth admin:secure_password_here
```

---

### Tool 3: AWS ALB and NLB

```
┌─────────────────────────────────────────────────────────────────┐
│                   AWS LOAD BALANCERS                              │
│                                                                 │
│  ┌───────────────────────────┐  ┌────────────────────────────┐ │
│  │  ALB (Application LB)     │  │  NLB (Network LB)          │ │
│  │                           │  │                            │ │
│  │  Layer: 7 (HTTP/HTTPS)    │  │  Layer: 4 (TCP/UDP/TLS)   │ │
│  │  Use: Web apps, APIs,     │  │  Use: Databases, gaming,   │ │
│  │       microservices       │  │       TCP services, gRPC   │ │
│  │                           │  │                            │ │
│  │  Features:                │  │  Features:                 │ │
│  │  ├── Path-based routing   │  │  ├── Ultra-low latency     │ │
│  │  ├── Host-based routing   │  │  ├── Static IP addresses  │ │
│  │  ├── WebSocket support    │  │  ├── Millions of req/s    │ │
│  │  ├── HTTP/2              │  │  ├── TLS passthrough       │ │
│  │  ├── Lambda targets       │  │  ├── Preserve source IP   │ │
│  │  ├── Authentication       │  │  └── Cross-zone LB        │ │
│  │  ├── WAF integration      │  │                            │ │
│  │  └── Sticky sessions      │  │  Latency: ~100μs          │ │
│  │                           │  │  Throughput: millions pps  │ │
│  │  Latency: 1-5ms           │  │                            │ │
│  │  Cost: ~$0.0225/hr + LCU  │  │  Cost: ~$0.006/hr + LCU   │ │
│  └───────────────────────────┘  └────────────────────────────┘ │
│                                                                 │
│  MANAGED: AWS handles scaling, patching, redundancy             │
│  You configure WHAT, AWS handles HOW                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### AWS ALB — Terraform Configuration

```hcl
# main.tf — AWS ALB with path-based routing

resource "aws_lb" "app" {
  name               = "my-app-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids

  enable_deletion_protection = true
}

# Target groups (backend pools)
resource "aws_lb_target_group" "api" {
  name     = "api-targets"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    path                = "/health"
    interval            = 15
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
    matcher             = "200"
  }

  stickiness {
    type            = "lb_cookie"
    cookie_duration = 3600
    enabled         = false  # Disabled by default
  }
}

resource "aws_lb_target_group" "web" {
  name     = "web-targets"
  port     = 3000
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    path    = "/"
    matcher = "200"
  }
}

# HTTPS listener with routing rules
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# Rule: /api/* → API target group
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  condition {
    path_pattern { values = ["/api/*"] }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}
```

---

### Tool 4: Envoy Proxy

```
┌─────────────────────────────────────────────────────────────────┐
│                          ENVOY                                    │
│                                                                 │
│  TYPE: Modern L4/L7 proxy for cloud-native architectures        │
│  LAYER: Layer 4 + Layer 7                                       │
│  LICENSE: Open source (Apache 2.0, CNCF graduated project)      │
│  WRITTEN IN: C++                                                │
│  CREATED: 2016, by Lyft                                         │
│                                                                 │
│  WHAT MAKES IT DIFFERENT:                                       │
│  ├── Designed for microservices (not monoliths)                 │
│  ├── Dynamic configuration via API (no config file reloads!)    │
│  ├── Built-in observability (metrics, tracing, logging)         │
│  ├── Foundation for service meshes (Istio uses Envoy)           │
│  ├── gRPC-native (first-class support)                          │
│  ├── Hot restart (zero-downtime config updates)                 │
│  └── Extensible via WASM filters                                │
│                                                                 │
│  KEY CONCEPT: xDS APIs                                          │
│  ├── CDS: Cluster Discovery Service (find backends)            │
│  ├── EDS: Endpoint Discovery Service (server IPs)              │
│  ├── LDS: Listener Discovery Service (ports to listen on)      │
│  └── RDS: Route Discovery Service (routing rules)              │
│  → Control plane pushes config to Envoy dynamically!           │
│                                                                 │
│  PERFORMANCE:                                                   │
│  ├── Very high throughput (comparable to Nginx)                 │
│  ├── ~1ms latency for L7 proxying                              │
│  └── Designed for sidecar deployment (one per service)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Envoy Configuration

```yaml
# envoy.yaml — Envoy proxy as load balancer
static_resources:
  listeners:
  - name: http_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 8080
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              # /api/* → api_cluster
              - match: { prefix: "/api" }
                route:
                  cluster: api_cluster
                  timeout: 30s
                  retry_policy:
                    retry_on: "5xx,reset,connect-failure"
                    num_retries: 2
              # Everything else → web_cluster
              - match: { prefix: "/" }
                route:
                  cluster: web_cluster
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: api_cluster
    connect_timeout: 5s
    type: STRICT_DNS
    lb_policy: LEAST_REQUEST       # Least connections equivalent
    load_assignment:
      cluster_name: api_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: 10.0.1.1, port_value: 8080 }
        - endpoint:
            address:
              socket_address: { address: 10.0.1.2, port_value: 8080 }
    health_checks:
    - timeout: 3s
      interval: 10s
      unhealthy_threshold: 3
      healthy_threshold: 2
      http_health_check:
        path: "/health"

  - name: web_cluster
    connect_timeout: 5s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: web_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: 10.0.2.1, port_value: 3000 }
        - endpoint:
            address:
              socket_address: { address: 10.0.2.2, port_value: 3000 }

# Built-in admin interface for metrics/health
admin:
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }
```

---

## Head-to-Head Comparison

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    DETAILED COMPARISON TABLE                              │
├─────────────────┬──────────┬──────────┬──────────────┬─────────────────┤
│   Feature       │  Nginx   │  HAProxy │  AWS ALB/NLB │  Envoy          │
├─────────────────┼──────────┼──────────┼──────────────┼─────────────────┤
│ Layer           │ L7 (+L4) │ L4 + L7  │ ALB:L7 NLB:L4│ L4 + L7         │
│ Managed         │ No       │ No       │ Yes          │ No              │
│ Serves files    │ Yes      │ No       │ No           │ No              │
│ gRPC support    │ Yes      │ Yes      │ ALB: Yes     │ Native          │
│ WebSocket       │ Yes      │ Yes      │ ALB: Yes     │ Yes             │
│ HTTP/2          │ Yes      │ Yes      │ ALB: Yes     │ Yes             │
│ HTTP/3 (QUIC)   │ Yes      │ No       │ No           │ Yes             │
│ Dynamic config  │ Reload   │ Reload   │ API          │ xDS API (live!) │
│ Service mesh    │ No       │ No       │ No           │ Yes (Istio)     │
│ Observability   │ Basic    │ Good     │ CloudWatch   │ Excellent       │
│ Sticky sessions │ Plus only│ Yes      │ Yes          │ Yes             │
│ Rate limiting   │ Yes      │ Basic    │ WAF          │ Yes             │
│ Cost            │ Free/$$  │ Free     │ Pay per use  │ Free            │
│ Learning curve  │ Low      │ Medium   │ Low          │ High            │
│ Config format   │ Custom   │ Custom   │ Console/TF   │ YAML/xDS       │
│ Sidecar mode    │ Awkward  │ No       │ N/A          │ Designed for it │
└─────────────────┴──────────┴──────────┴──────────────┴─────────────────┘
```

---

## How It Works Internally

### Architecture Comparison

```
NGINX ARCHITECTURE:
┌───────────────────────────────────────────┐
│  Master Process                           │
│  ├── Worker 1 (event loop, thousands of connections)
│  ├── Worker 2 (event loop, thousands of connections)
│  ├── Worker 3 (event loop, thousands of connections)
│  └── Worker 4 (event loop, thousands of connections)
│                                           │
│  Config reload: new workers start, old ones drain
│  (brief gap during reload — not truly zero-downtime)
└───────────────────────────────────────────┘

HAPROXY ARCHITECTURE:
┌───────────────────────────────────────────┐
│  Single Process, Multiple Threads         │
│  ├── Thread 1 ──┐                        │
│  ├── Thread 2 ──┤── Shared connection    │
│  ├── Thread 3 ──┤── tables and state     │
│  └── Thread 4 ──┘                        │
│                                           │
│  Config reload: new process starts, old   │
│  transfers connections via Unix socket    │
│  (seamless — active connections survive!)  │
└───────────────────────────────────────────┘

ENVOY ARCHITECTURE:
┌───────────────────────────────────────────┐
│  Main Thread (admin, xDS, health checks)  │
│  ├── Worker Thread 1 (listener + conn)   │
│  ├── Worker Thread 2 (listener + conn)   │
│  ├── Worker Thread 3 (listener + conn)   │
│  └── Worker Thread 4 (listener + conn)   │
│                                           │
│  Config update: xDS API pushes changes   │
│  instantly to all workers — NO reload!   │
│  (truly zero-downtime configuration)     │
└───────────────────────────────────────────┘

AWS ALB ARCHITECTURE:
┌───────────────────────────────────────────┐
│  ┌─────────────────────────────────────┐  │
│  │  AWS Managed Infrastructure         │  │
│  │  (you never see this)               │  │
│  │                                     │  │
│  │  • Multiple AZs automatically       │  │
│  │  • Auto-scales with traffic         │  │
│  │  • AWS handles patches/upgrades     │  │
│  │  • Built-in DDoS protection         │  │
│  │                                     │  │
│  │  YOU CONFIGURE:                     │  │
│  │  Listeners, Rules, Target Groups    │  │
│  │                                     │  │
│  │  AWS HANDLES:                       │  │
│  │  Scaling, HA, Patching, Monitoring  │  │
│  └─────────────────────────────────────┘  │
└───────────────────────────────────────────┘
```

---

## Code Examples

### Python — Health Check Monitor for Any LB

```python
# lb_monitor.py — Monitor load balancer backends (works with any LB)
import httpx
import time
from dataclasses import dataclass, field
from enum import Enum

class LBType(Enum):
    NGINX = "nginx"
    HAPROXY = "haproxy"
    ALB = "alb"
    ENVOY = "envoy"

@dataclass
class Backend:
    address: str
    healthy: bool = True
    consecutive_failures: int = 0
    last_response_time_ms: float = 0

@dataclass
class LoadBalancerConfig:
    lb_type: LBType
    backends: list = field(default_factory=list)
    health_path: str = "/health"
    check_interval: int = 10
    failure_threshold: int = 3

def check_backend_health(backend: Backend, health_path: str) -> bool:
    """Check if a single backend is healthy."""
    try:
        start = time.time()
        response = httpx.get(
            f"http://{backend.address}{health_path}",
            timeout=5.0
        )
        backend.last_response_time_ms = (time.time() - start) * 1000
        
        if response.status_code == 200:
            backend.consecutive_failures = 0
            backend.healthy = True
            return True
        else:
            backend.consecutive_failures += 1
            return False
    except Exception:
        backend.consecutive_failures += 1
        return False

def run_health_monitor(config: LoadBalancerConfig):
    """Continuously monitor all backends."""
    print(f"Monitoring {config.lb_type.value} with {len(config.backends)} backends")
    
    while True:
        for backend in config.backends:
            healthy = check_backend_health(backend, config.health_path)
            
            if not healthy and backend.consecutive_failures >= config.failure_threshold:
                backend.healthy = False
                print(f"ALERT: {backend.address} is DOWN! "
                      f"({backend.consecutive_failures} failures)")
            elif healthy:
                print(f"OK: {backend.address} ({backend.last_response_time_ms:.1f}ms)")
        
        time.sleep(config.check_interval)

# Usage
config = LoadBalancerConfig(
    lb_type=LBType.NGINX,
    backends=[Backend("10.0.1.1:8080"), Backend("10.0.1.2:8080")],
)
run_health_monitor(config)
```

### Java — Load Balancer Configuration Generator

```java
// LBConfigGenerator.java — Generate configs for different LB tools
import java.util.*;
import java.util.stream.Collectors;

public class LBConfigGenerator {
    
    record Backend(String host, int port, int weight) {}
    record LBConfig(String name, List<Backend> backends, String algorithm, String healthPath) {}
    
    // Generate Nginx upstream block
    public static String generateNginxConfig(LBConfig config) {
        StringBuilder sb = new StringBuilder();
        sb.append("upstream ").append(config.name()).append(" {\n");
        
        // Algorithm directive
        switch (config.algorithm()) {
            case "least_conn" -> sb.append("    least_conn;\n");
            case "ip_hash" -> sb.append("    ip_hash;\n");
            // round_robin is default, no directive needed
        }
        
        // Server entries
        for (Backend b : config.backends()) {
            sb.append(String.format("    server %s:%d weight=%d;\n",
                b.host(), b.port(), b.weight()));
        }
        sb.append("    keepalive 32;\n");
        sb.append("}\n");
        return sb.toString();
    }
    
    // Generate HAProxy backend block
    public static String generateHAProxyConfig(LBConfig config) {
        StringBuilder sb = new StringBuilder();
        sb.append("backend ").append(config.name()).append("\n");
        sb.append("    balance ").append(config.algorithm()).append("\n");
        sb.append("    option httpchk GET ").append(config.healthPath()).append("\n");
        sb.append("    default-server inter 10s fall 3 rise 2\n");
        
        int i = 1;
        for (Backend b : config.backends()) {
            sb.append(String.format("    server srv%d %s:%d check weight %d\n",
                i++, b.host(), b.port(), b.weight()));
        }
        return sb.toString();
    }
    
    public static void main(String[] args) {
        LBConfig config = new LBConfig(
            "api_servers",
            List.of(
                new Backend("10.0.1.1", 8080, 3),
                new Backend("10.0.1.2", 8080, 2),
                new Backend("10.0.1.3", 8080, 1)
            ),
            "leastconn",
            "/health"
        );
        
        System.out.println("=== NGINX CONFIG ===");
        System.out.println(generateNginxConfig(config));
        System.out.println("=== HAPROXY CONFIG ===");
        System.out.println(generateHAProxyConfig(config));
    }
}
```

---

## Real-World Example

### Who Uses What — Industry Examples

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  NGINX:                                                              │
│  ├── Dropbox — serves billions of requests/day                      │
│  ├── WordPress.com — handles 20+ billion page views/month           │
│  ├── Netflix — uses Nginx for some edge routing                     │
│  └── Most web startups start here (simplicity + versatility)        │
│                                                                      │
│  HAPROXY:                                                            │
│  ├── GitHub — handles all git traffic through HAProxy              │
│  ├── Reddit — primary load balancer for the entire site            │
│  ├── Stack Overflow — HAProxy in front of IIS servers              │
│  ├── Airbnb — HAProxy for internal service routing                 │
│  └── Any company needing maximum raw performance                    │
│                                                                      │
│  AWS ALB/NLB:                                                        │
│  ├── Basically every company on AWS                                 │
│  ├── Slack — ALB for web traffic, NLB for real-time messaging      │
│  ├── Lyft — ALB for API routing before moving to Envoy             │
│  └── Startups (managed = less ops work)                             │
│                                                                      │
│  ENVOY:                                                              │
│  ├── Lyft — created it, uses it everywhere                         │
│  ├── Google — uses Envoy in Cloud Run, GKE                         │
│  ├── Uber — migrated to Envoy for microservices                    │
│  ├── Salesforce — Envoy for service mesh                           │
│  └── Any company with 50+ microservices                             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using Nginx when you need pure LB performance | Nginx is great but HAProxy beats it for raw throughput | Use HAProxy for high-performance TCP proxying |
| Self-managing LBs when you should use cloud-managed | Operational overhead, patching, scaling, HA setup | Use AWS ALB/NLB unless you have specific requirements |
| Choosing Envoy for a simple 3-server setup | Massive over-engineering, steep learning curve | Use Nginx or ALB for simple architectures |
| Not enabling keep-alive to backends | New TCP connection per request = terrible performance | Configure `keepalive` (Nginx) or `http-keep-alive` (HAProxy) |
| Ignoring LB metrics | Can't detect issues until users complain | Monitor: connection count, latency, error rate, queue depth |
| Using ALB for non-HTTP traffic | ALB can't handle raw TCP/database connections | Use NLB for TCP/UDP/TLS traffic |

---

## When to Use / When NOT to Use

### Decision Matrix

```
CHOOSE NGINX WHEN:
✅ You need a web server + load balancer combo
✅ Serving static files + proxying to backends
✅ Simple setup with moderate traffic
✅ Team is already familiar with Nginx
✅ Budget is limited (free and open source)

CHOOSE HAPROXY WHEN:
✅ Maximum raw load balancing performance
✅ Advanced TCP proxying (databases, message queues)
✅ Need detailed real-time statistics
✅ Complex routing rules and ACLs
✅ Enterprise-grade reliability

CHOOSE AWS ALB/NLB WHEN:
✅ Running on AWS (obvious choice)
✅ Want zero operational overhead
✅ Need auto-scaling load balancer
✅ Path/host-based routing for microservices (ALB)
✅ Ultra-low latency TCP (NLB)

CHOOSE ENVOY WHEN:
✅ Running 10+ microservices
✅ Need dynamic configuration without restarts
✅ Building/using a service mesh (Istio, etc.)
✅ Need advanced observability (tracing, metrics)
✅ Using gRPC extensively
✅ Need programmable proxy (WASM filters)
```

---

## Key Takeaways

1. **Nginx is the best starting point** — it's a web server, reverse proxy, AND load balancer. Most startups should start here.

2. **HAProxy is the performance king** — when you need maximum throughput and advanced TCP proxying, nothing beats it.

3. **AWS ALB/NLB eliminates operational burden** — if you're on AWS and don't have specific requirements for self-managed, use these.

4. **Envoy is the future for microservices** — dynamic configuration, observability, and service mesh readiness make it ideal for complex distributed systems.

5. **You don't have to choose just one** — many companies use ALB as the external entry point, Nginx for web serving, and Envoy as internal sidecar proxies.

6. **The "best" tool depends on your architecture** — a monolith on 3 servers needs Nginx; 200 microservices on Kubernetes needs Envoy.

7. **Start simple, evolve** — Nginx → ALB → Envoy is a natural progression as your architecture grows in complexity.

---

## What's Next?

These tools handle load balancing at the infrastructure level. But in modern microservices architectures, there's a more sophisticated approach: **service mesh load balancing**. In **Chapter 6.8: Service Mesh Load Balancing (Istio, Linkerd)**, we'll explore how service meshes use sidecar proxies to handle load balancing, retries, circuit breaking, and observability at the application level — without changing a single line of your code.
