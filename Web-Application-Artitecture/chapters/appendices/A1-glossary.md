# Glossary of Terms — Every Term Explained Simply

> A complete reference of all technical terms used throughout this guide, explained in plain English with analogies.

---

## A

| Term | Simple Explanation |
|------|-------------------|
| **ACID** | Database guarantee: Atomic (all or nothing), Consistent (valid state), Isolated (no interference), Durable (survives crashes) |
| **Active-Active** | Multiple servers all handling requests simultaneously (like multiple cashiers open) |
| **Active-Passive** | One server handles requests, another waits as backup (like a backup goalkeeper) |
| **API** | Application Programming Interface — a menu of operations a service offers to other programs |
| **API Gateway** | The front door of your system — routes requests, handles auth, rate limiting |
| **APM** | Application Performance Monitoring — tools that track how fast your app responds |
| **Async** | Asynchronous — fire and forget; don't wait for a response before continuing |
| **Auto Scaling** | Automatically adding/removing servers based on traffic (like hiring seasonal workers) |
| **Availability** | Percentage of time a system is operational (99.99% = 52 minutes downtime/year) |
| **AZ (Availability Zone)** | Separate data centers within a region, far enough apart to survive disasters independently |

---

## B

| Term | Simple Explanation |
|------|-------------------|
| **Backpressure** | Telling the sender to slow down when the receiver is overwhelmed |
| **BASE** | Basically Available, Soft state, Eventually consistent — relaxed database guarantee |
| **Batch Processing** | Processing data in large chunks at scheduled intervals (opposite of streaming) |
| **BFF** | Backend For Frontend — a dedicated backend tailored for each client type (web, mobile) |
| **Blue-Green Deployment** | Running two identical environments; switching traffic to the new one instantly |
| **Broker** | A middleman that receives, stores, and delivers messages (like a post office) |
| **Bulkhead** | Isolating components so one failure doesn't sink the entire system (like ship compartments) |

---

## C

| Term | Simple Explanation |
|------|-------------------|
| **Cache** | A fast temporary storage for frequently accessed data (like keeping a sticky note vs reading the encyclopedia) |
| **Cache Eviction** | Removing old items from cache to make room for new ones (LRU, LFU, TTL) |
| **Cache Invalidation** | Knowing when cached data is stale and needs refreshing |
| **Canary Deployment** | Rolling out changes to a small percentage of users first to test safely |
| **CAP Theorem** | In a distributed system, you can only guarantee 2 of 3: Consistency, Availability, Partition tolerance |
| **CDC** | Change Data Capture — streaming database changes as events |
| **CDN** | Content Delivery Network — servers worldwide that cache content close to users |
| **CI/CD** | Continuous Integration / Continuous Deployment — automated build, test, and deploy |
| **Circuit Breaker** | Stops calling a failing service to prevent cascading failures (like an electrical circuit breaker) |
| **Client** | The requester (browser, mobile app) that asks a server for data |
| **Cluster** | A group of servers working together as one unit |
| **Concurrency** | Handling multiple tasks that overlap in time (not necessarily simultaneously) |
| **Connection Pool** | Pre-created reusable connections to avoid the overhead of creating new ones each time |
| **Consensus** | Multiple nodes agreeing on a value (Paxos, Raft algorithms) |
| **Container** | A lightweight, isolated package containing an app and all its dependencies (Docker) |
| **CORS** | Cross-Origin Resource Sharing — rules for which websites can call your API |
| **CQRS** | Command Query Responsibility Segregation — separate models for reading and writing |
| **CSRF** | Cross-Site Request Forgery — tricking a user's browser into making unwanted requests |
| **CSR** | Client-Side Rendering — JavaScript builds the page in the browser |

---

## D

