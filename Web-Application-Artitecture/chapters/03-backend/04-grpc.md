# Chapter 3.4: gRPC — High-Performance Communication Between Services

> **Level**: ⭐⭐⭐ Advanced  
> **What you'll learn**: How gRPC uses HTTP/2 and Protocol Buffers to enable blazing-fast, type-safe communication between microservices — why Google, Netflix, and Uber use it for internal service-to-service calls.

---

## 🧠 Real-Life Analogy: Phone Call vs Letter

**REST API** = Sending a letter. You write it in English (JSON), put it in an envelope (HTTP), and mail it. The recipient reads the English text, understands it, and writes back.

**gRPC** = A phone call. You speak directly, both sides understand immediately (binary protocol), it's faster, and you can even talk **at the same time** (streaming).

```
    REST (Letter-based):
    ═══════════════════
    
    Service A writes JSON: {"user_id": 42, "name": "Ritesh"}
                                   │
                                   ▼
              Text-based, human-readable, but LARGE
              JSON parsing is SLOW (string → object)
              One request at a time per connection
                                   │
                                   ▼
    Service B receives, parses JSON, processes
    
    Overhead: ~500 bytes for a simple request + ~1ms parsing
    
    
    gRPC (Phone call):
    ═════════════════
    
    Service A sends binary: 0A 02 2A 12 06 52 69 74 65 73 68
                                   │
                                   ▼
              Binary, compact, NOT human-readable, but TINY
              No parsing needed (direct memory mapping)
              Multiple streams on ONE connection (HTTP/2)
                                   │
                                   ▼
    Service B receives, instantly uses the data
    
    Overhead: ~50 bytes for same request + ~0.01ms "parsing"
    
    10x smaller messages, 10x faster processing!
```

---

## 📖 What is gRPC?

**gRPC** = **g**oogle **R**emote **P**rocedure **C**all. It's a framework for calling functions on another server as if they were local functions.

```
    THE BIG PICTURE:
    ════════════════
    
    ┌──────────────────┐                    ┌──────────────────┐
    │  Service A       │                    │  Service B       │
    │  (Python)        │                    │  (Java)          │
    │                  │     gRPC call      │                  │
    │  result =        │                    │  def getUser(    │
    │   stub.getUser(  │ ──────────────────▶│    request):     │
    │     id=42        │                    │    return User(  │
    │   )              │ ◀──────────────────│      name="Ritesh"│
    │                  │     response       │    )             │
    │  print(result.   │                    │                  │
    │    name)         │                    │                  │
    │  → "Ritesh"      │                    │                  │
    └──────────────────┘                    └──────────────────┘
    
    Service A calls getUser(42) as if it were a LOCAL function.
    But it actually runs on Service B, possibly on another machine!
    
    gRPC handles:
    • Serializing the request (→ binary)
    • Sending it over HTTP/2
    • Deserializing on the other side
    • Sending the response back
    
    The developer just writes:  result = stub.getUser(id=42)
    gRPC does everything else!
    
    
    THREE KEY TECHNOLOGIES:
    ═══════════════════════
    
    ┌─────────────────────────────────────────────────────────┐
    │  1. Protocol Buffers (Protobuf) — Data format           │
    │     Like JSON but binary, 10x smaller, 100x faster     │
    │                                                         │
    │  2. HTTP/2 — Transport protocol                         │
    │     Multiplexing, streaming, header compression         │
    │                                                         │
    │  3. Code Generation — Auto-generates client/server code │
    │     Write a .proto file → get Python, Java, Go classes │
    └─────────────────────────────────────────────────────────┘
```

---

## 📝 Protocol Buffers — The Schema

```
    Step 1: Define your API in a .proto file
    ═════════════════════════════════════════
    
    This is like a CONTRACT between services.
    Both sides know exactly what data looks like.
```

