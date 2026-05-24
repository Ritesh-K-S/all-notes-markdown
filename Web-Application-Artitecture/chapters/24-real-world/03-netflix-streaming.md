# How Netflix Streams to 200 Million Users

> **What you'll learn**: How Netflix delivers 15% of the world's internet bandwidth, serves 200+ million subscribers across 190 countries, and maintains a seamless streaming experience — even when entire data centers fail.

---

## Real-Life Analogy

Imagine you run a **movie theater chain** with a twist:
- You have **200 million customers**, each watching a different movie at any given moment
- Each customer can pause, rewind, fast-forward, and switch devices mid-movie
- The picture quality automatically adjusts based on their internet speed — like magic
- You've placed **mini theaters** (caches) inside every major neighborhood so movies don't travel far
- If one of your main studios burns down, customers don't even notice — movies keep playing from the local copies

That's Netflix. They turned video streaming into a globally distributed, fault-tolerant, self-healing system that accounts for 15% of all internet traffic.

---

## Core Concept Explained Step-by-Step

### Step 1: The Two Halves of Netflix

Netflix's architecture is split into two distinct parts:

```
┌────────────────────────────────────────────────────────────────────┐
│                    NETFLIX: TWO PLANES                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌────────────────────────────┐  ┌────────────────────────────┐   │
│  │     CONTROL PLANE          │  │      DATA PLANE            │   │
│  │     (AWS Cloud)            │  │      (Open Connect CDN)    │   │
│  │                            │  │                            │   │
│  │  • User authentication    │  │  • Video file storage      │   │
│  │  • Recommendations        │  │  • Video streaming         │   │
│  │  • Billing                │  │  • Edge servers worldwide  │   │
│  │  • Content metadata       │  │  • 15% of internet traffic│   │
│  │  • Search                 │  │                            │   │
│  │  • Personalization        │  │  Handles: Data-heavy ops   │   │
│  │                            │  │  (actual video bytes)     │   │
│  │  Handles: Business logic  │  │                            │   │
│  │  (lightweight API calls)  │  │                            │   │
│  └────────────────────────────┘  └────────────────────────────┘   │
│                                                                     │
│  IMPORTANT: These are SEPARATE systems. The control plane          │
│  (in AWS) tells the data plane (Open Connect) WHAT to serve.      │
│  The data plane handles the heavy lifting of streaming bytes.      │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

### Step 2: What Happens When You Press Play

```
User presses PLAY on "Stranger Things S4E1"
         │
         ▼
