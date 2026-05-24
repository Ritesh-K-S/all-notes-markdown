# Spring Microservices & Cloud — Complete Notes (Beginner to Advanced)

---

## Table of Contents

1. [Introduction to Microservices](#1-introduction-to-microservices)
2. [Monolith vs Microservices](#2-monolith-vs-microservices)
3. [Microservices Design Principles](#3-microservices-design-principles)
4. [Spring Cloud Overview](#4-spring-cloud-overview)
5. [Creating Microservices with Spring Boot](#5-creating-microservices-with-spring-boot)
6. [Inter-Service Communication — RestTemplate, WebClient & OpenFeign](#6-inter-service-communication)
7. [Service Discovery — Eureka](#7-service-discovery--eureka)
8. [API Gateway — Spring Cloud Gateway](#8-api-gateway--spring-cloud-gateway)
9. [Load Balancing — Spring Cloud LoadBalancer](#9-load-balancing--spring-cloud-loadbalancer)
10. [Centralized Configuration — Spring Cloud Config](#10-centralized-configuration--spring-cloud-config)
11. [Circuit Breaker — Resilience4j](#11-circuit-breaker--resilience4j)
12. [Rate Limiter, Bulkhead & Retry — Resilience4j](#12-rate-limiter-bulkhead--retry--resilience4j)
13. [Distributed Tracing — Micrometer Tracing & Zipkin](#13-distributed-tracing--micrometer-tracing--zipkin)
14. [Centralized Logging — ELK Stack](#14-centralized-logging--elk-stack)
15. [Message-Driven Microservices — Spring Cloud Stream](#15-message-driven-microservices--spring-cloud-stream)
16. [Event-Driven Architecture with Kafka](#16-event-driven-architecture-with-kafka)
17. [Event-Driven Architecture with RabbitMQ](#17-event-driven-architecture-with-rabbitmq)
18. [Saga Pattern — Managing Distributed Transactions](#18-saga-pattern--managing-distributed-transactions)
19. [CQRS Pattern](#19-cqrs-pattern)
20. [Security in Microservices — OAuth2 & JWT](#20-security-in-microservices--oauth2--jwt)
21. [Spring Cloud Bus](#21-spring-cloud-bus)
22. [Spring Cloud Vault — Secrets Management](#22-spring-cloud-vault--secrets-management)
23. [Containerization with Docker](#23-containerization-with-docker)
24. [Docker Compose for Microservices](#24-docker-compose-for-microservices)
25. [Orchestration with Kubernetes](#25-orchestration-with-kubernetes)
26. [Deploying Spring Boot on Kubernetes](#26-deploying-spring-boot-on-kubernetes)
27. [Helm Charts for Microservices](#27-helm-charts-for-microservices)
28. [Service Mesh — Istio](#28-service-mesh--istio)
29. [Health Checks & Readiness / Liveness Probes](#29-health-checks--readiness--liveness-probes)
30. [Spring Boot Actuator in Microservices](#30-spring-boot-actuator-in-microservices)
31. [Monitoring — Prometheus & Grafana](#31-monitoring--prometheus--grafana)
32. [CI/CD Pipelines for Microservices](#32-cicd-pipelines-for-microservices)
33. [Database per Service Pattern](#33-database-per-service-pattern)
34. [API Versioning](#34-api-versioning)
35. [Contract Testing — Spring Cloud Contract](#35-contract-testing--spring-cloud-contract)
36. [12-Factor App Methodology](#36-12-factor-app-methodology)
37. [Best Practices & Production Checklist](#37-best-practices--production-checklist)
38. [Important Annotations Reference](#38-important-annotations-reference)
39. [Common Configuration Properties Reference](#39-common-configuration-properties-reference)

---

## 🔁 COMPLETE MICROSERVICES ARCHITECTURE — END-TO-END FLOW

> This diagram shows how **all the components** in a Spring Cloud Microservices ecosystem fit together. Refer back to this as you study individual sections.

```
                                    ┌─────────────────────────────────────┐
                                    │        CONFIG SERVER (:8888)        │
                                    │   (Centralized Configuration)       │
                                    │                                     │
                                    │  WHY? All services fetch their      │
                                    │  config (DB URLs, feature flags,    │
                                    │  secrets) from one place instead    │
                                    │  of maintaining separate files.     │
                                    │  Backed by Git repo or Vault.       │
                                    └──────────────────┬──────────────────┘
                                                       │ All services pull
                                                       │ config at startup
                                                       │
                                    ┌──────────────────▼──────────────────┐
                                    │     EUREKA SERVER / SERVICE         │
                                    │     REGISTRY (:8761)                │
                                    │   (Service Discovery)               │
                                    │                                     │
                                    │  WHY? Services register here so     │
                                    │  other services can find them by    │
                                    │  name instead of hardcoded IPs.    │
                                    │  Maintains all instances & their   │
                                    │  health status via heartbeats.     │
                                    └──┬───────────┬───────────┬──────────┘
                                       │           │           │
                              Register │  Register │  Register │  ← Every service with
                              + Fetch  │  + Fetch  │  + Fetch  │    eureka-client dependency
                              Registry │  Registry │  Registry │    does BOTH by default
                                       │           │           │
┌──────────┐  HTTPS    ┌───────────────▼───────────▼───────────▼──────────────────┐
│          │──────────►│                  API GATEWAY (:8080)                     │
│  CLIENT  │           │              (Spring Cloud Gateway)                      │
│ (Browser/│◄──────────│                                                         │
│  Mobile) │           │  WHY? Single entry point for ALL client requests.       │
│          │           │  • Routes requests to correct service (by path/header)  │
└──────────┘           │  • Authenticates (JWT validation) before forwarding     │
                       │  • Rate limiting, logging, CORS — applied globally     │
                       │  • Load balances using lb:// prefix + Eureka registry  │
                       └─────┬──────────────────┬──────────────────┬─────────────┘
                             │                  │                  │
                    ┌────────▼───────┐ ┌────────▼───────┐ ┌───────▼────────┐
                    │ USER SERVICE   │ │ ORDER SERVICE  │ │PRODUCT SERVICE │
                    │   (:8081)      │ │   (:8082)      │ │   (:8083)      │
                    │                │ │                │ │                │
                    │ Has eureka-    │ │ Calls User     │ │                │
                    │ client dep →   │ │ Service via    │ │                │
                    │ registers +    │ │ @LoadBalanced  │ │                │
                    │ fetches        │ │ RestTemplate / │ │                │
                    │ registry       │ │ @FeignClient   │ │                │
                    └───────┬────────┘ └──┬──────┬──────┘ └───────┬────────┘
                            │             │      │                │
                   ┌────────▼────────┐    │      │       ┌────────▼────────┐
                   │    User DB      │    │      │       │   Product DB    │
                   │  (PostgreSQL)   │    │      │       │   (MongoDB)     │
                   └─────────────────┘    │      │       └─────────────────┘
                                          │      │
                             ┌────────────▼──┐   │       ┌─────────────────┐
                             │   Order DB    │   │       │                 │
                             │  (PostgreSQL) │   └──────►│  MESSAGE BROKER │
                             └───────────────┘    Events │ (Kafka/RabbitMQ)│
                                                        │                 │
                                                        │ WHY? Async      │
                                                        │ communication   │
                                                        │ between services│
                                                        │ for decoupling  │
                                                        │ (e.g., order    │
                                                        │ created event → │
                                                        │ notification)   │
                                                        └────────┬────────┘
                                                                 │
                                                        consumed by other
                                                        services (consumers)
```

### Inter-Service Communication Flow (Detailed)
```
ORDER SERVICE wants to call USER SERVICE:

Step 1: Order Service has eureka-client dependency
        → On startup, it REGISTERS itself with Eureka AND FETCHES the full registry cache

Step 2: Order Service uses @LoadBalanced RestTemplate (or @FeignClient)
        → Makes call: restTemplate.getForObject("http://user-service/api/users/1", ...)

Step 3: @LoadBalanced interceptor kicks in
        → Looks up "user-service" in the LOCAL registry cache (fetched from Eureka)
        → Finds: user-service → [192.168.1.10:8081, 192.168.1.11:8081, 192.168.1.12:8081]

Step 4: Load Balancer picks ONE instance (Round Robin by default)
        → Selected: 192.168.1.10:8081

Step 5: Actual HTTP call made
        → GET http://192.168.1.10:8081/api/users/1

Step 6: If the call FAILS → Circuit Breaker (Resilience4j) activates
        → After threshold failures → OPEN state → fallback method called
        → After wait duration → HALF_OPEN → tries again
```

### Observability & Resilience Layer
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OBSERVABILITY STACK                                 │
│                                                                           │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│  │   ZIPKIN     │    │  PROMETHEUS  │    │   ELK STACK  │                  │
│  │             │    │              │    │   / Loki     │                  │
│  │ WHY?        │    │ WHY?         │    │              │                  │
│  │ Distributed │    │ Collects     │    │ WHY?         │                  │
│  │ tracing —   │    │ metrics from │    │ Centralized  │                  │
│  │ tracks a    │    │ all services │    │ logging —    │                  │
│  │ request     │    │ (CPU, memory,│    │ search logs  │                  │
│  │ across ALL  │    │ request rate,│    │ across all   │                  │
│  │ services    │    │ errors) and  │    │ services in  │                  │
│  │ with a      │    │ stores them  │    │ one place    │                  │
│  │ single      │    │ for alerting │    │ using traceId│                  │
│  │ Trace ID    │    │              │    │              │                  │
│  └──────┬──────┘    └──────┬───────┘    └──────┬───────┘                  │
│         │                  │                   │                          │
│         │           ┌──────▼───────┐           │                          │
│         │           │   GRAFANA    │           │                          │
│         │           │ (Dashboards  │           │                          │
│         │           │  & Alerts)   │           │                          │
│         │           └──────────────┘           │                          │
└─────────┼──────────────────────────────────────┼──────────────────────────┘
          │                                      │
          └──── All services send traces ────────┘
                and logs automatically
```

### Resilience Patterns (Applied at Service Level)
```
Incoming Request
       │
       ▼
   ┌─── RETRY ───┐           WHY? Transient failures (network blip) may
   │  max 3      │                succeed on retry
   │  attempts   │
   └──────┬──────┘
          ▼
   ┌─ CIRCUIT BREAKER ─┐     WHY? If downstream is DOWN, stop hammering it.
   │  CLOSED → normal  │          Fail fast with fallback instead of waiting
   │  OPEN → blocked   │
   │  HALF_OPEN → test │
   └──────┬─────────────┘
          ▼
   ┌─ RATE LIMITER ─┐        WHY? Protect service from being overwhelmed
   │  10 req/sec    │              by too many requests
   └──────┬─────────┘
          ▼
   ┌─ BULKHEAD ─┐            WHY? Isolate thread pools so one slow
   │  max 10    │                  downstream call can't exhaust all threads
   │  concurrent│
   └──────┬─────┘
          ▼
     Actual Method Call
```

### Quick Reference — What Each Component Does
| Component | What? | Why? | Spring Cloud Project |
|-----------|-------|------|---------------------|
| **Config Server** | Centralized config store | Avoid duplicating config files; change config without redeployment | Spring Cloud Config |
| **Eureka Server** | Service Registry | Services find each other by name, not by hardcoded IP:port | Netflix Eureka |
| **API Gateway** | Single entry point, router | Routing, auth, rate limiting, CORS — all in one place before requests reach services | Spring Cloud Gateway |
| **Load Balancer** | Distributes requests across instances | No single instance gets overloaded; if one dies, others serve | Spring Cloud LoadBalancer |
| **Circuit Breaker** | Fault tolerance wrapper | Prevents cascade failures; provides fallback responses | Resilience4j |
| **Distributed Tracing** | Tracks request across services | Debug slow requests; see the full chain of a single user request | Micrometer + Zipkin |
| **Message Broker** | Async event communication | Decouple services; fire-and-forget; eventual consistency | Kafka / RabbitMQ |
| **Centralized Logging** | Aggregated logs | Search/filter logs from all services in one dashboard | ELK / Loki |
| **Monitoring** | Metrics collection + visualization | Know when something is wrong BEFORE users complain | Prometheus + Grafana |

---

## 1. INTRODUCTION TO MICROSERVICES

### What are Microservices?
- **Microservices** = an architectural style where an application is composed of **small, independent, loosely coupled services**, each running in its own process and communicating via lightweight protocols (usually HTTP/REST or messaging).
- Each service is **independently deployable**, **scalable**, and **maintainable**.
- Coined / popularized by **Martin Fowler** and **James Lewis** (2014).

### Key Characteristics
| Characteristic | Description |
|---------------|-------------|
| **Single Responsibility** | Each service does one thing well |
| **Independently Deployable** | Deploy one service without affecting others |
| **Decentralized Data** | Each service owns its own database |
| **Lightweight Communication** | REST, gRPC, messaging (Kafka, RabbitMQ) |
| **Technology Agnostic** | Each service can use a different tech stack |
| **Fault Isolation** | Failure in one service doesn't crash the system |
| **Team Autonomy** | Small teams own individual services end-to-end |

### When to Use Microservices
- Large, complex applications with multiple teams.
- Need for independent scaling of certain components.
- Frequent releases and continuous deployment.
- Different parts need different technology stacks.

### When NOT to Use
- Small/simple applications (overhead not worth it).
- Small team (coordination cost > benefit).
- Prototype or MVP stage.
- No DevOps maturity (CI/CD, monitoring, logging).

---

## 2. MONOLITH VS MICROSERVICES

### Comparison Table
| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| Codebase | Single codebase | Multiple codebases |
| Deployment | Deploy entire app | Deploy individual services |
| Scaling | Scale entire app | Scale specific services |
| Technology | Single tech stack | Polyglot (mix of techs) |
| Database | Shared database | Database per service |
| Team Size | Any size | Small, autonomous teams |
| Complexity | Simple initially | Complex infrastructure |
| Fault Isolation | Single failure can crash all | Failures are isolated |
| Communication | In-process method calls | Network calls (HTTP, messaging) |
| Testing | Easier (single app) | Harder (integration across services) |
| DevOps Requirement | Low | High |

### Migration Path: Monolith → Microservices
1. **Identify bounded contexts** (Domain-Driven Design).
2. **Strangler Fig Pattern** — gradually replace monolith modules with services.
3. **Start with the edge** — pick a non-critical module first.
4. **Anti-Corruption Layer** — adapter between old and new systems.
5. **Database decomposition** — split shared DB into per-service DBs.

---

## 3. MICROSERVICES DESIGN PRINCIPLES

### Core Principles
1. **Single Responsibility Principle** — one service = one business capability.
2. **Loose Coupling** — services know as little as possible about each other.
3. **High Cohesion** — related functionality stays together in one service.
4. **API First** — design the contract before implementation.
5. **Design for Failure** — expect network calls to fail; use retries, circuit breakers, fallbacks.
6. **Decentralize Everything** — data, governance, technology choices.
7. **Automate Everything** — CI/CD, testing, infrastructure provisioning.

### Domain-Driven Design (DDD) — Key Concepts
| Concept | Description |
|---------|-------------|
| **Bounded Context** | Logical boundary around a domain model |
| **Aggregate** | Cluster of entities treated as a single unit |
| **Entity** | Object with a unique identity |
| **Value Object** | Object defined by its attributes (no identity) |
| **Domain Event** | Something that happened in the domain |
| **Ubiquitous Language** | Shared language between devs and domain experts |

### Common Microservices Patterns
| Pattern | Purpose |
|---------|---------|
| API Gateway | Single entry point, routing, auth |
| Service Discovery | Dynamic service registration & lookup |
| Circuit Breaker | Prevent cascade failures |
| Saga | Manage distributed transactions |
| CQRS | Separate read and write models |
| Event Sourcing | Store state changes as events |
| Strangler Fig | Incremental monolith migration |
| Sidecar | Attach helper processes to services |
| Bulkhead | Isolate resources per service |
| Database per Service | Decentralized data ownership |

---

## 4. SPRING CLOUD OVERVIEW

### What is Spring Cloud?
- **Spring Cloud** = a suite of tools and frameworks for building **cloud-native, distributed microservices** on top of Spring Boot.
- Provides solutions for common distributed system patterns: config management, service discovery, circuit breakers, routing, tracing, etc.
- Current release train: **Spring Cloud 2023.x / 2024.x** (aligned with Spring Boot 3.x).

### Spring Cloud Components
| Component | Purpose | Project |
|-----------|---------|---------|
| Config Server | Centralized external configuration | Spring Cloud Config |
| Service Discovery | Register & find services | Spring Cloud Netflix Eureka / Consul |
| API Gateway | Routing, filtering, rate limiting | Spring Cloud Gateway |
| Load Balancer | Client-side load balancing | Spring Cloud LoadBalancer |
| Circuit Breaker | Fault tolerance | Resilience4j |
| Distributed Tracing | Request tracing across services | Micrometer Tracing + Zipkin |
| Messaging | Event-driven communication | Spring Cloud Stream (Kafka/RabbitMQ) |
| Bus | Broadcast config changes | Spring Cloud Bus |
| Contract | Consumer-driven contract testing | Spring Cloud Contract |
| Vault | Secrets management | Spring Cloud Vault |
| Kubernetes | K8s native integration | Spring Cloud Kubernetes |

### Deprecated / Replaced Components
| Old (Netflix OSS) | Replacement |
|-------------------|-------------|
| Hystrix | Resilience4j |
| Ribbon | Spring Cloud LoadBalancer |
| Zuul | Spring Cloud Gateway |
| Archaius | Spring Cloud Config |

### Dependencies (BOM)
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## 5. CREATING MICROSERVICES WITH SPRING BOOT

### Typical Multi-Service Architecture
```
                        ┌─────────────────┐
                        │   API Gateway    │
                        │ (Spring Cloud GW)│
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                   │
     ┌────────▼──────┐  ┌───────▼───────┐  ┌───────▼───────┐
     │ User Service  │  │ Order Service │  │Product Service│
     │   (port 8081) │  │  (port 8082)  │  │  (port 8083)  │
     └───────┬───────┘  └───────┬───────┘  └───────┬───────┘
             │                  │                   │
     ┌───────▼───────┐  ┌──────▼────────┐  ┌──────▼────────┐
     │   User DB     │  │   Order DB    │  │  Product DB   │
     └───────────────┘  └───────────────┘  └───────────────┘
```

### Example: Creating a Simple Microservice

**1. pom.xml**
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

**2. application.yml**
```yaml
spring:
  application:
    name: user-service       # unique name for this microservice — used for discovery & config
server:
  port: 8081                 # port this service runs on — each service uses a different port
```

**3. Main Class**
```java
@SpringBootApplication
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

**4. Entity, Repository, Service, Controller** — follow standard Spring Boot REST patterns.

### Multi-Module Maven Project (Optional)
```
parent-project/
├── pom.xml (parent POM)
├── user-service/
│   └── pom.xml
├── order-service/
│   └── pom.xml
├── product-service/
│   └── pom.xml
├── api-gateway/
│   └── pom.xml
└── config-server/
    └── pom.xml
```

---

## 6. INTER-SERVICE COMMUNICATION

### Communication Styles
| Style | Type | Examples | When to Use |
|-------|------|----------|-------------|
| **Synchronous** | Request-Response | REST, gRPC, GraphQL | Need immediate response |
| **Asynchronous** | Event-Driven | Kafka, RabbitMQ, SQS | Fire-and-forget, decoupling |

### RestTemplate (Legacy — Deprecated in Spring 6.1)
```java
@Bean
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// Usage
@Service
public class OrderService {
    @Autowired
    private RestTemplate restTemplate;

    public UserDto getUser(Long userId) {
        return restTemplate.getForObject(
            "http://user-service/api/users/{id}", UserDto.class, userId);
    }
}
```

### WebClient (Reactive — Recommended)
```java
@Bean
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}

// Usage
@Service
public class OrderService {
    @Autowired
    private WebClient.Builder webClientBuilder;

    public Mono<UserDto> getUser(Long userId) {
        return webClientBuilder.build()
            .get()
            .uri("http://user-service/api/users/{id}", userId)
            .retrieve()
            .bodyToMono(UserDto.class);
    }
}
```

### RestClient (Spring 6.1+ — New Recommended Blocking Client)
```java
@Bean
public RestClient restClient() {
    return RestClient.builder()
        .baseUrl("http://user-service")
        .build();
}

// Usage
UserDto user = restClient.get()
    .uri("/api/users/{id}", userId)
    .retrieve()
    .body(UserDto.class);
```

### OpenFeign (Declarative — Most Popular in Microservices)
**Dependency:**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

**Enable Feign:**
```java
@SpringBootApplication
@EnableFeignClients   // WHY? Tells Spring to scan for interfaces annotated with @FeignClient
                      // and create proxy implementations automatically at runtime.
                      // Without this, @FeignClient interfaces won't be picked up.
public class OrderServiceApplication { }
```

**Feign Client Interface:**
```java
@FeignClient(name = "user-service", url = "http://localhost:8081")
// name = service name in Eureka (used for discovery + LB when url is not specified)
// url = hardcoded URL (only for local dev / testing — REMOVE in production, let Eureka resolve)
public interface UserClient {

    @GetMapping("/api/users/{id}")
    UserDto getUserById(@PathVariable("id") Long id);

    @PostMapping("/api/users")
    UserDto createUser(@RequestBody UserDto userDto);
}
```

**Usage:**
```java
@Service
public class OrderService {
    @Autowired
    private UserClient userClient;

    public OrderResponse getOrder(Long orderId) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        UserDto user = userClient.getUserById(order.getUserId());
        return new OrderResponse(order, user);
    }
}
```

**Feign with Service Discovery (no hardcoded URL):**
```java
// name = service name registered in Eureka; load balancing is automatic
@FeignClient(name = "user-service")  // resolved via Eureka, no url needed
public interface UserClient { ... }
```

> **Note:** For `RestTemplate` / `WebClient` / `RestClient`, you use `@LoadBalanced` on the bean and replace `host:port` with the service name in the URL. For `FeignClient`, the `name` attribute itself acts as the service identifier — no `@LoadBalanced` needed.

### gRPC (High Performance)
- Binary protocol using **Protocol Buffers**.
- Much faster than REST (HTTP/2, streaming).
- Use `grpc-spring-boot-starter`.
- Best for internal service-to-service communication.

---

## 7. SERVICE DISCOVERY — EUREKA

### What is Service Discovery?
- In microservices, services run on **dynamic IPs and ports**.
- **Service Discovery** = a mechanism for services to **register themselves** and **discover other services** without hardcoding addresses.

### How Eureka Works
```
┌──────────────┐     Register     ┌─────────────────┐
│ User Service │ ──────────────►  │  Eureka Server   │
│  (instance 1)│                  │ (Service Registry)│
└──────────────┘  ◄────────────── └─────────────────┘
                    Heartbeat              │
                                          │  Query
┌──────────────┐                          ▼
│ Order Service│  ── fetches registry ── gets user-service address
└──────────────┘
```

### Setting Up Eureka Server

**Dependency:**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

**Main Class:**
```java
@SpringBootApplication
@EnableEurekaServer   // WHY? Marks this app as the Eureka Server (Service Registry)
                      // Without this, it's just a normal Spring Boot app
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

**application.yml:**
```yaml
server:
  port: 8761                         # default Eureka port — all clients point here

eureka:
  client:
    register-with-eureka: false      # WHY false? This IS the server — no need to register itself
                                     # (in HA setup with multiple Eureka servers, set to true so
                                     #  they register with each other)
    fetch-registry: false            # WHY false? Server doesn't need to fetch its own registry
                                     # (in HA setup, set to true to sync with peer Eureka servers)
  instance:
    hostname: localhost
```

### Registering a Service (Eureka Client)

**Dependency:**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

**application.yml:**
```yaml
spring:
  application:
    name: user-service              # IMPORTANT: this name is how OTHER services find this service
                                    # e.g., @FeignClient(name = "user-service") or
                                    # restTemplate.getForObject("http://user-service/...", ...)

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/  # WHERE is the Eureka Server?
                                                  # This service will register here AND fetch
                                                  # the registry from here
  instance:
    prefer-ip-address: true         # WHY? Use IP instead of hostname — in Docker/K8s,
                                    # hostnames may not resolve across containers
```

- `@EnableDiscoveryClient` is **optional** in Spring Boot 3.x (auto-detected via classpath).
  - WHY optional? Spring Boot auto-detects the `eureka-client` dependency on classpath and enables discovery automatically.

### How Discovery Client Works — Two Key Behaviors

Any service with the `spring-cloud-starter-netflix-eureka-client` dependency (including the discovery server itself) performs **two things** by default:

| Behavior | Property | Default | Description |
|----------|----------|---------|-------------|
| **Register itself** | `eureka.client.register-with-eureka` | `true` | Registers this service instance with Eureka Server |
| **Fetch registry** | `eureka.client.fetch-registry` | `true` | Fetches the registry cache of all registered services |

- For the **Eureka Server** itself, both are set to `false` (since it IS the registry — no need to register with or fetch from itself in a single-node setup).
- For **all client services**, both default to `true` — the service registers itself AND fetches the full registry cache locally so it can discover other services.
- The registry is cached locally and refreshed periodically (default every **30 seconds**) via the property `eureka.client.registry-fetch-interval-seconds`.

### Instance Metadata

Services can attach **custom metadata** when registering with Eureka. This is useful for passing information like context root, version, region, etc.

```yaml
eureka:
  instance:
    metadata-map:                   # custom key-value pairs attached to this instance in Eureka
      context-root: /api/v1         # app context path — other services can read this
      version: 2.0                  # app version — useful for canary routing
      region: us-east-1             # deployment region — useful for zone-aware routing
      environment: production       # env label — useful for filtering instances
```

- Other services can read this metadata after fetching the registry.
- Useful for routing decisions, canary deployments, or passing contextual info.

### Manual Service Discovery — DiscoveryClient & LoadBalancerClient

Instead of relying on `@LoadBalanced` REST clients, you can **manually** look up service instances:

**Using DiscoveryClient (get all instances):**
```java
@Autowired
private DiscoveryClient discoveryClient;

public List<ServiceInstance> getUserServiceInstances() {
    return discoveryClient.getInstances("user-service");
}

// Then manually pick an instance and build the URL
public String callUserService(Long userId) {
    List<ServiceInstance> instances = discoveryClient.getInstances("user-service");
    ServiceInstance instance = instances.get(0); // manual selection
    String url = instance.getUri() + "/api/users/" + userId;
    return restTemplate.getForObject(url, String.class);
}
```

**Using LoadBalancerClient (auto-selects one instance with LB strategy):**
```java
@Autowired
private LoadBalancerClient loadBalancerClient;

public String callUserService(Long userId) {
    ServiceInstance instance = loadBalancerClient.choose("user-service");
    String url = instance.getUri() + "/api/users/" + userId;
    return restTemplate.getForObject(url, String.class);
}
```

| Class | Method | Purpose |
|-------|--------|---------|
| `DiscoveryClient` | `getInstances(serviceId)` | Returns all registered instances of a service |
| `DiscoveryClient` | `getServices()` | Returns all registered service names |
| `LoadBalancerClient` | `choose(serviceId)` | Picks one instance using the configured LB strategy |

> **Note:** These are useful for advanced scenarios (custom routing, manual URL construction). For most cases, `@LoadBalanced` on REST clients is simpler and preferred.

### Eureka Dashboard
- Access at `http://localhost:8761`.
- Shows registered services, instances, status.

### Self-Preservation Mode
- If Eureka doesn't receive heartbeats from many clients, it enters **self-preservation** — stops evicting instances.
- Prevents mass deregistration during **network partitions**.
- Disable in dev (not recommended in production):
```yaml
eureka:
  server:
    enable-self-preservation: false   # Only disable in DEV — in PROD, self-preservation
                                      # prevents Eureka from removing services during
                                      # temporary network issues (it assumes the services
                                      # are still alive, just can't reach Eureka)
```

### Multiple Eureka Instances (HA)
```yaml
# Eureka Server 1 — peers with Server 2
eureka:
  client:
    service-url:
      defaultZone: http://eureka2:8762/eureka/   # register with peer Eureka Server 2

# Eureka Server 2 — peers with Server 1
eureka:
  client:
    service-url:
      defaultZone: http://eureka1:8761/eureka/   # register with peer Eureka Server 1
```

### Alternatives to Eureka
| Tool | Type | Notes |
|------|------|-------|
| **Consul** | HashiCorp | Health checks, KV store |
| **Zookeeper** | Apache | Used by Kafka |
| **Kubernetes DNS** | K8s native | Built-in in K8s |

---

## 8. API GATEWAY — SPRING CLOUD GATEWAY

### What is an API Gateway?
- **Single entry point** for all client requests.
- Handles **routing**, **filtering**, **authentication**, **rate limiting**, **load balancing**.
- Replaces the deprecated **Netflix Zuul**.

### Architecture
```
Client ──► API Gateway ──► Service Discovery ──► Target Service
              │
              ├── Route matching
              ├── Pre-filters (auth, logging, rate limiting)
              ├── Load balancing
              └── Post-filters (response modification)
```

### Setup

**Dependency:**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

> **Note:** Spring Cloud Gateway is built on **Spring WebFlux** (reactive). Do NOT include `spring-boot-starter-web`.

### Route Configuration (YAML)
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service                 # unique route identifier
          uri: lb://USER-SERVICE           # lb:// = load-balanced via Eureka
                                           # WHY lb://? Gateway looks up "USER-SERVICE" in
                                           # Eureka registry, gets all instances, and load
                                           # balances across them automatically
                                           # WITHOUT lb://, you'd hardcode: http://localhost:8081
          predicates:
            - Path=/api/users/**           # WHEN? Route matches if path starts with /api/users/
          filters:
            - StripPrefix=0                # HOW MANY path segments to remove before forwarding
                                           # 0 = forward as-is; 1 = remove first segment

        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/api/orders/**

        - id: product-service
          uri: lb://PRODUCT-SERVICE
          predicates:
            - Path=/api/products/**
```

### Route Configuration (Java DSL)
```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r
                .path("/api/users/**")
                .uri("lb://USER-SERVICE"))
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f.addRequestHeader("X-Source", "gateway"))
                .uri("lb://ORDER-SERVICE"))
            .build();
    }
}
```

### Built-in Predicates
| Predicate | Example |
|-----------|---------|
| `Path` | `Path=/api/users/**` |
| `Method` | `Method=GET,POST` |
| `Header` | `Header=X-Custom, \\d+` |
| `Query` | `Query=name, abc` |
| `Host` | `Host=**.example.com` |
| `After` | `After=2024-01-01T00:00:00` |
| `Before` | `Before=2025-01-01T00:00:00` |
| `Between` | `Between=datetime1, datetime2` |
| `Cookie` | `Cookie=session, abc` |
| `RemoteAddr` | `RemoteAddr=192.168.1.0/24` |

### Built-in Filters
| Filter | Purpose |
|--------|---------|
| `AddRequestHeader` | Add header to downstream request |
| `AddResponseHeader` | Add header to response |
| `StripPrefix` | Remove N path segments |
| `PrefixPath` | Prepend path segments |
| `RewritePath` | Regex-based path rewrite |
| `Retry` | Retry failed requests |
| `CircuitBreaker` | Integrate with Resilience4j |
| `RequestRateLimiter` | Rate limiting (Redis-backed) |
| `SetStatus` | Override response status |
| `RedirectTo` | HTTP redirect |

### Custom Global Filter
```java
@Component
public class LoggingFilter implements GlobalFilter, Ordered {

    private static final Logger log = LoggerFactory.getLogger(LoggingFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("Request path: {}", exchange.getRequest().getPath());
        log.info("Request method: {}", exchange.getRequest().getMethod());
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -1;  // higher priority
    }
}
```

### Authentication Filter (JWT Validation at Gateway)

> **Gateway does Authentication only (AuthN), NOT Authorization (AuthZ).**
> The Gateway validates the token (is it real? is it expired?) and forwards it.
> Individual microservices then handle Authorization (does this user have the right role/permission?).
> For full OAuth2 architecture with Keycloak/Okta, see **[Section 20: Security in Microservices](#20-security-in-microservices--oauth2--jwt)**.

```java
@Component
public class AuthenticationFilter implements GlobalFilter, Ordered {

    @Autowired
    private JwtUtil jwtUtil;  // validates JWT signature + expiry using cached JWK public keys

    private final List<String> openEndpoints = List.of(
        "/api/auth/login", "/api/auth/register"  // public endpoints — no token required
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().toString();

        // STEP 1: Skip auth for whitelisted/public endpoints (login, register, health)
        if (openEndpoints.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        // STEP 2: Extract token from Authorization header
        String authHeader = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();  // 401 — no token provided
        }

        // STEP 3: Validate token (signature, expiry, issuer) — NO network call needed
        //         Uses JWK public keys cached from Auth Server at startup
        String token = authHeader.substring(7);
        if (!jwtUtil.isValidToken(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();  // 401 — invalid/expired token
        }

        // STEP 4: Token is valid → forward request to downstream service
        //         The original JWT is passed along in the Authorization header
        //         Downstream service will handle AUTHORIZATION (role/permission checks)
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

### Rate Limiting with Redis
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/users/**
          filters:
            - name: RequestRateLimiter                        # built-in rate limiter filter (requires Redis)
              args:
                redis-rate-limiter.replenishRate: 10          # how many requests/sec to allow (steady rate)
                redis-rate-limiter.burstCapacity: 20          # max burst of requests allowed at once
                key-resolver: "#{@userKeyResolver}"           # SpEL — which bean decides the rate-limit key
                                                              # (e.g., limit per IP, per user, per API key)
```

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
    );
}
```

---

## 9. LOAD BALANCING — SPRING CLOUD LOADBALANCER

### What is Client-Side Load Balancing?
- The **client** (calling service) decides which instance to call.
- Unlike server-side LB (NGINX, HAProxy), the client has the registry information.
- Spring Cloud LoadBalancer replaced **Netflix Ribbon**.

### How It Works
```
Order Service ──► LoadBalancer ──► Eureka Registry
                     │
                     ├── user-service:8081 (instance 1)
                     ├── user-service:8082 (instance 2)
                     └── user-service:8083 (instance 3)
```

### Setup
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

### How Client-Side Load Balancing Works Internally
1. Every REST client (`RestTemplate`, `WebClient`, `RestClient`) annotated with `@LoadBalanced` internally uses the **Discovery Client** to fetch the **registry cache** from Eureka.
2. When a request is made using a **service name** (e.g., `http://user-service/api/users/1`), the load balancer intercepts the call.
3. It looks up all available instances of `user-service` from the cached registry.
4. It applies the **load balancing strategy** (default: Round Robin) to pick one instance.
5. The service name in the URL is replaced with the actual `host:port` of the chosen instance, and the request is sent.

```
restTemplate.getForObject("http://user-service/api/users/1", ...)
        │
        ▼
   @LoadBalanced interceptor
        │
        ▼
   Fetch instances of "user-service" from local registry cache
        │
        ▼
   Apply LB strategy → pick instance (e.g., 192.168.1.10:8081)
        │
        ▼
   Actual call: GET http://192.168.1.10:8081/api/users/1
```

### @LoadBalanced Annotation
- `@LoadBalanced` is placed on the **bean definition** of the REST client (`RestTemplate`, `WebClient.Builder`).
- It enables a **client-side load balancing interceptor** that resolves service names to actual instance addresses.
- **Important:** When using `@LoadBalanced`, you must use the **service name** (as registered in Eureka) in place of `host:port` in the URL.
- Without `@LoadBalanced`, using a service name like `http://user-service/...` will fail with `UnknownHostException`.

### Using with RestTemplate
```java
@Bean
@LoadBalanced   // enables LB — use service name instead of host:port
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// Use service name "user-service" instead of "localhost:8081"
restTemplate.getForObject("http://user-service/api/users/1", UserDto.class);
```

### Using with WebClient
```java
@Bean
@LoadBalanced   // enables LB — use service name instead of host:port
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}

webClientBuilder.build()
    .get()
    .uri("http://user-service/api/users/{id}", 1)
    .retrieve()
    .bodyToMono(UserDto.class);
```

### Using with OpenFeign
- FeignClient does **NOT** use `@LoadBalanced`. Instead, it uses the `name` (or `value`) parameter in the `@FeignClient` annotation.
- The `name` must match the **service name** registered in Eureka.
- Feign automatically integrates with Spring Cloud LoadBalancer to resolve and load-balance across instances.

```java
@FeignClient(name = "user-service")   // "user-service" = Eureka service name, LB is automatic
public interface UserClient {
    @GetMapping("/api/users/{id}")
    UserDto getUserById(@PathVariable("id") Long id);
}
```

### Summary: How Each REST Client Enables Load Balancing
| REST Client | How to Enable LB | Service Name Usage |
|-------------|-------------------|--------------------|
| `RestTemplate` | `@LoadBalanced` on `@Bean` | Use service name in URL: `http://service-name/...` |
| `WebClient` | `@LoadBalanced` on `WebClient.Builder` `@Bean` | Use service name in URL: `http://service-name/...` |
| `RestClient` | `@LoadBalanced` on `@Bean` | Use service name in URL: `http://service-name/...` |
| `OpenFeign` | `@FeignClient(name = "service-name")` | Name parameter = Eureka service name, no URL needed |

### Load Balancing Strategies
| Strategy | Description |
|----------|-------------|
| **RoundRobin** (default) | Cycles through instances sequentially |
| **Random** | Picks a random instance |
| **Custom** | Implement `ReactorServiceInstanceLoadBalancer` |

### Custom Load Balancer
```java
public class CustomLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment env, LoadBalancerClientFactory factory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            factory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}

// Apply to specific service
@LoadBalancerClient(name = "user-service",
                    configuration = CustomLoadBalancerConfig.class)
public class OrderServiceApplication { }
```

---

## 10. CENTRALIZED CONFIGURATION — SPRING CLOUD CONFIG

### Why Centralized Config?
- Microservices have **many config files** across many services.
- Changing config requires **redeployment** without centralization.
- Environment-specific configs (dev, staging, prod) need management.

### Architecture
```
┌─────────────┐         ┌──────────────────┐         ┌──────────┐
│ Microservice│ ──GET─► │ Config Server    │ ──────► │ Git Repo │
│ (client)    │         │ (Spring Cloud)   │         │(or Vault)│
└─────────────┘         └──────────────────┘         └──────────┘
```

### Config Server Setup

**Dependency:**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

**Main Class:**
```java
@SpringBootApplication
@EnableConfigServer   // WHY? Makes this app a centralized configuration server.
                      // All other microservices fetch their config from here at startup
                      // instead of each maintaining their own application.yml
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

**application.yml:**
```yaml
server:
  port: 8888                              # standard Config Server port

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo   # WHERE are configs stored?
                                                         # Git = versioned, auditable config
          default-label: main                             # which Git branch to read from
          search-paths: '{application}'     # folder per service — e.g., user-service/
                                            # {application} is replaced by spring.application.name
                                            # of the requesting service
          clone-on-start: true              # WHY? Clone repo at startup to fail fast if
                                            # Git repo is unreachable, instead of failing
                                            # on first client request
```

### Config Repository Structure
```
config-repo/
├── application.yml               # shared across all services
├── user-service.yml             # user-service specific
├── user-service-dev.yml         # user-service + dev profile
├── user-service-prod.yml        # user-service + prod profile
├── order-service.yml
└── order-service-prod.yml
```

### Config Client Setup

**Dependency:**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

**application.yml:**
```yaml
spring:
  application:
    name: user-service               # Config Server uses this name to find the right config file
                                     # e.g., fetches user-service-dev.yml from the Git repo
  profiles:
    active: dev                      # combined with name → fetches user-service-dev.yml
  config:
    import: configserver:http://localhost:8888   # WHERE is the Config Server?
                                                 # This service will pull its config from here
                                                 # BEFORE starting up
```

### Accessing Config
- **URL pattern:** `http://config-server:8888/{application}/{profile}/{label}`
- Examples:
  - `http://localhost:8888/user-service/dev/main`
  - `http://localhost:8888/order-service/prod/main`

### Dynamic Config Refresh with @RefreshScope
```java
@RestController
@RefreshScope   // WHY? Without this, @Value fields are injected once at startup and never change.
                // With @RefreshScope, calling POST /actuator/refresh re-injects the values
                // from Config Server WITHOUT restarting the service.
public class MessageController {

    @Value("${app.welcome-message}")
    private String message;

    @GetMapping("/message")
    public String getMessage() {
        return message;
    }
}
```

**Trigger refresh:** `POST http://localhost:8081/actuator/refresh`
> This tells Spring to re-read config from Config Server and re-create `@RefreshScope` beans with new values.

### Encrypting Sensitive Properties
```yaml
# In config repo
spring:
  datasource:
    password: '{cipher}AQB+...'
```

```yaml
# Config server — encrypt key
encrypt:
  key: my-secret-key
```

**Encrypt/Decrypt endpoints:**
- `POST /encrypt` — encrypt a value
- `POST /decrypt` — decrypt a value

---

## 11. CIRCUIT BREAKER — RESILIENCE4J

### What is a Circuit Breaker?
- Prevents a service from repeatedly calling a **failing downstream service**.
- Three states: **CLOSED** (normal), **OPEN** (blocked), **HALF_OPEN** (testing recovery).

### Circuit Breaker States
```
         success                    failure threshold exceeded
CLOSED ──────────► CLOSED     CLOSED ──────────────────────► OPEN
                                                               │
                              wait duration expires            │
OPEN ◄────────────────────── HALF_OPEN ◄───────────────────────┘
                                │
                    success threshold met
                                │
                                ▼
                              CLOSED
```

### Setup

**Dependency:**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### Annotation-Based Usage
```java
@Service
public class OrderService {

    @Autowired
    private UserClient userClient;

    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    // WHY? If user-service is down, without circuit breaker:
    //   → every request would wait for timeout → threads pile up → ORDER service also goes down!
    // With circuit breaker:
    //   → after X failures, it STOPS calling user-service (OPEN state)
    //   → immediately returns fallback response → protects THIS service from cascading failure
    public UserDto getUser(Long userId) {
        return userClient.getUserById(userId);
    }

    // Fallback method — MUST have: same return type + extra Throwable param
    // WHY? Gives a degraded but usable response instead of throwing error to the client
    public UserDto getUserFallback(Long userId, Throwable throwable) {
        return new UserDto(userId, "Default User", "N/A");
    }
}
```

### Configuration
```yaml
resilience4j:
  circuitbreaker:
    instances:
      userService:                                            # must match @CircuitBreaker(name = "userService")
        register-health-indicator: true                       # expose CB state in /actuator/health
        sliding-window-type: COUNT_BASED                      # track last N calls (vs TIME_BASED = last N seconds)
        sliding-window-size: 10                               # evaluate failure rate over last 10 calls
        failure-rate-threshold: 50                            # OPEN circuit when 50% of last 10 calls failed
        wait-duration-in-open-state: 10s                      # stay OPEN for 10s, then move to HALF_OPEN to test
        permitted-number-of-calls-in-half-open-state: 3       # allow 3 test calls in HALF_OPEN
                                                              # if these succeed → CLOSED; if fail → back to OPEN
        minimum-number-of-calls: 5                            # don't calculate failure rate until at least 5 calls
                                                              # (avoids opening circuit on 1 failure out of 2 calls)
        automatic-transition-from-open-to-half-open-enabled: true  # auto-move to HALF_OPEN after wait duration
```

### Configuration Properties Explained
| Property | Description | Default |
|----------|-------------|---------|
| `sliding-window-type` | `COUNT_BASED` or `TIME_BASED` | `COUNT_BASED` |
| `sliding-window-size` | Number of calls / seconds to track | 100 |
| `failure-rate-threshold` | % failures to trip breaker | 50 |
| `wait-duration-in-open-state` | Duration before moving to HALF_OPEN | 60s |
| `permitted-number-of-calls-in-half-open-state` | Calls allowed in HALF_OPEN | 10 |
| `minimum-number-of-calls` | Min calls before calculating failure rate | 100 |
| `slow-call-rate-threshold` | % slow calls to trip | 100 |
| `slow-call-duration-threshold` | What counts as "slow" | 60s |

### Monitoring Circuit Breaker State
```yaml
management:
  health:
    circuitbreakers:
      enabled: true                  # include circuit breaker status in /actuator/health
  endpoints:
    web:
      exposure:
        include: health              # expose health endpoint over HTTP
  endpoint:
    health:
      show-details: always           # show full health details (UP/DOWN per component)
```

---

## 12. RATE LIMITER, BULKHEAD & RETRY — RESILIENCE4J

### Retry
```java
@Retry(name = "userService", fallbackMethod = "getUserFallback")
public UserDto getUser(Long userId) {
    return userClient.getUserById(userId);
}
```

```yaml
resilience4j:
  retry:
    instances:
      userService:                                  # must match @Retry(name = "userService")
        max-attempts: 3                              # try up to 3 times before giving up
        wait-duration: 2s                            # wait 2 seconds between retries
        retry-exceptions:                            # ONLY retry on these specific exceptions
          - java.io.IOException                      # network I/O failure
          - java.net.SocketTimeoutException           # connection timeout
```

### Rate Limiter
```java
@RateLimiter(name = "userService", fallbackMethod = "getUserFallback")
public UserDto getUser(Long userId) {
    return userClient.getUserById(userId);
}
```

```yaml
resilience4j:
  ratelimiter:
    instances:
      userService:                                   # must match @RateLimiter(name = "userService")
        limit-for-period: 10                          # max 10 calls allowed per period
        limit-refresh-period: 1s                      # period resets every 1 second
        timeout-duration: 3s                           # wait up to 3s for permission if limit exceeded
```

### Bulkhead (Concurrency Limiter)
```java
@Bulkhead(name = "userService", fallbackMethod = "getUserFallback")
public UserDto getUser(Long userId) {
    return userClient.getUserById(userId);
}
```

```yaml
resilience4j:
  bulkhead:
    instances:
      userService:                                   # must match @Bulkhead(name = "userService")
        max-concurrent-calls: 10                     # max 10 simultaneous calls allowed
        max-wait-duration: 500ms                     # wait up to 500ms if all 10 slots are busy
```

### Combining Resilience Patterns
```java
@CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
@Retry(name = "userService")
@RateLimiter(name = "userService")
@Bulkhead(name = "userService")
public UserDto getUser(Long userId) {
    return userClient.getUserById(userId);
}
```

**Execution order:** `Retry → CircuitBreaker → RateLimiter → Bulkhead → Method`

---

## 13. DISTRIBUTED TRACING — MICROMETER TRACING & ZIPKIN

### Why Distributed Tracing?
- A single request traverses **multiple services**.
- Need to track the **entire request flow** across services.
- Identify **latency bottlenecks** and **failures**.

### Key Concepts
| Concept | Description |
|---------|-------------|
| **Trace** | End-to-end journey of a request (unique Trace ID) |
| **Span** | A single unit of work within a trace |
| **Trace ID** | Unique ID shared across all services for one request |
| **Span ID** | Unique ID for each hop/service |
| **Parent Span** | The calling span |

### Setup (Spring Boot 3.x)

**Dependencies:**
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

**Configuration:**
```yaml
management:
  tracing:
    sampling:
      probability: 1.0                  # 1.0 = trace 100% of requests; use 0.1 (10%) in prod to reduce overhead
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans   # where to send trace spans — Zipkin collector URL

logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"   # log pattern includes traceId & spanId
                                                                             # so logs can be correlated across services
```

### Running Zipkin
```bash
docker run -d -p 9411:9411 openzipkin/zipkin
```

**Zipkin UI:** `http://localhost:9411`

### Log Output with Trace Context
```
INFO [order-service,65f8a1b2c3d4e5f6,a1b2c3d4e5f6a7b8] - Processing order 123
INFO [user-service,65f8a1b2c3d4e5f6,c3d4e5f6a7b8c9d0] - Fetching user 456
```
- Same **Trace ID** `65f8a1b2c3d4e5f6` across both services.
- Different **Span IDs** for each service.

### Propagation
- Trace context is **automatically propagated** via HTTP headers (`traceparent`, `b3`).
- Works with `RestTemplate`, `WebClient`, `Feign`, `Spring Cloud Gateway`, `Kafka`, `RabbitMQ`.
- **WHY automatic?** Micrometer instruments these clients/listeners automatically. When Order Service calls User Service via RestTemplate, the Trace ID header is added to the outgoing HTTP request, so User Service continues the same trace — no manual code needed.

---

## 14. CENTRALIZED LOGGING — ELK STACK

### What is ELK?
| Component | Role |
|-----------|------|
| **Elasticsearch** | Search & analytics engine (stores logs) |
| **Logstash** | Log ingestion & transformation pipeline |
| **Kibana** | Visualization dashboard |
| **Filebeat** (optional) | Lightweight log shipper |

### Architecture
```
Microservice ──► Logstash ──► Elasticsearch ──► Kibana
     │                                            │
     └── JSON logs ──► Filebeat ──────────────────┘
```

### JSON Logging Configuration
**Dependency:**
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

**logback-spring.xml:**
```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>localhost:5044</destination>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="LOGSTASH"/>
    </root>
</configuration>
```

### Docker Compose for ELK
```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node          # run as single node (no cluster) — for dev/testing
      - xpack.security.enabled=false        # disable security (auth) — only for dev; enable in prod
    ports:
      - "9200:9200"                         # Elasticsearch REST API port

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    ports:
      - "5044:5044"                         # Logstash input port — services/Filebeat send logs here
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf   # pipeline config — defines input, filters, output

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    ports:
      - "5601:5601"                         # Kibana web UI — access at http://localhost:5601
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200   # tell Kibana where Elasticsearch is
```

### Alternative: Loki + Grafana
- **Grafana Loki** = lightweight, cost-effective alternative to ELK.
- Uses **labels** instead of full-text indexing.
- Often paired with **Promtail** (log shipper).

---

## 15. MESSAGE-DRIVEN MICROSERVICES — SPRING CLOUD STREAM

### What is Spring Cloud Stream?
- A framework for building **event-driven microservices** connected via **message brokers**.
- Provides **binder** abstraction — switch between **Kafka**, **RabbitMQ**, etc. without code changes.
- Uses **functional programming model** (Spring Cloud Function).

### Core Concepts
| Concept | Description |
|---------|-------------|
| **Binder** | Connector to the message broker (Kafka, RabbitMQ) |
| **Binding** | Bridge between app code and broker (input/output) |
| **Supplier** | Produces messages (`Supplier<T>`) |
| **Function** | Processes messages (`Function<T, R>`) |
| **Consumer** | Consumes messages (`Consumer<T>`) |

### Setup (Kafka Binder)
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
</dependency>
```

### Producer (Supplier)
```java
@Configuration
public class OrderEventProducer {

    @Bean
    public Supplier<OrderEvent> orderCreated() {
        return () -> new OrderEvent("ORD-123", "CREATED", LocalDateTime.now());
    }
}
```

### Consumer
```java
@Configuration
public class OrderEventConsumer {

    @Bean
    public Consumer<OrderEvent> handleOrderCreated() {
        return event -> {
            log.info("Received order event: {}", event);
            // process event
        };
    }
}
```

### Processor (Function)
```java
@Bean
public Function<OrderEvent, NotificationEvent> processOrder() {
    return orderEvent -> {
        log.info("Processing: {}", orderEvent);
        return new NotificationEvent(orderEvent.getOrderId(), "Order confirmed");
    };
}
```

### Configuration
```yaml
spring:
  cloud:
    stream:
      bindings:
        orderCreated-out-0:                          # output binding — <functionName>-out-<index>
          destination: order-events                  # Kafka topic / RabbitMQ exchange name to publish to
        handleOrderCreated-in-0:                     # input binding — <functionName>-in-<index>
          destination: order-events                  # same topic/exchange — consumer reads from here
          group: notification-service                # consumer group — ensures only one instance in the group
                                                     # processes each message (prevents duplicate processing)
      function:
        definition: handleOrderCreated               # which @Bean functions to activate as stream listeners
    kafka:
      binder:
        brokers: localhost:9092                      # Kafka broker address
```

### Naming Convention
| Type | Pattern | Example |
|------|---------|---------|
| Input | `<functionName>-in-<index>` | `handleOrderCreated-in-0` |
| Output | `<functionName>-out-<index>` | `orderCreated-out-0` |

---

## 16. EVENT-DRIVEN ARCHITECTURE WITH KAFKA

### Apache Kafka Basics
| Concept | Description |
|---------|-------------|
| **Broker** | Kafka server instance |
| **Topic** | Named feed/channel of messages |
| **Partition** | Sub-division of a topic for parallelism |
| **Producer** | Publishes messages to topics |
| **Consumer** | Reads messages from topics |
| **Consumer Group** | Group of consumers sharing workload |
| **Offset** | Position of a message in a partition |
| **Zookeeper/KRaft** | Cluster coordination (KRaft replacing Zookeeper) |

### Spring Boot + Kafka Setup

**Dependency:**
```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

**Configuration:**
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092                    # Kafka broker address(es) — comma-separated for cluster
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer      # serialize message key as String
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer  # serialize message value as JSON
    consumer:
      group-id: order-service-group                     # consumer group ID — messages are distributed among group members
      auto-offset-reset: earliest                       # where to start reading if no committed offset exists
                                                        # earliest = from beginning; latest = only new messages
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer    # deserialize message key from String
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer  # deserialize message value from JSON
      properties:
        spring.json.trusted.packages: "*"               # trust all packages for JSON deserialization
                                                        # in prod, restrict to specific packages for security
```

### Producer
```java
@Service
public class OrderEventProducer {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderEvent(OrderEvent event) {
        kafkaTemplate.send("order-events", event.getOrderId(), event);
    }
}
```

### Consumer
```java
@Service
public class OrderEventConsumer {

    @KafkaListener(topics = "order-events", groupId = "notification-group")
    public void handleOrderEvent(OrderEvent event) {
        log.info("Received: {}", event);
        // process event
    }
}
```

### Running Kafka (Docker Compose)
```yaml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181              # port ZooKeeper listens on for client connections

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"                            # Kafka broker port — producers/consumers connect here
    environment:
      KAFKA_BROKER_ID: 1                       # unique ID for this broker in the cluster
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181  # where to find ZooKeeper (for cluster coordination)
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092   # address clients use to connect to this broker
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1 # replication factor for consumer offsets topic
                                                # use 1 for dev (single broker); 3+ in production
```

---

## 17. EVENT-DRIVEN ARCHITECTURE WITH RABBITMQ

### RabbitMQ Basics
| Concept | Description |
|---------|-------------|
| **Exchange** | Receives messages and routes to queues |
| **Queue** | Buffer that stores messages |
| **Binding** | Rule linking exchange to queue |
| **Routing Key** | Key used for routing decisions |
| **Exchange Types** | Direct, Topic, Fanout, Headers |

### Spring Boot + RabbitMQ Setup

**Dependency:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

**Configuration:**
```yaml
spring:
  rabbitmq:
    host: localhost           # RabbitMQ server hostname
    port: 5672                # AMQP protocol port (default) — management UI is on 15672
    username: guest            # default RabbitMQ username — change in production
    password: guest            # default RabbitMQ password — change in production
```

### Exchange, Queue & Binding Configuration
```java
@Configuration
public class RabbitMQConfig {

    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order-exchange");
    }

    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable("order-queue").build();
    }

    @Bean
    public Binding orderBinding(Queue orderQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderQueue)
            .to(orderExchange)
            .with("order.created");
    }
}
```

### Producer
```java
@Service
public class OrderEventProducer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void publishOrderEvent(OrderEvent event) {
        rabbitTemplate.convertAndSend("order-exchange", "order.created", event);
    }
}
```

### Consumer
```java
@Service
public class OrderEventConsumer {

    @RabbitListener(queues = "order-queue")
    public void handleOrderEvent(OrderEvent event) {
        log.info("Received: {}", event);
    }
}
```

### Kafka vs RabbitMQ
| Feature | Kafka | RabbitMQ |
|---------|-------|----------|
| Model | Distributed log | Message broker |
| Throughput | Very high (millions/sec) | Moderate (thousands/sec) |
| Message Retention | Retains messages | Deletes after consumption |
| Ordering | Per partition | Per queue |
| Use Case | Event streaming, big data | Task queues, request-reply |
| Protocol | Custom binary | AMQP |
| Replay | Yes (offset-based) | No (once consumed, gone) |

---

## 18. SAGA PATTERN — MANAGING DISTRIBUTED TRANSACTIONS

### The Problem
- In microservices, a single business operation may span **multiple services**.
- Traditional **ACID transactions** don't work across service boundaries.
- Need a way to maintain **data consistency** without distributed transactions.

### What is the Saga Pattern?
- A **saga** = a sequence of local transactions, where each service performs its transaction and publishes an event to trigger the next step.
- If a step fails, **compensating transactions** are executed to undo previous steps.

### Saga Types
| Type | Description | Coordination |
|------|-------------|-------------|
| **Choreography** | Each service listens for events and reacts | Decentralized |
| **Orchestration** | A central orchestrator tells services what to do | Centralized |

### Choreography-Based Saga
```
Order Service ──(OrderCreated)──► Payment Service
                                      │
                                 (PaymentCompleted)
                                      │
                                      ▼
                                 Inventory Service
                                      │
                                 (InventoryReserved)
                                      │
                                      ▼
                                 Shipping Service

If Payment fails:
Payment Service ──(PaymentFailed)──► Order Service (cancel order)
```

### Orchestration-Based Saga
```
                    ┌──────────────────┐
                    │ Saga Orchestrator│
                    └────────┬─────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                  │
     Step 1: Create    Step 2: Process    Step 3: Reserve
     Order             Payment            Inventory
           │                 │                  │
     Compensate:       Compensate:        Compensate:
     Cancel Order      Refund Payment     Release Stock
```

### Example: Orchestration with Spring
```java
@Service
public class OrderSagaOrchestrator {

    @Autowired private OrderService orderService;
    @Autowired private PaymentClient paymentClient;
    @Autowired private InventoryClient inventoryClient;

    @Transactional
    public OrderResponse createOrder(OrderRequest request) {
        // Step 1: Create Order
        Order order = orderService.createOrder(request);

        try {
            // Step 2: Process Payment
            paymentClient.processPayment(new PaymentRequest(order.getId(), order.getTotal()));
        } catch (Exception e) {
            // Compensate Step 1
            orderService.cancelOrder(order.getId());
            throw new SagaException("Payment failed", e);
        }

        try {
            // Step 3: Reserve Inventory
            inventoryClient.reserveStock(new StockRequest(order.getId(), order.getItems()));
        } catch (Exception e) {
            // Compensate Step 2 & Step 1
            paymentClient.refundPayment(order.getId());
            orderService.cancelOrder(order.getId());
            throw new SagaException("Inventory reservation failed", e);
        }

        return new OrderResponse(order, "SUCCESS");
    }
}
```

---

## 19. CQRS PATTERN

### What is CQRS?
- **Command Query Responsibility Segregation** = separate the **write model** (commands) from the **read model** (queries).
- Write to one database, read from another (optimized for queries).

### Architecture
```
                   ┌────────────────┐
                   │     Client     │
                   └───────┬────────┘
                    Commands│ Queries
              ┌─────────────┼──────────────┐
              ▼                            ▼
     ┌────────────────┐          ┌─────────────────┐
     │ Command Service│          │  Query Service   │
     │  (Write Model) │          │  (Read Model)    │
     └───────┬────────┘          └────────┬─────────┘
             │                            │
     ┌───────▼────────┐          ┌────────▼─────────┐
     │   Write DB     │──Events─►│    Read DB       │
     │ (Normalized)   │          │ (Denormalized)   │
     └────────────────┘          └──────────────────┘
```

### When to Use CQRS
- **Read-heavy** applications (e.g., e-commerce product catalog).
- Different scaling needs for reads vs writes.
- Complex domain with different read/write models.
- Often combined with **Event Sourcing**.

---

## 20. SECURITY IN MICROSERVICES — OAUTH2 & JWT

### Security Challenges in Microservices
| Challenge | Solution |
|-----------|----------|
| Authentication across services | Centralized auth (OAuth2/OIDC) |
| Token propagation | JWT tokens passed via headers |
| Service-to-service auth | Client credentials grant / mTLS |
| API Gateway security | Token validation at gateway level |
| Secret management | Vault, sealed secrets |

### OAuth2 Flows for Microservices
| Flow | Use Case |
|------|----------|
| **Authorization Code + PKCE** | Web/mobile apps (user-facing) |
| **Client Credentials** | Service-to-service communication |
| **Resource Owner Password** | Legacy (not recommended) |

---

### Complete OAuth2 Architecture with Keycloak/Okta — Who Does What?

> **Key Insight:** In microservices, authentication and authorization are **split across components**. No single service handles everything. Understanding which component does what is critical.

#### Component Roles — Clear Responsibility Matrix

| Component | Role | Responsibilities | Does NOT Do |
|-----------|------|-------------------|-------------|
| **Auth Server (Keycloak/Okta)** | **Identity Provider (IdP)** | • Manages users, roles, groups, and client registrations<br>• Issues Access Tokens (JWT) and Refresh Tokens<br>• Handles login page (hosted login)<br>• Validates user credentials (username/password)<br>• Provides OIDC discovery endpoints (`.well-known/openid-configuration`)<br>• Publishes JWK (JSON Web Key) public keys for token signature verification | • Does NOT route requests<br>• Does NOT know about your microservices' business logic |
| **API Gateway** | **Authentication (AuthN) — Gatekeeper** | • Validates the JWT token on EVERY incoming request (signature, expiry, issuer)<br>• Rejects requests with missing/invalid/expired tokens (returns 401)<br>• Downloads and caches JWK public keys from Auth Server at startup<br>• Whitelists public endpoints (login, register, health) — skips auth for these<br>• Forwards the validated JWT token to downstream services via `Authorization` header<br>• Optionally extracts claims (userId, email) and passes them as custom headers | • Does NOT generate or issue tokens<br>• Does NOT check roles/permissions (that's authorization, handled by services)<br>• Does NOT store user data |
| **Individual Microservices** | **Authorization (AuthZ) — Resource Server** | • Extracts roles/scopes/claims from the JWT token<br>• Enforces fine-grained access control (e.g., only ADMIN can delete, only OWNER can update)<br>• Uses `@PreAuthorize`, `@Secured`, or `SecurityFilterChain` for role-based access<br>• Optionally re-validates the token (defense-in-depth) using the same JWK keys<br>• Handles business-level authorization (e.g., "user can only see their own orders") | • Does NOT authenticate users<br>• Does NOT issue or refresh tokens<br>• Does NOT handle login |

#### Complete OAuth2 Flow — Step-by-Step (With Keycloak/Okta)

```
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │                    COMPLETE OAuth2 + JWT FLOW IN MICROSERVICES                   │
  │                        (Using Keycloak / Okta as Auth Server)                    │
  └──────────────────────────────────────────────────────────────────────────────────┘

  ┌────────┐         ┌─────────────────┐       ┌───────────────────┐     ┌───────────────┐
  │ Client │         │   API GATEWAY   │       │   AUTH SERVER     │     │ MICROSERVICES │
  │(Browser│         │   (:8080)       │       │  (Keycloak/Okta)  │     │ (User, Order, │
  │/Mobile)│         │                 │       │   (:9090)         │     │  Product)     │
  └───┬────┘         └───────┬─────────┘       └────────┬──────────┘     └──────┬────────┘
      │                      │                          │                       │
      │ ─── PHASE 1: GET TOKEN (Authentication) ─────────────────────────────── │
      │                      │                          │                       │
      │  1. Login Request    │                          │                       │
      │  POST /api/auth/login│                          │                       │
      │  {username, password}│                          │                       │
      │─────────────────────►│                          │                       │
      │                      │                          │                       │
      │                      │  2. Gateway sees /api/auth/login                │
      │                      │     is WHITELISTED →     │                       │
      │                      │     forwards WITHOUT     │                       │
      │                      │     token validation     │                       │
      │                      │─────────────────────────►│                       │
      │                      │                          │                       │
      │                      │                          │  3. Auth Server       │
      │                      │                          │     validates          │
      │                      │                          │     credentials,       │
      │                      │                          │     generates JWT      │
      │                      │                          │     with claims:       │
      │                      │                          │     {sub, roles,       │
      │                      │                          │      email, exp, iss}  │
      │                      │                          │                       │
      │                      │  4. Returns JWT tokens   │                       │
      │                      │◄─────────────────────────│                       │
      │                      │  {access_token,          │                       │
      │                      │   refresh_token}         │                       │
      │  5. Returns tokens   │                          │                       │
      │◄─────────────────────│                          │                       │
      │                      │                          │                       │
      │ ─── PHASE 2: ACCESS PROTECTED RESOURCE (AuthN + AuthZ) ──────────────── │
      │                      │                          │                       │
      │  6. Request with JWT │                          │                       │
      │  GET /api/orders     │                          │                       │
      │  Authorization:      │                          │                       │
      │    Bearer <JWT>      │                          │                       │
      │─────────────────────►│                          │                       │
      │                      │                          │                       │
      │                      │  7. API Gateway:                                 │
      │                      │     a) Extracts token from Authorization header  │
      │                      │     b) Verifies JWT signature using              │
      │                      │        cached JWK public keys                    │
      │                      │        (downloaded from Auth Server at startup)  │
      │                      │     c) Checks token is NOT expired               │
      │                      │     d) Validates issuer (iss) claim              │
      │                      │     ✅ Token valid → forward request             │
      │                      │     ❌ Token invalid → return 401 immediately    │
      │                      │                          │                       │
      │                      │  8. Forwards request with│original JWT           │
      │                      │────────────────────────────────────────────────► │
      │                      │     Headers:             │                       │
      │                      │     Authorization: Bearer <JWT>                  │
      │                      │     (optionally)         │                       │
      │                      │     X-User-Id: 12345     │                       │
      │                      │     X-User-Roles: ADMIN  │                       │
      │                      │                          │                       │
      │                      │                          │   9. Microservice:    │
      │                      │                          │      a) Extracts JWT  │
      │                      │                          │      b) Reads roles/  │
      │                      │                          │         scopes from   │
      │                      │                          │         claims        │
      │                      │                          │      c) Checks:       │
      │                      │                          │         Does user     │
      │                      │                          │         have required │
      │                      │                          │         role?         │
      │                      │                          │         e.g., ADMIN   │
      │                      │                          │         for DELETE    │
      │                      │                          │      d) Business      │
      │                      │                          │         logic check:  │
      │                      │                          │         Can this user │
      │                      │                          │         access THIS   │
      │                      │                          │         specific      │
      │                      │                          │         resource?     │
      │                      │                          │      ✅ Authorized    │
      │                      │                          │      → process        │
      │                      │                          │      ❌ Forbidden     │
      │                      │                          │      → return 403     │
      │                      │                          │                       │
      │  10. Response        │                          │                       │
      │◄─────────────────────│◄────────────────────────────────────────────────│
      │                      │                          │                       │
      │ ─── PHASE 3: SERVICE-TO-SERVICE CALL ────────────────────────────────── │
      │                      │                          │                       │
      │                      │                          │  11. Order Service    │
      │                      │                          │      needs to call    │
      │                      │                          │      User Service:    │
      │                      │                          │      Option A:        │
      │                      │                          │        Propagate      │
      │                      │                          │        user's JWT     │
      │                      │                          │        (via Feign     │
      │                      │                          │        interceptor)   │
      │                      │                          │      Option B:        │
      │                      │                          │        Use Client     │
      │                      │                          │        Credentials    │
      │                      │                          │        grant to get   │
      │                      │                          │        service-level  │
      │                      │                          │        token from     │
      │                      │                          │        Auth Server    │
      │                      │                          │                       │
```

> **IMPORTANT: The API Gateway does NOT contact the Auth Server on every request.**
> At startup, the Gateway downloads the Auth Server's **JWK (JSON Web Key) public keys** and caches them.
> It uses these cached keys to verify JWT signatures locally — making token validation fast and offline.
> The keys are refreshed periodically or when an unknown key ID (`kid`) is encountered.

#### How JWK Caching Works
```
STARTUP:
  API Gateway ──GET──► Auth Server (/.well-known/openid-configuration)
                       └──► fetches jwk-set-uri
  API Gateway ──GET──► Auth Server (/protocol/openid-connect/certs)
                       └──► downloads public keys (JWK Set)
                       └──► CACHES them in memory

EVERY REQUEST:
  API Gateway:
    1. Extract JWT from request
    2. Read "kid" (Key ID) from JWT header
    3. Find matching public key from CACHED JWK Set
    4. Verify signature locally (NO network call to Auth Server)
    5. Check expiry, issuer, audience — all local
```

#### Summary: Authentication vs Authorization in Microservices

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                         │
  │   Auth Server (Keycloak/Okta):  WHO ARE YOU?                           │
  │     → Issues tokens, manages users                                      │
  │     → Only involved during login and token refresh                      │
  │                                                                         │
  │   API Gateway:  ARE YOU ALLOWED IN?                                     │
  │     → Authentication = verifies token is valid                          │
  │     → Guards the front door (rejects 401 for bad/missing tokens)        │
  │     → Does NOT check what you can do inside                             │
  │                                                                         │
  │   Individual Services:  WHAT CAN YOU DO?                                │
  │     → Authorization = checks roles, scopes, ownership                   │
  │     → e.g., "Only ADMIN can delete" / "Only owner can edit own order"   │
  │     → Uses claims from JWT: roles, scopes, sub (user ID)                │
  │                                                                         │
  └─────────────────────────────────────────────────────────────────────────┘
```

---

### Resource Server Configuration (Individual Microservices — Authorization)

> **This config goes in EACH microservice** (User Service, Order Service, etc.) — NOT in the API Gateway.
> It makes the service a **Resource Server** that can read JWT claims and enforce role-based access.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

```yaml
# application.yml — for EACH microservice (e.g., order-service)
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/my-realm             # auto-discovers JWK set, issuer validation
                                                                        # Spring fetches /.well-known/openid-configuration
          # OR
          jwk-set-uri: http://localhost:8080/realms/my-realm/protocol/openid-connect/certs  # direct URL to public keys
                                                                                             # used to verify JWT signatures
```

```java
// SecurityConfig.java — in EACH microservice
// This handles AUTHORIZATION (role checks) — the service reads roles from the JWT
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()           // public endpoints — no auth needed
                .requestMatchers("/api/admin/**").hasRole("ADMIN")      // only users with ADMIN role
                .requestMatchers(HttpMethod.DELETE, "/api/orders/**")
                    .hasAuthority("SCOPE_order:delete")                  // scope-based access
                .anyRequest().authenticated()                            // all other endpoints — just need valid token
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())                          // tells Spring to decode JWT and map claims
            );
        return http.build();
    }
}
```

#### Method-Level Authorization in Controllers (Fine-Grained)
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping
    @PreAuthorize("hasRole('USER') or hasRole('ADMIN')")   // role-based — from JWT "realm_access.roles"
    public List<Order> getOrders() { ... }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")                       // only ADMIN can delete
    public void deleteOrder(@PathVariable Long id) { ... }

    @GetMapping("/my-orders")
    @PreAuthorize("#userId == authentication.principal.subject")  // ownership check — user can only see own orders
    public List<Order> getMyOrders(@RequestParam String userId) { ... }
}
```

> **Enable method-level security** by adding `@EnableMethodSecurity` to your config class.

### Propagating JWT in Feign Calls
```java
@Component
public class FeignClientInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        ServletRequestAttributes attrs =
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (attrs != null) {
            String token = attrs.getRequest().getHeader("Authorization");
            if (token != null) {
                template.header("Authorization", token);
            }
        }
    }
}
```

### Service-to-Service Auth (Client Credentials)
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          internal-service:                                  # registration ID — logical name for this client
            provider: keycloak                               # which provider config to use (defined below)
            client-id: order-service                         # OAuth2 client ID — identifies this service
            client-secret: ${ORDER_SERVICE_SECRET}           # client secret — injected from env var (never hardcode)
            authorization-grant-type: client_credentials     # grant type — machine-to-machine auth (no user involved)
            scope: openid                                    # requested scopes — openid for OIDC identity
        provider:
          keycloak:                                          # provider config — reusable across registrations
            token-uri: http://localhost:8080/realms/my-realm/protocol/openid-connect/token  # endpoint to get access token
```

---

## 21. SPRING CLOUD BUS

### What is Spring Cloud Bus?
- Broadcasts **configuration changes** to all connected services using a **message broker** (RabbitMQ / Kafka).
- Instead of calling `/actuator/refresh` on every service, call **once** and Spring Cloud Bus propagates.

### How It Works
```
POST /actuator/busrefresh (on any service)
        │
        ▼
   Message Broker (RabbitMQ/Kafka)
        │
   ┌────┼────┬────┐
   ▼    ▼    ▼    ▼
 Svc1 Svc2 Svc3 Svc4   ← all refresh their config
```

### Setup
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>  <!-- RabbitMQ -->
</dependency>
<!-- OR -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```

**Trigger:** `POST http://any-service/actuator/busrefresh`

---

## 22. SPRING CLOUD VAULT — SECRETS MANAGEMENT

### Why Vault?
- **Never store secrets** (DB passwords, API keys) in plain text in config files or Git.
- **HashiCorp Vault** = centralized secrets management, encryption as a service, access control.

### Setup
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    vault:
      uri: http://localhost:8200           # Vault server URL
      token: ${VAULT_TOKEN}                # auth token — injected from env var (never hardcode)
      kv:
        enabled: true                      # enable KV (Key-Value) secrets engine
        backend: secret                    # KV engine mount path in Vault
        default-context: user-service      # path under backend — reads from secret/user-service
```

### Storing Secrets in Vault
```bash
vault kv put secret/user-service \
    spring.datasource.username=admin \
    spring.datasource.password=s3cr3t
```

- Spring Boot will **automatically inject** these properties at startup.

---

## 23. CONTAINERIZATION WITH DOCKER

### Dockerfile for Spring Boot
```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Multi-Stage Dockerfile (Optimized)
```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN ./mvnw clean package -DskipTests

# Stage 2: Run
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Spring Boot Layered JAR (Best Practice)
```dockerfile
FROM eclipse-temurin:21-jre-alpine AS builder
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

### Spring Boot Buildpacks (No Dockerfile Needed)
```bash
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=myapp:latest
```

### Essential Docker Commands
| Command | Purpose |
|---------|---------|
| `docker build -t myapp .` | Build image |
| `docker run -p 8080:8080 myapp` | Run container |
| `docker images` | List images |
| `docker ps` | List running containers |
| `docker logs <container>` | View logs |
| `docker stop <container>` | Stop container |
| `docker network create my-net` | Create network |

---

## 24. DOCKER COMPOSE FOR MICROSERVICES

### Complete Docker Compose
```yaml
version: '3.8'

services:
  # --- Infrastructure ---
  eureka-server:
    build: ./eureka-server                 # build from local Dockerfile in ./eureka-server/
    ports:
      - "8761:8761"                        # Eureka dashboard + API

  config-server:
    build: ./config-server
    ports:
      - "8888:8888"                        # Config Server API
    depends_on:
      - eureka-server                      # start Eureka first — config server registers with it
    environment:
      EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE: http://eureka-server:8761/eureka/   # override Eureka URL for Docker network

  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"                        # only publicly exposed port — clients hit this
    depends_on:
      - eureka-server                      # needs Eureka to discover downstream services
      - config-server                      # needs Config Server for its configuration
    environment:
      EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE: http://eureka-server:8761/eureka/

  # --- Business Services ---
  user-service:
    build: ./user-service
    ports:
      - "8081:8081"
    depends_on:
      - eureka-server                      # register with Eureka on startup
      - user-db                            # needs its database to be available
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://user-db:5432/userdb              # DB URL using Docker service name
      EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE: http://eureka-server:8761/eureka/

  order-service:
    build: ./order-service
    ports:
      - "8082:8082"
    depends_on:
      - eureka-server
      - order-db
      - kafka                              # order-service publishes events to Kafka
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://order-db:5432/orderdb
      EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE: http://eureka-server:8761/eureka/
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092                                 # Kafka broker within Docker network

  # --- Databases ---
  user-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: userdb                  # auto-create this database on first run
      POSTGRES_USER: admin                 # DB superuser username
      POSTGRES_PASSWORD: ${USER_DB_PASSWORD}   # from .env file — never hardcode passwords
    volumes:
      - user-db-data:/var/lib/postgresql/data   # persist DB data across container restarts

  order-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orderdb
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ${ORDER_DB_PASSWORD}
    volumes:
      - order-db-data:/var/lib/postgresql/data

  # --- Messaging ---
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1                                     # unique broker identifier
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181                # ZooKeeper coordination address
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092     # how other containers find this broker
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1              # 1 for single-broker dev; 3+ in prod
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181          # ZooKeeper client connection port

  # --- Observability ---
  zipkin:
    image: openzipkin/zipkin
    ports:
      - "9411:9411"                        # Zipkin UI + trace collection API

volumes:
  user-db-data:                            # named volume for user DB persistence
  order-db-data:                           # named volume for order DB persistence
```

### Running
```bash
docker-compose up -d            # start all
docker-compose logs -f          # follow logs
docker-compose down             # stop all
docker-compose up -d --build    # rebuild and start
```

---

## 25. ORCHESTRATION WITH KUBERNETES

### Kubernetes Basics
| Concept | Description |
|---------|-------------|
| **Cluster** | Set of machines (nodes) running containers |
| **Node** | A single machine in the cluster |
| **Pod** | Smallest deployable unit (1+ containers) |
| **Deployment** | Manages pod replicas, rolling updates |
| **Service** | Stable network endpoint for pods |
| **Ingress** | External HTTP routing (like API gateway) |
| **ConfigMap** | External configuration (non-sensitive) |
| **Secret** | External configuration (sensitive) |
| **Namespace** | Logical isolation within a cluster |
| **HPA** | Horizontal Pod Autoscaler |

### Kubernetes Architecture
```
┌──────────────────────────────────────────────┐
│                KUBERNETES CLUSTER            │
│  ┌────────────────────────────────────────┐  │
│  │          Control Plane                 │  │
│  │  API Server │ Scheduler │ etcd │ CM   │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌──────────────┐     ┌──────────────┐       │
│  │   Node 1     │     │   Node 2     │       │
│  │ ┌──────────┐ │     │ ┌──────────┐ │       │
│  │ │  Pod     │ │     │ │  Pod     │ │       │
│  │ │(user-svc)│ │     │ │(order-svc│ │       │
│  │ └──────────┘ │     │ └──────────┘ │       │
│  │ ┌──────────┐ │     │ ┌──────────┐ │       │
│  │ │  Pod     │ │     │ │  Pod     │ │       │
│  │ │(user-svc)│ │     │ │(product) │ │       │
│  │ └──────────┘ │     │ └──────────┘ │       │
│  └──────────────┘     └──────────────┘       │
└──────────────────────────────────────────────┘
```

---

## 26. DEPLOYING SPRING BOOT ON KUBERNETES

### Deployment Manifest
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: microservices
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: myregistry/user-service:1.0.0
          ports:
            - containerPort: 8081
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "k8s"
            - name: SPRING_DATASOURCE_URL
              valueFrom:
                configMapKeyRef:
                  name: user-service-config
                  key: db-url
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: user-service-secrets
                  key: db-password
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8081
            initialDelaySeconds: 60
            periodSeconds: 15
```

### Service Manifest
```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: microservices
spec:
  selector:
    app: user-service
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
  type: ClusterIP
```

### ConfigMap & Secret
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
  namespace: microservices
data:
  db-url: "jdbc:postgresql://user-db:5432/userdb"

---
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secrets
  namespace: microservices
type: Opaque
data:
  db-password: czNjcjN0     # base64 encoded
```

### Ingress (External Access)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: microservices
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /users(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: user-service
                port:
                  number: 8081
          - path: /orders(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: order-service
                port:
                  number: 8082
```

### HPA (Auto-Scaling)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### Essential kubectl Commands
| Command | Purpose |
|---------|---------|
| `kubectl apply -f deployment.yaml` | Apply manifest |
| `kubectl get pods -n microservices` | List pods |
| `kubectl get svc -n microservices` | List services |
| `kubectl logs <pod-name>` | View logs |
| `kubectl describe pod <pod-name>` | Pod details |
| `kubectl scale deployment user-service --replicas=5` | Scale manually |
| `kubectl rollout status deployment/user-service` | Check rollout |
| `kubectl rollout undo deployment/user-service` | Rollback |

---

## 27. HELM CHARTS FOR MICROSERVICES

### What is Helm?
- **Helm** = package manager for Kubernetes.
- **Chart** = a collection of K8s manifests as a reusable template.
- Supports **versioning**, **rollbacks**, **values overrides**.

### Chart Structure
```
my-service/
├── Chart.yaml           # chart metadata
├── values.yaml          # default config values
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── ingress.yaml
│   └── hpa.yaml
└── charts/              # sub-chart dependencies
```

### Templated Deployment
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

### values.yaml
```yaml
replicaCount: 2                        # number of pod replicas to run
image:
  repository: myregistry/user-service  # Docker image name (registry/image)
  tag: "1.0.0"                         # image version tag
service:
  type: ClusterIP                      # service type — ClusterIP = internal only;
                                       # use LoadBalancer or NodePort for external access
  port: 8081                           # port the K8s Service listens on
```

### Helm Commands
| Command | Purpose |
|---------|---------|
| `helm install user-svc ./my-service` | Install chart |
| `helm upgrade user-svc ./my-service` | Upgrade release |
| `helm rollback user-svc 1` | Rollback to revision 1 |
| `helm uninstall user-svc` | Remove release |
| `helm list` | List releases |

---

## 28. SERVICE MESH — ISTIO

### What is a Service Mesh?
- **Infrastructure layer** that handles service-to-service communication.
- Provides **traffic management**, **security (mTLS)**, **observability** without changing application code.

### Istio Architecture
```
┌─────────────────────────────────────┐
│          Control Plane (Istiod)     │
│  Pilot │ Citadel │ Galley          │
└─────────────────┬───────────────────┘
                  │ config push
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌────────┐  ┌────────┐   ┌────────┐
│ Pod    │  │ Pod    │   │ Pod    │
│┌──────┐│  │┌──────┐│   │┌──────┐│
││Envoy ││  ││Envoy ││   ││Envoy ││  ← Sidecar proxies
│├──────┤│  │├──────┤│   │├──────┤│
││ App  ││  ││ App  ││   ││ App  ││
│└──────┘│  │└──────┘│   │└──────┘│
└────────┘  └────────┘   └────────┘
```

### Key Features
| Feature | Description |
|---------|-------------|
| **mTLS** | Automatic mutual TLS between services |
| **Traffic Splitting** | Canary / blue-green deployments |
| **Circuit Breaking** | At network level |
| **Rate Limiting** | Per-service rate limits |
| **Observability** | Metrics, tracing, logging |
| **Retries / Timeouts** | Configurable per route |

### When to Use a Service Mesh
- Large number of microservices (50+).
- Need for **zero-trust security** (mTLS everywhere).
- Complex traffic routing (canary deployments, A/B testing).
- Centralized observability without instrumenting each service.

---

## 29. HEALTH CHECKS — READINESS / LIVENESS PROBES

### Types of Probes
| Probe | Purpose | When |
|-------|---------|------|
| **Liveness** | Is the app alive? Restart if not. | App is stuck / deadlocked |
| **Readiness** | Is the app ready to serve traffic? | App is starting / warming up |
| **Startup** | Has the app finished starting? | Slow-starting apps |

### Spring Boot Actuator Health Groups
```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true                                  # enables /health/liveness and /health/readiness endpoints
                                                       # (used by K8s probes to check if app is alive/ready)
      group:
        readiness:                                     # readiness group — "is the app ready to serve traffic?"
          include: db, diskSpace                       # check DB connection + disk space for readiness
        liveness:                                      # liveness group — "is the app alive and not stuck?"
          include: ping                                # simple ping check — lightweight, just checks app responds
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus     # which actuator endpoints to expose over HTTP
```

### Kubernetes Probe Configuration
```yaml
readinessProbe:                          # K8s checks this to decide if pod should receive traffic
  httpGet:
    path: /actuator/health/readiness     # Spring Boot readiness health endpoint
    port: 8080
  initialDelaySeconds: 20                # wait 20s after container starts before first check
  periodSeconds: 10                      # check every 10 seconds
  failureThreshold: 3                    # mark as NOT ready after 3 consecutive failures

livenessProbe:                           # K8s checks this to decide if pod should be RESTARTED
  httpGet:
    path: /actuator/health/liveness      # Spring Boot liveness health endpoint
    port: 8080
  initialDelaySeconds: 60                # wait 60s — gives app time to fully start
  periodSeconds: 15                      # check every 15 seconds
  failureThreshold: 3                    # RESTART pod after 3 consecutive failures

startupProbe:                            # K8s checks this FIRST — before liveness/readiness
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 10                # wait 10s before first check
  periodSeconds: 5                       # check every 5 seconds
  failureThreshold: 30                   # 30 * 5 = 150s max time for app to start
                                         # if app doesn't start in 150s, pod is killed
```

---

## 30. SPRING BOOT ACTUATOR IN MICROSERVICES

### Essential Actuator Endpoints for Microservices
| Endpoint | Purpose |
|----------|---------|
| `/actuator/health` | Health status (liveness, readiness) |
| `/actuator/info` | App info (version, build) |
| `/actuator/metrics` | Application metrics |
| `/actuator/prometheus` | Prometheus-format metrics |
| `/actuator/env` | Environment properties |
| `/actuator/refresh` | Refresh `@RefreshScope` beans |
| `/actuator/busrefresh` | Broadcast config refresh |
| `/actuator/circuitbreakers` | Circuit breaker states |

### Configuration
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus, refresh, circuitbreakers   # expose these actuator endpoints over HTTP
  endpoint:
    health:
      show-details: always             # always show component-level health details (DB, disk, etc.)
  info:
    env:
      enabled: true                    # include env-based info properties in /actuator/info

info:                                  # custom info shown at /actuator/info
  app:
    name: ${spring.application.name}                   # app name from config
    version: '@project.version@'                       # Maven project version (resolved at build time)
    encoding: '@project.build.sourceEncoding@'         # source file encoding (e.g., UTF-8)
```

### Securing Actuator Endpoints
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth
        .requestMatchers("/actuator/health/**").permitAll()
        .requestMatchers("/actuator/info").permitAll()
        .requestMatchers("/actuator/**").hasRole("ADMIN")
        .anyRequest().authenticated()
    );
    return http.build();
}
```

---

## 31. MONITORING — PROMETHEUS & GRAFANA

### Architecture
```
Spring Boot ──(/actuator/prometheus)──► Prometheus ──► Grafana
   │                                     (scrape &      (visualize)
   └── micrometer metrics                 store)
```

### Setup

**Dependency:**
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

**Configuration:**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus, health, metrics    # expose Prometheus scrape endpoint + health + metrics
  metrics:
    tags:
      application: ${spring.application.name}   # add "application" tag to all metrics — lets you filter
                                                # by service name in Prometheus/Grafana queries
```

### Prometheus Configuration (prometheus.yml)
```yaml
global:
  scrape_interval: 15s                          # how often Prometheus scrapes targets for metrics

scrape_configs:
  - job_name: 'user-service'                    # logical name shown in Prometheus UI
    metrics_path: '/actuator/prometheus'         # Spring Boot Actuator's Prometheus endpoint
    static_configs:
      - targets: ['user-service:8081']           # host:port of the service to scrape

  - job_name: 'order-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['order-service:8082']
```

### Docker Compose
```yaml
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"                              # Prometheus web UI + query API
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml   # mount scrape config into container

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"                              # Grafana dashboard UI — access at http://localhost:3000
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin          # default admin password — change in production
```

### Key Metrics to Monitor
| Metric | Description |
|--------|-------------|
| `http_server_requests_seconds` | Request latency |
| `jvm_memory_used_bytes` | JVM memory usage |
| `jvm_threads_live_threads` | Active thread count |
| `process_cpu_usage` | CPU usage |
| `hikaricp_connections_active` | Active DB connections |
| `resilience4j_circuitbreaker_state` | Circuit breaker status |
| `logback_events_total` | Log event counts |

### Custom Metrics
```java
@Service
public class OrderService {

    private final Counter orderCounter;
    private final Timer orderTimer;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.created.total")
            .description("Total orders created")
            .register(registry);
        this.orderTimer = Timer.builder("orders.processing.time")
            .description("Order processing time")
            .register(registry);
    }

    public Order createOrder(OrderRequest request) {
        return orderTimer.record(() -> {
            Order order = processOrder(request);
            orderCounter.increment();
            return order;
        });
    }
}
```

---

## 32. CI/CD PIPELINES FOR MICROSERVICES

### Pipeline Stages
```
Code Push ──► Build ──► Test ──► Docker Build ──► Push Image ──► Deploy to K8s
```

### GitHub Actions Example
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
    paths:
      - 'user-service/**'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build with Maven
        run: cd user-service && mvn clean package -DskipTests

      - name: Run Tests
        run: cd user-service && mvn test

      - name: Build Docker Image
        run: docker build -t myregistry/user-service:${{ github.sha }} ./user-service

      - name: Push to Registry
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker push myregistry/user-service:${{ github.sha }}

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/user-service \
            user-service=myregistry/user-service:${{ github.sha }} \
            --namespace=microservices
```

### Deployment Strategies
| Strategy | Description | Risk |
|----------|-------------|------|
| **Rolling Update** | Gradually replace old pods with new | Low |
| **Blue-Green** | Two environments, switch traffic | Low (costly) |
| **Canary** | Route small % traffic to new version | Very low |
| **Recreate** | Kill all old, start all new | High (downtime) |

---

## 33. DATABASE PER SERVICE PATTERN

### Why Database per Service?
- Each service owns its **data store** — no shared database.
- Enables **independent schema evolution**.
- Different services can use **different DB technologies** (polyglot persistence).

### Approaches to Data Sharing
| Approach | Description | Pros | Cons |
|----------|-------------|------|------|
| **API Calls** | Service queries another service's API | Simple, clean boundary | Latency, coupling |
| **Events** | Publish data changes as events | Decoupled, eventual consistency | Complex |
| **Data Replication** | Copy data to local read model | Fast reads | Stale data, storage |
| **Shared DB (anti-pattern)** | Services share same DB | Simple queries | Tight coupling |

### Example: Polyglot Persistence
```
User Service ──► PostgreSQL (relational data)
Order Service ──► MongoDB (flexible schema)
Search Service ──► Elasticsearch (full-text search)
Session Service ──► Redis (fast key-value)
Analytics Service ──► ClickHouse (columnar analytics)
```

---

## 34. API VERSIONING

### Strategies
| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **URI Versioning** | `/api/v1/users` | Clear, cache-friendly | URL pollution |
| **Header Versioning** | `X-API-Version: 1` | Clean URLs | Hidden, no caching |
| **Query Param** | `/api/users?version=1` | Simple | Messy URLs |
| **Content Negotiation** | `Accept: application/vnd.myapp.v1+json` | RESTful | Complex |

### URI Versioning (Most Common)
```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {

    @GetMapping("/{id}")
    public UserDtoV1 getUser(@PathVariable Long id) { ... }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {

    @GetMapping("/{id}")
    public UserDtoV2 getUser(@PathVariable Long id) { ... }
}
```

### Gateway-Level Versioning
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-v1
          uri: lb://USER-SERVICE-V1
          predicates:
            - Path=/api/v1/users/**
        - id: user-v2
          uri: lb://USER-SERVICE-V2
          predicates:
            - Path=/api/v2/users/**
```

---

## 35. CONTRACT TESTING — SPRING CLOUD CONTRACT

### What is Contract Testing?
- Ensures **consumer** and **provider** agree on the API contract.
- **Provider** generates tests from contracts.
- **Consumer** uses **stubs** generated from contracts.

### Setup (Provider Side)
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-verifier</artifactId>
    <scope>test</scope>
</dependency>
```

### Contract DSL (Groovy)
```groovy
// src/test/resources/contracts/shouldReturnUser.groovy
Contract.make {
    description "should return user by ID"
    request {
        method GET()
        url "/api/users/1"
    }
    response {
        status 200
        headers {
            contentType applicationJson()
        }
        body([
            id: 1,
            name: "John",
            email: "john@example.com"
        ])
    }
}
```

### Consumer Side (Stub Runner)
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@AutoConfigureStubRunner(
    ids = "com.example:user-service:+:stubs:8081",
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class OrderServiceContractTest {

    @Autowired
    private UserClient userClient;

    @Test
    void shouldGetUserFromStub() {
        UserDto user = userClient.getUserById(1L);
        assertThat(user.getName()).isEqualTo("John");
    }
}
```

---

## 36. 12-FACTOR APP METHODOLOGY

### The 12 Factors
| # | Factor | Description | Spring Boot Approach |
|---|--------|-------------|---------------------|
| 1 | **Codebase** | One codebase, many deploys | Git repo per service |
| 2 | **Dependencies** | Explicitly declare & isolate | Maven/Gradle |
| 3 | **Config** | Store in environment | `application.yml`, Config Server, env vars |
| 4 | **Backing Services** | Treat as attached resources | Spring Data, connection URLs |
| 5 | **Build, Release, Run** | Strictly separate stages | CI/CD pipeline |
| 6 | **Processes** | Stateless processes | No session state; use Redis |
| 7 | **Port Binding** | Export services via port | Embedded Tomcat/Netty |
| 8 | **Concurrency** | Scale via process model | Multiple instances, K8s HPA |
| 9 | **Disposability** | Fast startup, graceful shutdown | `server.shutdown=graceful` |
| 10 | **Dev/Prod Parity** | Keep environments similar | Docker, same configs |
| 11 | **Logs** | Treat as event streams | stdout, ELK/Loki |
| 12 | **Admin Processes** | Run admin tasks as one-off | Spring Batch, Flyway migrations |

---

## 37. BEST PRACTICES & PRODUCTION CHECKLIST

### Design Best Practices
- [ ] **One service = one bounded context** (DDD).
- [ ] Design **API-first** — define contracts before coding.
- [ ] Use **asynchronous communication** where possible.
- [ ] Implement **idempotency** for all write operations.
- [ ] Use **correlation IDs** for request tracing.
- [ ] Implement **graceful degradation** (circuit breakers, fallbacks).

### Infrastructure Best Practices
- [ ] Use **service discovery** (Eureka, Consul, or K8s DNS).
- [ ] Centralize **configuration** (Config Server or K8s ConfigMaps).
- [ ] Centralize **logging** (ELK or Loki + Grafana).
- [ ] Enable **distributed tracing** (Zipkin / Jaeger).
- [ ] Set up **health checks** (liveness + readiness probes).
- [ ] Use **container orchestration** (Kubernetes).

### Security Best Practices
- [ ] **Never hardcode secrets** — use Vault or K8s Secrets.
- [ ] Validate **all inputs** at service boundaries.
- [ ] Use **OAuth2 / JWT** for authentication.
- [ ] Enable **mTLS** for service-to-service communication.
- [ ] Implement **rate limiting** at the gateway.
- [ ] Keep dependencies **up to date** (CVE scanning).

### Observability Best Practices
- [ ] **Structured logging** (JSON format).
- [ ] **Metrics** exposed via Prometheus.
- [ ] **Dashboards** in Grafana for every service.
- [ ] **Alerting** rules for critical metrics.
- [ ] Track **SLIs/SLOs** (latency, error rate, throughput).

### Resilience Best Practices
- [ ] Implement **circuit breakers** for all external calls.
- [ ] Configure **retries** with exponential backoff.
- [ ] Set **timeouts** on all HTTP calls.
- [ ] Use **bulkheads** to isolate resource pools.
- [ ] Design for **eventual consistency** (saga pattern).
- [ ] Test with **chaos engineering** (e.g., Chaos Monkey).

---

## 38. IMPORTANT ANNOTATIONS REFERENCE

### Spring Cloud Annotations
| Annotation | Purpose |
|------------|---------|
| `@EnableEurekaServer` | Marks app as Eureka Server |
| `@EnableDiscoveryClient` | Enable service discovery client |
| `@EnableFeignClients` | Enable Feign declarative clients |
| `@FeignClient` | Declare a Feign client interface |
| `@EnableConfigServer` | Marks app as Config Server |
| `@RefreshScope` | Refresh bean on config change |
| `@LoadBalanced` | Enable client-side load balancing |

### Resilience4j Annotations
| Annotation | Purpose |
|------------|---------|
| `@CircuitBreaker` | Apply circuit breaker pattern |
| `@Retry` | Apply retry pattern |
| `@RateLimiter` | Apply rate limiting |
| `@Bulkhead` | Apply bulkhead pattern |
| `@TimeLimiter` | Apply time limiting (async) |

### Kafka / Messaging Annotations
| Annotation | Purpose |
|------------|---------|
| `@KafkaListener` | Listen to Kafka topic |
| `@RabbitListener` | Listen to RabbitMQ queue |
| `@SendTo` | Forward result to another destination |
| `@EnableKafka` | Enable Kafka listener |

### Kubernetes / Cloud Native
| Annotation | Purpose |
|------------|---------|
| `@Scheduled` | Scheduled tasks (cron) |
| `@Async` | Async method execution |
| `@ConditionalOnCloudPlatform` | Conditional on cloud env |

---

## 39. COMMON CONFIGURATION PROPERTIES REFERENCE

### Eureka Client
```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/          # Eureka Server URL to register with and fetch registry from
    register-with-eureka: true                            # register this service instance with Eureka
    fetch-registry: true                                  # download registry of all services (for discovery)
    registry-fetch-interval-seconds: 30                   # how often to refresh the local registry cache (seconds)
  instance:
    prefer-ip-address: true                               # register with IP instead of hostname
    lease-renewal-interval-in-seconds: 30                 # heartbeat interval — how often to tell Eureka "I'm alive"
    lease-expiration-duration-in-seconds: 90              # Eureka removes this instance if no heartbeat for 90s
    instance-id: ${spring.application.name}:${random.value}   # unique instance ID — allows multiple instances of same service
```

### Spring Cloud Gateway
```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true                                   # auto-create routes for every service in Eureka
                                                          # (e.g., /USER-SERVICE/** routes to user-service)
          lower-case-service-id: true                     # use lowercase service names in URLs
                                                          # (e.g., /user-service/** instead of /USER-SERVICE/**)
      default-filters:                                    # filters applied to ALL routes
        - DedupeResponseHeader=Access-Control-Allow-Origin    # remove duplicate CORS headers
      globalcors:                                         # global CORS configuration
        cors-configurations:
          '[/**]':                                        # apply to all paths
            allowedOrigins: "*"                           # allow requests from any origin — restrict in prod
            allowedMethods: "*"                           # allow all HTTP methods (GET, POST, PUT, DELETE, etc.)
```

### Resilience4j
```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:                                            # shared default config — reusable by any instance
        sliding-window-size: 10                           # track last 10 calls for failure rate
        failure-rate-threshold: 50                        # open circuit at 50% failure rate
        wait-duration-in-open-state: 10s                  # wait 10s before testing recovery (HALF_OPEN)
    instances:
      myService:
        base-config: default                              # inherit from "default" config above
  retry:
    configs:
      default:
        max-attempts: 3                                   # retry up to 3 times
        wait-duration: 2s                                 # wait 2s between retries
  ratelimiter:
    configs:
      default:
        limit-for-period: 10                              # allow 10 calls per period
        limit-refresh-period: 1s                          # reset limit every 1 second
```

### Distributed Tracing
```yaml
management:
  tracing:
    sampling:
      probability: 1.0                                    # trace 100% of requests (use 0.1 in prod for 10%)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans           # Zipkin collector endpoint to send trace spans
```

### Spring Cloud Config Client
```yaml
spring:
  config:
    import: configserver:http://config-server:8888        # pull config from Config Server at startup
  cloud:
    config:
      fail-fast: true                                     # fail startup immediately if Config Server is unreachable
                                                          # (instead of starting with missing config)
      retry:                                              # retry connecting to Config Server if it's not ready yet
        max-attempts: 6                                   # try up to 6 times
        initial-interval: 1000                            # first retry after 1000ms (1 second)
        multiplier: 1.5                                   # each subsequent wait = previous * 1.5
                                                          # (1s, 1.5s, 2.25s, 3.375s, ...)
```

### Graceful Shutdown
```yaml
server:
  shutdown: graceful                                      # stop accepting new requests, finish in-flight ones
                                                          # (vs "immediate" which kills everything instantly)

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s                       # max time to wait for in-flight requests to complete
                                                          # after 30s, force-kills remaining requests
```

---