```protobuf
// user_service.proto — The API contract

syntax = "proto3";

package userservice;

// ─── Message types (like JSON objects, but typed) ───
message GetUserRequest {
    int32 id = 1;           // Field number 1
}

message User {
    int32 id = 1;           // Field number 1
    string name = 2;        // Field number 2
    string email = 3;       // Field number 3
    int32 age = 4;          // Field number 4
    repeated string roles = 5;  // Array of strings
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
    int32 age = 3;
}

message UserList {
    repeated User users = 1;
}

message Empty {}

// ─── Service definition (like REST endpoints) ───
service UserService {
    // Unary RPC (single request → single response)
    rpc GetUser(GetUserRequest) returns (User);
    rpc CreateUser(CreateUserRequest) returns (User);
    
    // Server streaming (single request → multiple responses)
    rpc ListUsers(Empty) returns (stream User);
    
    // Client streaming (multiple requests → single response)
    rpc UploadUsers(stream CreateUserRequest) returns (UserList);
    
    // Bidirectional streaming (multiple ↔ multiple)
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

```
    Step 2: Generate code from .proto file
    ═══════════════════════════════════════
    
    protoc --python_out=. --grpc_python_out=. user_service.proto
    protoc --java_out=. --grpc_java_out=. user_service.proto
    
    This AUTOMATICALLY generates:
    
    Python:                          Java:
    ├── user_service_pb2.py          ├── UserServiceGrpc.java
    ├── user_service_pb2_grpc.py     ├── User.java
    │                                ├── GetUserRequest.java
    │   Contains:                    │   Contains:
    │   • User class                 │   • User class
    │   • GetUserRequest class       │   • GetUserRequest class
    │   • Server stub (abstract)     │   • Server stub (abstract)
    │   • Client stub (ready-to-use) │   • Client stub (ready-to-use)
    
    You don't write serialization code — it's all generated!
```

---

## 🔄 Four Types of gRPC Communication

```
    TYPE 1: UNARY (most common — like REST)
    ═══════════════════════════════════════
    
    Client ──── 1 request ───▶ Server
    Client ◀── 1 response ─── Server
    
    Example: GetUser(id=42) → User{name="Ritesh"}
    
    
    TYPE 2: SERVER STREAMING
    ════════════════════════
    
    Client ──── 1 request ──────────────────▶ Server
    Client ◀── response 1 ───────────────── Server
    Client ◀── response 2 ───────────────── Server
    Client ◀── response 3 ───────────────── Server
    Client ◀── ... (keeps streaming) ─────── Server
    
    Example: ListUsers() → streams users one by one
    Use case: Real-time stock prices, live log tailing
    
    
    TYPE 3: CLIENT STREAMING
    ════════════════════════
    
    Client ──── request 1 ──▶
    Client ──── request 2 ──▶ Server
    Client ──── request 3 ──▶
    Client ──── done ───────▶
    Client ◀── 1 summary ─── Server
    
    Example: Upload multiple files, then get summary
    Use case: Bulk data upload, sensor data ingestion
    
    
    TYPE 4: BIDIRECTIONAL STREAMING
    ═══════════════════════════════
    
    Client ──── request 1 ──▶ Server
    Client ◀── response 1 ─── Server
    Client ──── request 2 ──▶ Server
    Client ──── request 3 ──▶ Server
    Client ◀── response 2 ─── Server
    Client ◀── response 3 ─── Server
    
    Both sides send messages independently!
    Example: Chat application, live gaming
    Use case: Real-time collaboration, IoT
```

---

## 💻 Code Examples

### Python gRPC Server

```python
"""
gRPC server implementation in Python.
Handles GetUser and ListUsers RPCs.
"""
import grpc
from concurrent import futures
import user_service_pb2 as pb2
import user_service_pb2_grpc as pb2_grpc

# Simulated database
USERS = {
    1: {"name": "Ritesh", "email": "ritesh@example.com", "age": 28},
    2: {"name": "Priya", "email": "priya@example.com", "age": 25},
    3: {"name": "Amit", "email": "amit@example.com", "age": 32},
}