┌──────────────────────────────────────────────────────────────────┐
│ Step 1: AUTHENTICATE & AUTHORIZE (AWS - Control Plane)           │
│ • Is the user logged in? Valid subscription?                     │
│ • Is this content available in their region?                     │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│ Step 2: GET STREAMING URL (Playback Service)                     │
│ • Determine which video files are available                      │
│ • Pick optimal encoding (resolution, codec)                      │
│ • Generate signed URL for the nearest Open Connect server       │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│ Step 3: CONNECT TO NEAREST CDN SERVER (Open Connect - Data Plane)│
│ • Client connects to ISP's local Netflix appliance              │
│ • Video chunks start downloading                                 │
│ • Adaptive bitrate: starts low, ramps up as bandwidth allows    │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│ Step 4: ADAPTIVE STREAMING                                        │
│ • Client continuously measures available bandwidth               │
│ • Switches between quality levels (480p → 720p → 1080p → 4K)   │
│ • Buffers ahead (usually 5-30 seconds)                           │
│ • Seamless quality changes — you barely notice                   │
└──────────────────────────────────────────────────────────────────┘
```

### Step 3: Open Connect — Netflix's Custom CDN

Instead of using third-party CDNs, Netflix built their own called **Open Connect**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OPEN CONNECT CDN NETWORK                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Netflix places custom hardware INSIDE ISPs worldwide               │
│                                                                      │
│                      ┌────────────────────┐                          │
│                      │  Netflix Origin    │                          │
│                      │  (AWS S3)          │                          │
│                      │  All video files   │                          │
│                      └─────────┬──────────┘                          │
│                                │                                     │
│            During off-peak hours, pushes popular content             │
│                                │                                     │
│         ┌──────────────────────┼──────────────────────┐             │
│         ▼                      ▼                      ▼             │
│  ┌─────────────┐    ┌─────────────────┐    ┌─────────────┐         │
│  │ OCA at ISP  │    │  OCA at ISP     │    │ OCA at ISP  │         │
│  │ (Jio India) │    │  (Comcast US)   │    │ (BT UK)     │         │
│  │             │    │                 │    │             │         │
│  │ 100-200 TB  │    │  100-200 TB     │    │ 100-200 TB  │         │
│  │ of storage  │    │  of storage     │    │ of storage  │         │
│  └──────┬──────┘    └────────┬────────┘    └──────┬──────┘         │
│         │                    │                    │                  │
│         ▼                    ▼                    ▼                  │
│    ┌─────────┐         ┌─────────┐         ┌─────────┐            │
│    │ Users   │         │ Users   │         │ Users   │            │
│    │ in India│         │ in US   │         │ in UK   │            │
│    └─────────┘         └─────────┘         └─────────┘            │
│                                                                      │
│  OCA = Open Connect Appliance                                        │
│  • Custom-built servers with 100-200 TB SSDs                        │
│  • Placed directly inside ISP data centers (free for ISP!)          │
│  • Serves 90%+ of Netflix traffic without crossing the internet     │
│  • 17,000+ servers across 6,000+ locations in 190 countries         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 4: Video Encoding Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│              VIDEO ENCODING PIPELINE                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Original Master File (e.g., 4K HDR, 100+ GB)                       │
│         │                                                            │
│         ▼                                                            │
│  ┌──────────────────────────────────────────────┐                   │
│  │  SHOT-BASED ENCODING (Netflix's innovation)   │                   │
│  │                                                │                   │
│  │  Video split into "shots" (scene changes)     │                   │
│  │  Each shot encoded independently at optimal   │                   │
│  │  bitrate. Dark scenes need less data.         │                   │
│  │  Action scenes need more.                     │                   │
│  └──────────────────────┬───────────────────────┘                   │
│                         │                                            │
│         ┌───────────────┼───────────────────────────┐               │
│         ▼               ▼               ▼           ▼               │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐ ┌───────────┐        │
│  │  240p     │  │  480p     │  │  720p     │ │  1080p    │        │
│  │  H.264    │  │  H.264    │  │  H.265    │ │  H.265    │        │
│  │  ~300kbps │  │  ~1 Mbps  │  │  ~3 Mbps  │ │  ~5 Mbps  │        │
│  └───────────┘  └───────────┘  └───────────┘ └───────────┘        │
│         │               │               │           │               │
│         │          ┌────────────┐        │           │               │
│         │          │  4K HDR    │        │           │               │
│         │          │  VP9/AV1   │        │           │               │
│         │          │  ~15 Mbps  │        │           │               │
│         │          └────────────┘        │           │               │
│         │                                │           │               │
│         └────────────────────────────────┼───────────┘               │
│                                          ▼                           │
│                              ┌────────────────────┐                  │
│                              │  Each version       │                  │
│                              │  chunked into       │                  │
│                              │  4-second segments  │                  │
│                              └────────────────────┘                  │
│                                                                      │
│  Result: One movie → 1,200+ different files (versions × chunks)     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 5: Adaptive Bitrate Streaming (ABR)

```
┌─────────────────────────────────────────────────────────────────┐
│           ADAPTIVE BITRATE STREAMING                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Timeline: User watching a movie over 1 hour                     │
│                                                                   │
│  Bandwidth                                                        │
│  (Mbps)                                                          │
│    20 ─┐                                                         │
│        │      ╭──────╮                    ╭─────                 │
│    15 ─┤      │      │         ╭──╮       │                      │
│        │  ╭───╯      ╰────╮   │  │   ╭───╯                      │
│    10 ─┤  │               │   │  │   │                           │
│        │  │               ╰───╯  ╰───╯                           │
│     5 ─┤──╯                                                      │
│        │                                                          │
│     0 ─┴────────────────────────────────────────────             │
│        0    10    20    30    40    50    60 (min)                │
│                                                                   │
│  Quality selected by Netflix client:                             │
│        0-2 min:   480p (starting low, probing bandwidth)         │
│        2-10 min:  720p (bandwidth confirmed good)                │
│        10-30 min: 1080p (great bandwidth)                        │
│        30-35 min: 720p (bandwidth dropped — WiFi congestion)     │
│        35-60 min: 1080p (bandwidth recovered)                    │
│                                                                   │
│  Key: Buffer stays 5-30 seconds ahead                            │
│  If buffer drains → quality drops BEFORE buffering              │
│  User never sees "loading spinner" if ABR works correctly        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Netflix's Microservices Architecture (on AWS)