| Term | Simple Explanation |
|------|-------------------|
| **Database Sharding** | Splitting a database across multiple machines by some key (like dividing a phone book A-M, N-Z) |
| **DDoS** | Distributed Denial of Service — flooding a server with fake traffic to take it down |
| **Dead Letter Queue** | Where failed/unprocessable messages go (so they don't block the main queue) |
| **DNS** | Domain Name System — translates human-readable names (google.com) to IP addresses |
| **Docker** | Tool for creating, running, and managing containers |
| **Domain-Driven Design** | Structuring code around business domains rather than technical layers |

---

## E

| Term | Simple Explanation |
|------|-------------------|
| **EDA** | Event-Driven Architecture — systems that react to events rather than direct calls |
| **Edge Computing** | Processing data close to the user (at CDN/edge servers) instead of a central data center |
| **Elastic** | Can grow and shrink automatically based on demand |
| **Endpoint** | A specific URL/path where an API accepts requests |
| **ETL** | Extract, Transform, Load — moving data from one system to another with transformations |
| **Event Sourcing** | Storing every change as an immutable event (like a bank ledger, not just current balance) |
| **Eventually Consistent** | All copies will become the same... eventually (not instantly) |
| **Exactly-Once** | Processing each message exactly one time (the hardest delivery guarantee) |

---

## F

| Term | Simple Explanation |
|------|-------------------|
| **Failover** | Automatically switching to a backup when the primary fails |
| **Fan-Out** | One message triggers multiple downstream actions (like one email → many recipients) |
| **Feature Flag** | A toggle that enables/disables features without deploying new code |
| **FinOps** | Financial Operations — managing and optimizing cloud costs |
| **Forward Proxy** | A proxy that acts on behalf of clients (like a VPN) |

---

## G

| Term | Simple Explanation |
|------|-------------------|
| **Gateway** | Entry point that routes, transforms, and secures incoming requests |
| **GeoDNS** | DNS that returns different IPs based on the user's geographic location |
| **GitOps** | Managing infrastructure through Git repositories (infrastructure as code via Git) |
| **Gossip Protocol** | Nodes share information with random neighbors, spreading like gossip |
| **GraphQL** | API query language where clients request exactly the data they need |
| **gRPC** | High-performance RPC framework using Protocol Buffers (binary, fast) |
| **GSLB** | Global Server Load Balancing — distributing traffic across worldwide data centers |

---

## H

| Term | Simple Explanation |
|------|-------------------|
| **Health Check** | Periodic ping to verify a server is alive and responsive |
| **Helm** | Package manager for Kubernetes (like apt/yum but for K8s) |
| **Horizontal Scaling** | Adding more machines (scaling out) vs making one machine bigger |
| **Hot Spot** | A single point receiving disproportionate traffic/load |
| **HPA** | Horizontal Pod Autoscaler — Kubernetes feature that auto-scales pods |
| **HTTP/2** | Improved HTTP with multiplexing, header compression, server push |
| **HTTP/3** | HTTP over QUIC (UDP-based), eliminating TCP head-of-line blocking |

---

## I

| Term | Simple Explanation |
|------|-------------------|
| **Idempotent** | An operation that produces the same result no matter how many times you run it |
| **Index** | A data structure that speeds up database queries (like a book's index) |
| **Ingress** | Entry point for external traffic into a Kubernetes cluster |
| **ISR** | Incremental Static Regeneration — updating static pages without full rebuild |

---

## J-K

| Term | Simple Explanation |
|------|-------------------|
| **Jitter** | Variation in network latency (makes real-time apps glitchy) |
| **JWT** | JSON Web Token — a self-contained token carrying user identity and claims |
| **Kafka** | Distributed event streaming platform (like a massive, durable message queue) |
| **Kubernetes (K8s)** | Container orchestration platform — manages, scales, and heals containers |

---

## L

| Term | Simple Explanation |
|------|-------------------|
| **Latency** | Time from request sent to response received (lower is better) |
| **Leader Election** | Algorithm for nodes to agree on which one is "in charge" |
| **Load Balancer** | Distributes incoming traffic across multiple servers evenly |
| **LRU** | Least Recently Used — eviction policy that removes oldest-accessed items first |

---

## M

| Term | Simple Explanation |
|------|-------------------|
| **MCU** | Multipoint Control Unit — decodes, mixes, and re-encodes video (expensive) |
| **Message Queue** | A buffer between sender and receiver that stores messages until consumed |
| **Microservices** | Architecture where each business capability is a separate, independently deployable service |
| **Middleware** | Software that sits between client and server adding cross-cutting functionality |
| **Monolith** | A single application where all code lives together in one deployable unit |
| **MPA** | Multi-Page Application — traditional websites where each page is a full server request |
| **Multi-Tenancy** | One system serving multiple customers (tenants) with isolated data |

---

## N-O

| Term | Simple Explanation |
|------|-------------------|
| **NAT** | Network Address Translation — maps private IPs to public IPs |
| **NewSQL** | Databases combining SQL features with NoSQL scalability (CockroachDB, Spanner) |
| **NoSQL** | Non-relational databases (document, key-value, column, graph) |
| **N+1 Problem** | Making N extra database queries instead of fetching all data in one query |
| **OAuth 2.0** | Authorization framework for granting limited access without sharing passwords |
| **Observability** | Ability to understand system internals from external outputs (logs, metrics, traces) |
| **OLAP** | Online Analytical Processing — optimized for complex queries on large datasets |
| **OLTP** | Online Transaction Processing — optimized for fast individual transactions |
| **Orchestration** | A central coordinator telling services what to do (vs choreography) |
| **Outbox Pattern** | Storing events in a database table to guarantee reliable event publishing |

---

## P

| Term | Simple Explanation |
|------|-------------------|
| **Partition** | A subset of data or a segment of a message queue topic |
| **Partition Tolerance** | System continues working despite network splits between nodes |
| **Peer-to-Peer (P2P)** | Nodes communicate directly without a central server |
| **Pod** | Smallest deployable unit in Kubernetes (one or more containers) |
| **Polyglot Persistence** | Using different database types for different use cases in one system |
| **Pub/Sub** | Publish/Subscribe — publishers send messages to topics, subscribers receive them |

---

## Q-R

| Term | Simple Explanation |
|------|-------------------|
| **QUIC** | UDP-based transport protocol (HTTP/3), reduces connection setup time |
| **Raft** | Consensus algorithm — simpler alternative to Paxos for leader election |
| **Rate Limiting** | Restricting how many requests a client can make in a time period |
| **Read Replica** | A copy of the database optimized for read queries (reducing primary load) |
| **Redundancy** | Having backup components so the system survives individual failures |
| **Replication** | Copying data across multiple nodes for availability and performance |
| **REST** | Representational State Transfer — API design using HTTP methods (GET, POST, etc.) |
| **Reverse Proxy** | A proxy that acts on behalf of servers (Nginx, HAProxy) |
| **RPC** | Remote Procedure Call — calling a function on a remote server as if it were local |

---

## S

| Term | Simple Explanation |
|------|-------------------|
| **Saga Pattern** | Managing distributed transactions through a sequence of local transactions |
| **Scalability** | Ability to handle increased load by adding resources |
| **Schema Registry** | Central store for message schemas ensuring compatibility |
| **Server** | A computer that responds to client requests |
| **Serverless** | Cloud runs your code without you managing servers (AWS Lambda, etc.) |
| **Service Discovery** | How services find each other's network addresses dynamically |
| **Service Mesh** | Infrastructure layer handling service-to-service communication (Istio, Linkerd) |
| **SFU** | Selective Forwarding Unit — routes video packets without decoding (efficient) |
| **Sidecar** | A helper container that runs alongside the main app container |
| **SLA** | Service Level Agreement — contract guaranteeing minimum uptime/performance |
| **SLI** | Service Level Indicator — the actual measured metric (e.g., 99.95% uptime) |
| **SLO** | Service Level Objective — the target goal for an SLI |
| **SOA** | Service-Oriented Architecture — predecessor to microservices |
| **SPA** | Single Page Application — one HTML page, JavaScript handles navigation |
| **Split Brain** | When network partition causes nodes to think they're both the leader |
| **SQL** | Structured Query Language — standard language for relational databases |
| **SSE** | Server-Sent Events — server pushing updates to client over HTTP |
| **SSG** | Static Site Generation — pre-building HTML pages at build time |
| **SSR** | Server-Side Rendering — server generates full HTML for each request |
| **Sticky Session** | Load balancer sends same user to same server (for session state) |
| **Strangler Fig** | Gradually replacing a monolith by routing traffic to new services piece by piece |
| **Stream Processing** | Processing data continuously as it arrives (Kafka Streams, Flink) |
| **Strong Consistency** | All nodes see the same data at the same time (like a single-server database) |

---

## T

| Term | Simple Explanation |
|------|-------------------|
| **TCP** | Transmission Control Protocol — reliable, ordered delivery (slower) |
| **Throughput** | How much work is completed per unit of time (requests/second) |
| **TLS** | Transport Layer Security — encrypts data in transit (the S in HTTPS) |
| **Token** | A piece of data representing authentication/authorization (JWT, API key) |
| **Topic** | A named channel in a message broker where events are published |
| **TTL** | Time To Live — how long a cached item or DNS record is valid |
| **Twelve-Factor App** | Best practices for building cloud-native applications |

---

## U-V

| Term | Simple Explanation |
|------|-------------------|
| **UDP** | User Datagram Protocol — fast, unreliable delivery (good for real-time) |
| **Vertical Scaling** | Making a single machine bigger (more CPU, RAM) — scaling up |
| **Virtual Machine (VM)** | A software emulation of a physical computer |
| **VPA** | Vertical Pod Autoscaler — adjusts CPU/memory limits for Kubernetes pods |
| **VPC** | Virtual Private Cloud — isolated network in the cloud |

---

## W-Z

| Term | Simple Explanation |
|------|-------------------|
| **WAF** | Web Application Firewall — filters malicious HTTP traffic |
| **Watermark** | In streaming: a marker indicating "all events up to this time have arrived" |
| **WebSocket** | Full-duplex bidirectional communication over a single TCP connection |
| **Worker** | A process that handles background/async tasks from a queue |
| **XSS** | Cross-Site Scripting — injecting malicious scripts into web pages |
| **Zero Trust** | Security model: never trust, always verify — even inside the network |
| **ZooKeeper** | Distributed coordination service (used by Kafka, HBase for leader election) |

---

> **Tip**: Bookmark this page. As you read through the chapters, come back here whenever you encounter an unfamiliar term.