class UserServicer(pb2_grpc.UserServiceServicer):
    """Implements the UserService defined in .proto file."""
    
    def GetUser(self, request, context):
        """Unary RPC: single request → single response"""
        user_data = USERS.get(request.id)
        if not user_data:
            context.set_code(grpc.StatusCode.NOT_FOUND)
            context.set_details(f"User {request.id} not found")
            return pb2.User()
        
        return pb2.User(
            id=request.id,
            name=user_data["name"],
            email=user_data["email"],
            age=user_data["age"]
        )
    
    def ListUsers(self, request, context):
        """Server streaming: sends users one at a time"""
        for uid, data in USERS.items():
            yield pb2.User(id=uid, name=data["name"],
                          email=data["email"], age=data["age"])

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    pb2_grpc.add_UserServiceServicer_to_server(UserServicer(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    print("gRPC server running on port 50051")
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

### Python gRPC Client

```python
"""
gRPC client — calls the server as if calling a local function.
"""
import grpc
import user_service_pb2 as pb2
import user_service_pb2_grpc as pb2_grpc

def main():
    # Connect to gRPC server
    channel = grpc.insecure_channel('localhost:50051')
    stub = pb2_grpc.UserServiceStub(channel)
    
    # Unary call — feels like calling a local function!
    user = stub.GetUser(pb2.GetUserRequest(id=1))
    print(f"Got user: {user.name}, {user.email}")
    
    # Server streaming — iterate over results as they arrive
    print("\nAll users:")
    for user in stub.ListUsers(pb2.Empty()):
        print(f"  - {user.name} ({user.email})")

if __name__ == '__main__':
    main()
```

### Java gRPC Server

```java
/**
 * gRPC server in Java using Spring Boot + gRPC starter.
 */
@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {

    private final Map<Integer, UserData> users = Map.of(
        1, new UserData("Ritesh", "ritesh@example.com", 28),
        2, new UserData("Priya", "priya@example.com", 25)
    );

    @Override
    public void getUser(GetUserRequest request,
                        StreamObserver<User> responseObserver) {
        UserData data = users.get(request.getId());
        if (data == null) {
            responseObserver.onError(Status.NOT_FOUND
                .withDescription("User not found").asException());
            return;
        }
        // Build protobuf response
        User user = User.newBuilder()
            .setId(request.getId())
            .setName(data.name())
            .setEmail(data.email())
            .setAge(data.age())
            .build();
        
        responseObserver.onNext(user);     // Send response
        responseObserver.onCompleted();     // Signal completion
    }

    @Override
    public void listUsers(Empty request,
                          StreamObserver<User> responseObserver) {
        // Server streaming — send users one by one
        for (var entry : users.entrySet()) {
            User user = User.newBuilder()
                .setId(entry.getKey())
                .setName(entry.getValue().name())
                .build();
            responseObserver.onNext(user);  // Stream each user
        }
        responseObserver.onCompleted();     // Done streaming
    }
}
```

---

## 📊 REST vs GraphQL vs gRPC — When to Use What

```
    ┌─────────────────┬──────────────┬──────────────┬──────────────────┐
    │  Aspect         │  REST        │  GraphQL     │  gRPC            │
    ├─────────────────┼──────────────┼──────────────┼──────────────────┤
    │  Format         │  JSON (text) │  JSON (text) │  Protobuf (binary│
    │                 │              │              │  10x smaller)    │
    ├─────────────────┼──────────────┼──────────────┼──────────────────┤
    │  Protocol       │  HTTP/1.1    │  HTTP/1.1    │  HTTP/2          │
    ├─────────────────┼──────────────┼──────────────┼──────────────────┤
    │  Streaming      │  ❌ No      │  Subscription│  ✅ Full          │
    │                 │              │  (limited)   │  (4 patterns)    │
    ├─────────────────┼──────────────┼──────────────┼──────────────────┤
    │  Speed          │  🟡 Good    │  🟡 Good    │  ⚡ Fastest       │
    ├─────────────────┼──────────────┼──────────────┼──────────────────┤
    │  Type safety    │  Optional   │  Schema      │  Strong (.proto) │
    ├─────────────────┼──────────────┼──────────────┼──────────────────┤
    │  Browser support│  ✅ Native  │  ✅ Native   │  ❌ Needs proxy  │
    ├─────────────────┼──────────────┼──────────────┼──────────────────┤
    │  Human readable │  ✅ Yes     │  ✅ Yes      │  ❌ Binary        │
    ├─────────────────┼──────────────┼──────────────┼──────────────────┤
    │  Best for       │  Public APIs │  Complex UIs │  Service-to-     │
    │                 │  Web apps    │  Mobile apps │  service calls   │
    │                 │  Simple CRUD │  Multi-client│  Internal APIs   │
    │                 │              │              │  High-performance│
    └─────────────────┴──────────────┴──────────────┴──────────────────┘
    
    
    COMMON ARCHITECTURE:
    ════════════════════
    
    ┌──────────┐   REST/GraphQL   ┌──────────────┐   gRPC    ┌───────────┐
    │  Browser │ ────────────────▶│  API Gateway │ ─────────▶│ Service A │
    │  Mobile  │                  │              │ ─────────▶│ Service B │
    └──────────┘                  └──────────────┘ ─────────▶│ Service C │
                                                             └───────────┘
    
    External clients → REST or GraphQL (human-friendly)
    Internal services → gRPC (machine-optimized)
```

---

## 🏢 Real-World Examples

```
    ┌──────────────┬───────────────────────────────────────────────────┐
    │  Company     │  How They Use gRPC                               │
    ├──────────────┼───────────────────────────────────────────────────┤
    │  Google      │  Invented gRPC. ALL internal services use it.    │
    │              │  10 billion+ gRPC calls per second internally!   │
    ├──────────────┼───────────────────────────────────────────────────┤
    │  Netflix     │  Uses gRPC between microservices. REST for      │
    │              │  external APIs, gRPC for internal.               │
    ├──────────────┼───────────────────────────────────────────────────┤
    │  Uber        │  Migrated from Thrift to gRPC. 2,000+ micro-   │
    │              │  services communicating via gRPC.                │
    ├──────────────┼───────────────────────────────────────────────────┤
    │  Dropbox     │  Migrated from REST to gRPC for internal calls. │
    │              │  Saw ~50% reduction in latency.                  │
    ├──────────────┼───────────────────────────────────────────────────┤
    │  Square      │  Uses gRPC for payment processing services.     │
    │              │  Type safety is critical for financial data.     │
    └──────────────┴───────────────────────────────────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Using gRPC for browser-facing APIs
       → Browsers don't natively support HTTP/2 binary framing for gRPC
       ✅ Use gRPC-Web or REST/GraphQL for browser clients
    
    ❌ Not versioning .proto files
       → Changing field numbers breaks backward compatibility!
       ✅ Never reuse or change field numbers. Add new fields instead.
    
    ❌ Ignoring error handling
       → gRPC has its own status codes (NOT HTTP status codes!)
       ✅ Use proper gRPC status codes: NOT_FOUND, INVALID_ARGUMENT, etc.
    
    ❌ Sending huge messages
       → Default max message size is 4MB. Streaming is better for large data.
       ✅ Use streaming RPCs for large datasets.
    
    ❌ Not using deadlines/timeouts
       → gRPC calls can hang forever if the server is unresponsive!
       ✅ Always set deadlines: stub.GetUser(req, timeout=5)
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. gRPC = fast, binary, type-safe RPC framework using Protocol     ║
║     Buffers (data) + HTTP/2 (transport) + code generation.          ║
║                                                                      ║
║  2. 10x smaller messages, 10x faster serialization than JSON/REST.  ║
║     Built for high-performance service-to-service communication.     ║
║                                                                      ║
║  3. Four communication patterns: Unary, Server Streaming,          ║
║     Client Streaming, Bidirectional Streaming.                       ║
║                                                                      ║
║  4. Write a .proto file → auto-generate client and server code     ║
║     in Python, Java, Go, C++, etc. Strong type safety.              ║
║                                                                      ║
║  5. Use REST/GraphQL for external APIs (browsers, public clients).  ║
║     Use gRPC for internal service-to-service calls.                  ║
║                                                                      ║
║  6. Used by Google (10B+ calls/sec), Netflix, Uber, Dropbox.       ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

REST, GraphQL, and gRPC all follow a request-response model. But what if you need the server to **push** data to the client in real-time — like live chat, notifications, or stock tickers? Next: [Chapter 3.5: WebSockets](./05-websockets.md).

---

[⬅️ Previous: GraphQL](./03-graphql.md) | [⬆️ Index](../../00-INDEX.md) | [Next: WebSockets ➡️](./05-websockets.md)