Netflix runs **1,000+ microservices** on AWS:

```
┌─────────────────────────────────────────────────────────────────────┐
│              NETFLIX CONTROL PLANE (AWS)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐                                                    │
│  │   Zuul      │  ← API Gateway (edge proxy)                       │
│  │   Gateway   │     Rate limiting, auth, routing                   │
│  └──────┬──────┘                                                    │
│         │                                                            │
│    ┌────┴────┬──────────┬──────────┬──────────┐                     │
│    ▼         ▼          ▼          ▼          ▼                     │
│ ┌──────┐ ┌──────┐ ┌──────────┐ ┌───────┐ ┌──────────┐            │
│ │User  │ │Play  │ │ Recommend│ │Search │ │ Billing  │            │
│ │Svc   │ │back  │ │ Service  │ │Service│ │ Service  │            │
│ │      │ │Svc   │ │          │ │       │ │          │            │
│ └──┬───┘ └──┬───┘ └────┬─────┘ └──┬────┘ └────┬─────┘            │
│    │         │          │          │           │                    │
│    └─────────┴──────────┴──────────┴───────────┘                    │
│                         │                                            │
│                         ▼                                            │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │            NETFLIX OSS PLATFORM LAYER                    │       │
│  │                                                         │       │
│  │  Eureka (Service Discovery)  │  Ribbon (Client LB)     │       │
│  │  Hystrix (Circuit Breaker)   │  Zuul (Edge Gateway)    │       │
│  │  Archaius (Config)           │  EVCache (Caching)      │       │
│  │  Chaos Monkey (Resilience)   │  Titus (Containers)     │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Recommendation System

Netflix's recommendation engine is arguably their most important software:

```
┌─────────────────────────────────────────────────────────────────┐
│          NETFLIX RECOMMENDATION ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Data collected per user:                                        │
│  • What they watched and when                                    │
│  • How long they watched (did they finish?)                      │
│  • Time of day watching patterns                                 │
│  • Device type (TV vs phone)                                     │
│  • Pause/rewind/fast-forward behavior                            │
│  • Browsing patterns (what they looked at but didn't play)       │
│                                                                   │
│  ┌─────────────────────────────────────────────────┐             │
│  │         OFFLINE PROCESSING (Daily/Hourly)       │             │
│  │                                                 │             │
│  │  Apache Spark + Custom ML Models                │             │
│  │                                                 │             │
│  │  1. Matrix Factorization (collaborative)        │             │
│  │  2. Deep Neural Networks                        │             │
│  │  3. Contextual Bandits (explore vs exploit)     │             │
│  │  4. User taste clusters                         │             │
│  │                                                 │             │
│  │  Output: Pre-computed recommendations          │             │
│  │  per user, stored in Cassandra/EVCache         │             │
│  └─────────────────────────────────────────────────┘             │
│                      │                                            │
│                      ▼                                            │
│  ┌─────────────────────────────────────────────────┐             │
│  │         ONLINE LAYER (Real-time)                │             │
│  │                                                 │             │
│  │  • Re-rank based on time of day                 │             │
│  │  • Apply business rules (promote new releases)  │             │
│  │  • A/B test different algorithms                │             │
│  │  • Generate artwork (personalized thumbnails!)  │             │
│  └─────────────────────────────────────────────────┘             │
│                                                                   │
│  Fun fact: Netflix generates DIFFERENT artwork/thumbnails         │
│  for the SAME show based on what you're likely to click.         │
│  Action fan? You see the action scene thumbnail.                 │
│  Romance fan? You see the romance angle.                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Chaos Engineering — Breaking Things on Purpose

Netflix pioneered chaos engineering to ensure resilience:

```
┌─────────────────────────────────────────────────────────────────┐
│              NETFLIX CHAOS ENGINEERING                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────────────────────────────────────┐                  │
│  │  Chaos Monkey                              │                  │
│  │  Randomly kills individual instances       │                  │
│  │  Runs EVERY DAY in production              │                  │
│  └────────────────────────────────────────────┘                  │
│                                                                   │
│  ┌────────────────────────────────────────────┐                  │
│  │  Chaos Kong                                │                  │
│  │  Simulates failure of an ENTIRE REGION     │                  │
│  │  (e.g., "What if all of US-East goes down?")│                 │
│  └────────────────────────────────────────────┘                  │
│                                                                   │
│  ┌────────────────────────────────────────────┐                  │
│  │  Latency Monkey                            │                  │
│  │  Injects artificial delays into requests   │                  │
│  │  Tests timeout/retry behavior              │                  │
│  └────────────────────────────────────────────┘                  │
│                                                                   │
│  ┌────────────────────────────────────────────┐                  │
│  │  Conformity Monkey                         │                  │
│  │  Finds instances not following best        │                  │
│  │  practices and shuts them down             │                  │
│  └────────────────────────────────────────────┘                  │
│                                                                   │
│  Philosophy: "If you want to find weaknesses in production,      │
│  don't wait for them to find you."                               │
│  (See Chapter 12.9: Chaos Engineering for deep dive)             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Content Delivery Decision Logic

```
User in Mumbai presses play
         │
         ▼
┌────────────────────────────┐
│  Steering Service          │
│  Decides WHICH OCA to use  │
└────────────┬───────────────┘
             │
             ├── Is content on Jio's OCA? ─── YES ──▶ Stream from Jio OCA
             │                                         (1-2ms latency)
             │
             ├── Is content on Airtel's OCA? ── YES ──▶ Stream from Airtel OCA
             │                                          (5ms latency)
             │
             ├── Is content on Mumbai IXP? ──── YES ──▶ Stream from IXP OCA
             │                                          (10ms latency)
             │
             └── Fallback: Stream from AWS origin ──▶  Stream from AWS
                                                       (100ms+ latency)
                                                       (only for rare/new content)
```

---

## Code Examples

### Python — Adaptive Bitrate Selection Algorithm

```python
# Simplified version of Netflix's adaptive bitrate algorithm
from dataclasses import dataclass
from typing import List

@dataclass
class QualityLevel:
    name: str
    bitrate_kbps: int   # Required bandwidth in kbps
    resolution: str

class AdaptiveBitrateSelector:
    """
    Netflix uses a buffer-based algorithm (similar to BBA).
    Instead of predicting bandwidth, it looks at buffer level.
    """
    
    QUALITY_LEVELS = [
        QualityLevel("240p", 300, "426x240"),
        QualityLevel("480p", 1000, "854x480"),
        QualityLevel("720p", 3000, "1280x720"),
        QualityLevel("1080p", 5000, "1920x1080"),
        QualityLevel("4K", 15000, "3840x2160"),
    ]
    
    def __init__(self):
        self.buffer_seconds = 0.0
        self.current_quality_idx = 0  # Start at lowest
        self.download_history_kbps = []
    
    def select_quality(self, buffer_seconds: float, 
                       measured_bandwidth_kbps: float) -> QualityLevel:
        """
        Buffer-based ABR: 
        - Buffer low → drop quality (avoid rebuffering)
        - Buffer high → increase quality (room to experiment)
        """
        self.buffer_seconds = buffer_seconds
        self.download_history_kbps.append(measured_bandwidth_kbps)
        
        # Conservative: use 70% of measured bandwidth
        safe_bandwidth = measured_bandwidth_kbps * 0.7
        
        # Buffer-based rules
        if buffer_seconds < 5:
            # DANGER: Buffer draining, drop to lowest immediately
            self.current_quality_idx = 0
        elif buffer_seconds < 10:
            # LOW: Don't increase, maybe decrease
            for i, level in enumerate(self.QUALITY_LEVELS):
                if level.bitrate_kbps <= safe_bandwidth:
                    self.current_quality_idx = min(i, self.current_quality_idx)
        elif buffer_seconds > 30:
            # HEALTHY: Try to increase quality
            for i, level in enumerate(self.QUALITY_LEVELS):
                if level.bitrate_kbps <= safe_bandwidth:
                    self.current_quality_idx = i  # Pick highest affordable
        
        return self.QUALITY_LEVELS[self.current_quality_idx]

# Usage
selector = AdaptiveBitrateSelector()
quality = selector.select_quality(buffer_seconds=25, measured_bandwidth_kbps=8000)
print(f"Selected: {quality.name} ({quality.resolution})")  # 1080p
```

### Java — Circuit Breaker for Microservice Calls

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

/**
 * Simplified Hystrix-style circuit breaker (Netflix invented this pattern).
 * Prevents cascading failures when a downstream service is unhealthy.
 */
public class CircuitBreaker {
    enum State { CLOSED, OPEN, HALF_OPEN }
    
    private volatile State state = State.CLOSED;
    private final AtomicInteger failureCount = new AtomicInteger(0);
    private final AtomicLong lastFailureTime = new AtomicLong(0);
    
    private final int failureThreshold = 5;      // Open after 5 failures
    private final long resetTimeoutMs = 30_000;   // Try again after 30s

    public <T> T execute(ServiceCall<T> call, ServiceCall<T> fallback) 
            throws Exception {
        if (state == State.OPEN) {
            // Check if timeout has passed → move to HALF_OPEN
            if (System.currentTimeMillis() - lastFailureTime.get() > resetTimeoutMs) {
                state = State.HALF_OPEN;
            } else {
                // Circuit is OPEN: return fallback immediately (don't call service)
                return fallback.invoke();
            }
        }
        
        try {
            T result = call.invoke();
            // Success! Reset counter
            if (state == State.HALF_OPEN) {
                state = State.CLOSED;
            }
            failureCount.set(0);
            return result;
        } catch (Exception e) {
            failureCount.incrementAndGet();
            lastFailureTime.set(System.currentTimeMillis());
            
            if (failureCount.get() >= failureThreshold) {
                state = State.OPEN;  // Trip the circuit!
                System.out.println("Circuit OPENED - too many failures");
            }
            return fallback.invoke();
        }
    }

    @FunctionalInterface
    interface ServiceCall<T> {
        T invoke() throws Exception;
    }
}
```

---

## Infrastructure Examples

### Netflix's Complete Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **CDN** | Open Connect (17,000+ servers) | Video delivery |
| **Cloud** | AWS (all regions) | Control plane |
| **Gateway** | Zuul 2 (Netty-based) | API edge proxy |
| **Service Mesh** | Custom (no Istio) | Inter-service communication |
| **Discovery** | Eureka | Service registry |
| **Database** | Cassandra, CockroachDB | User data, viewing history |
| **Cache** | EVCache (Memcached fork) | 30M+ ops/sec |
| **Messaging** | Apache Kafka | Event streaming |
| **Compute** | Titus (container platform on EC2) | Application hosting |
| **ML/Data** | Spark, Flink, Presto | Recommendations, analytics |
| **Encoding** | Custom (Cosmos platform) | Video transcoding |
| **Monitoring** | Atlas (custom) + Mantis | Metrics and alerting |

### Traffic Flow at Scale

```
┌─────────────────────────────────────────────────────────────────┐
│             NETFLIX TRAFFIC: BY THE NUMBERS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Peak concurrent streams: ~10 million simultaneously             │
│  Bandwidth served: 400+ Tbps during peak                        │
│  API requests/sec: Millions (control plane)                     │
│  Open Connect hit rate: 95%+ (only 5% goes to origin)           │
│                                                                   │
│  Traffic flow:                                                    │
│                                                                   │
│  User Request ──▶ DNS (Route 53) ──▶ AWS (API call ~50ms)       │
│                                        │                         │
│                                        ▼                         │
│                              "Play this content"                  │
│                              "Your nearest OCA is: 10.0.0.50"    │
│                                        │                         │
│                                        ▼                         │
│  User Player  ◀── Video bytes ── Open Connect Appliance         │
│               (directly from ISP's local Netflix server)         │
│               Typical: 3-15 Mbps sustained per stream            │
│                                                                   │
│  Total: 200M users × ~1.5 hrs/day avg = 300M streaming hours/day│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### How Netflix Survived AWS US-East Outage

In 2012, AWS US-East (their primary region) had a major outage. Netflix kept streaming because:

1. **Multi-region deployment**: Services replicated across 3 AWS regions
2. **Zuul routing**: Detected unhealthy region, rerouted traffic
3. **Cassandra multi-region**: Data replicated across all regions (eventual consistency)
4. **Open Connect independence**: Video delivery CDN operates independently of AWS
5. **Chaos Kong practice**: They'd already simulated this exact scenario regularly

```
Normal Operation:
  Users ──▶ US-East (primary) ──▶ Services ──▶ Cassandra
  Users ──▶ US-West (secondary) ──▶ Services ──▶ Cassandra
  Users ──▶ EU-West (secondary) ──▶ Services ──▶ Cassandra

During US-East Failure:
  Users ──╳── US-East (DOWN)
       └──▶ US-West (absorbs traffic) ──▶ Services ──▶ Cassandra
       └──▶ EU-West (absorbs traffic) ──▶ Services ──▶ Cassandra

  Video streaming: Unaffected (served by Open Connect, not AWS)
  Result: Users didn't notice anything
```

### Netflix's Encoding Innovation

Traditional encoding: Encode everything at fixed bitrates (1 Mbps, 3 Mbps, 5 Mbps...).

Netflix's per-title/per-shot encoding:
- An animated movie (simple visuals) looks great at 2 Mbps
- An action movie (complex motion) needs 8 Mbps for same perceived quality
- Netflix analyzes EACH SCENE and picks optimal bitrate

**Result**: 20% bandwidth savings (massive at their scale = millions of dollars/month saved)

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Better Approach |
|---------|---------------|-----------------|
| Using generic CDN for all video | Generic CDNs not optimized for sustained high-bandwidth streaming | Build custom CDN or use dedicated video CDN |
| Single-bitrate encoding | Wastes bandwidth or buffers on slow connections | Use adaptive bitrate with multiple quality levels |
| Monolithic streaming service | Single point of failure, can't scale independently | Separate control plane (API) from data plane (video bytes) |
| Not pre-positioning content | Every play request goes to origin = slow start | Push popular content to edge during off-peak hours |
| Testing only in lab conditions | Real networks are unpredictable (packet loss, congestion) | Use chaos engineering in production |
| Ignoring client-side intelligence | Server can't know real-time bandwidth as well as client | Put ABR logic in the player (client-driven) |

---

## When to Use / When NOT to Use

### When to Build a Netflix-Like Architecture
- Streaming large files (video, audio) to millions of concurrent users
- Content is consumed more than it's created (high read:write ratio)
- Global audience requiring low-latency delivery
- You can afford to build/maintain your own CDN ($100M+ investment)

### When NOT to Build This
- Small user base (< 100K) → Use Cloudflare Stream, AWS IVS, or Mux
- Live streaming only → Different architecture (lower latency requirements)
- User-generated content with low view counts → Standard CDN is fine
- Limited budget → Partner with existing CDNs (Akamai, Cloudflare)

---

## Key Takeaways

1. **Separate control plane from data plane**: Lightweight API calls (AWS) vs heavy video bytes (Open Connect CDN) are completely different systems
2. **Open Connect serves 95% of traffic locally**: By placing hardware inside ISPs, Netflix avoids internet congestion entirely
3. **Adaptive bitrate streaming** prevents buffering — the client dynamically switches quality based on real-time bandwidth measurements
4. **Per-title encoding** saves 20% bandwidth by analyzing visual complexity of each movie/show
5. **Chaos engineering in production** (Chaos Monkey, Chaos Kong) is what makes Netflix survive real outages — they practice failure daily
6. **Multi-region active-active** means no single point of failure — Netflix can lose an entire AWS region and keep streaming
7. **Recommendations drive 80% of content played** — the ML-powered personalization system is their competitive moat

---

## What's Next?

Next, we'll explore [How WhatsApp/Messenger Handles Billions of Messages](./04-whatsapp-messaging.md) — understanding how a tiny team of 50 engineers built a system delivering 100+ billion messages per day with end-to-end encryption.
