# Chapter 3.5: WebSockets — Real-Time Bidirectional Communication

> **Level**: ⭐⭐ Intermediate  
> **What you'll learn**: How WebSockets keep a permanent, two-way connection between browser and server — enabling real-time features like live chat, notifications, multiplayer games, and collaborative editing.

---

## 🧠 Real-Life Analogy: Phone Call vs Text Messages

**HTTP (REST)** = Text messages (SMS). You send a message, wait for a reply, conversation over. To say something else, you start a NEW message. The other person can't text you unless you text first.

**WebSocket** = A phone call. You dial once, the line stays open, and BOTH sides can talk whenever they want — simultaneously. No need to "dial again" for each sentence.

```
    HTTP (Request-Response):
    ════════════════════════
    
    Client: "Any new messages?"  ──▶  Server: "No"
    Client: "Any new messages?"  ──▶  Server: "No"
    Client: "Any new messages?"  ──▶  Server: "No"
    Client: "Any new messages?"  ──▶  Server: "Yes! Here's one"
    Client: "Any new messages?"  ──▶  Server: "No"
    
    Client keeps asking again and again! (POLLING)
    Wasteful — like calling someone every 2 seconds asking "anything new?"
    
    
    WebSocket (Persistent Connection):
    ═══════════════════════════════════
    
    Client: "Let's open a connection"  ──▶  Server: "OK, connected!"
    
    ... connection stays open ...
    
    Server: "Hey, you got a message!"  ──▶  Client  (server PUSHES)
    Client: "Thanks! Here's my reply"  ──▶  Server  (client sends)
    Server: "Another message for you"  ──▶  Client  (instant!)
    Client: "I'm typing..."           ──▶  Server  (real-time!)
    
    No wasted requests! Server pushes when it has something!
```

---

## 📖 How WebSockets Work

### The Handshake — Upgrading from HTTP

```
    WebSocket connections START as regular HTTP, then "upgrade":
    
    ┌──────────────────────────────────────────────────────────────┐
    │  Step 1: Client sends HTTP request with Upgrade header      │
    │                                                              │
    │  GET /chat HTTP/1.1                                          │
    │  Host: myapp.com                                             │
    │  Upgrade: websocket          ← "I want to upgrade!"         │
    │  Connection: Upgrade                                         │
    │  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==                │
    │  Sec-WebSocket-Version: 13                                   │
    │                                                              │
    │  Step 2: Server responds with 101 Switching Protocols        │
    │                                                              │
    │  HTTP/1.1 101 Switching Protocols                            │
    │  Upgrade: websocket          ← "OK, switching!"              │
    │  Connection: Upgrade                                         │
    │  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=        │
    │                                                              │
    │  Step 3: Connection UPGRADED — now it's a WebSocket! 🎉     │
    │                                                              │
    │  From this point on, BOTH sides can send messages anytime.  │
    │  No more HTTP request-response — it's a raw TCP pipe.       │
    └──────────────────────────────────────────────────────────────┘
    
    
    TIMELINE:
    ═════════
    
    Time ─────────────────────────────────────────────────────▶
    
    │ HTTP handshake │     WebSocket connection (persistent)     │
    │  (one-time)    │                                           │
    ├────────────────┼───────────────────────────────────────────┤
    │                │  ◀──▶ ◀──▶ ◀──▶ ◀──▶ ◀──▶ ◀──▶ ◀──▶    │
    │  Upgrade       │  Messages flow freely in BOTH directions │
    │  request       │  until either side closes the connection │
    │  + response    │                                           │
    
    Connection stays open for minutes, hours, or even DAYS!
```

### The WebSocket Protocol

```
    After the handshake, data flows as FRAMES:
    
    ┌──────────────────────────────────────────────────────┐
    │  WebSocket Frame Structure:                          │
    │                                                      │
    │  ┌──────┬──────────┬─────────┬────────────────────┐ │
    │  │ FIN  │ Opcode   │ Mask    │  Payload Data      │ │
    │  │ 1bit │ 4 bits   │ 1 bit   │  (your message)    │ │
    │  └──────┴──────────┴─────────┴────────────────────┘ │
    │                                                      │
    │  Opcodes:                                            │
    │  0x1 = Text frame (UTF-8 string)                    │
    │  0x2 = Binary frame (raw bytes)                     │
    │  0x8 = Close frame                                  │
    │  0x9 = Ping (keepalive)                             │
    │  0xA = Pong (keepalive response)                    │
    │                                                      │
    │  Overhead: just 2-14 bytes per frame!                │
    │  Compare: HTTP header = 200-800 bytes per request   │
    └──────────────────────────────────────────────────────┘
```

---

## 💻 Code Examples

### Python — WebSocket Server (using `websockets` library)

```python
"""
WebSocket chat server in Python.
Multiple clients can connect and chat in real-time.
"""
import asyncio
import websockets
import json

# Set of all connected clients
connected_clients = set()

async def handle_client(websocket):
    """Handle a single WebSocket connection."""
    connected_clients.add(websocket)
    client_addr = websocket.remote_address
    print(f"New client connected: {client_addr}")
    
    try:
        async for message in websocket:
            # Parse incoming message
            data = json.loads(message)
            print(f"Received from {client_addr}: {data}")
            
            # Broadcast to ALL connected clients
            broadcast_msg = json.dumps({
                "user": data.get("user", "Anonymous"),
                "text": data.get("text", ""),
                "timestamp": data.get("timestamp", "")
            })
            
            # Send to everyone EXCEPT the sender
            others = connected_clients - {websocket}
            if others:
                await asyncio.gather(
                    *[client.send(broadcast_msg) for client in others]
                )
    except websockets.exceptions.ConnectionClosed:
        print(f"Client disconnected: {client_addr}")
    finally:
        connected_clients.discard(websocket)

async def main():
    async with websockets.serve(handle_client, "0.0.0.0", 8765):
        print("WebSocket server running on ws://0.0.0.0:8765")
        await asyncio.Future()  # Run forever

if __name__ == "__main__":
    asyncio.run(main())
```

### JavaScript — WebSocket Client (Browser)

```javascript
// Browser-side WebSocket client
const ws = new WebSocket('ws://localhost:8765');

// ─── Connection opened ───
ws.onopen = () => {
    console.log('Connected to server!');
    // Send a message
    ws.send(JSON.stringify({
        user: 'Ritesh',
        text: 'Hello everyone!',
        timestamp: new Date().toISOString()
    }));
};

// ─── Receive messages from server ───
ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log(`${data.user}: ${data.text}`);
    // Update UI with new message
};

// ─── Connection closed ───
ws.onclose = (event) => {
    console.log('Disconnected from server');
    // Implement reconnection logic
};

// ─── Error handling ───
ws.onerror = (error) => {
    console.error('WebSocket error:', error);
};

// ─── Send message function ───
function sendMessage(text) {
    if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ user: 'Ritesh', text }));
    }
}
```

### Java — WebSocket Server (Spring Boot)

```java
/**
 * WebSocket server using Spring Boot.
 * Handles real-time chat messages.
 */
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new ChatWebSocketHandler(), "/chat")
                .setAllowedOrigins("*");
    }
}

public class ChatWebSocketHandler extends TextWebSocketHandler {
    
    // Track all connected sessions
    private final Set<WebSocketSession> sessions = 
        ConcurrentHashMap.newKeySet();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        sessions.add(session);
        System.out.println("Client connected: " + session.getId());
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, 
                                      TextMessage message) throws Exception {
        String payload = message.getPayload();
        System.out.println("Received: " + payload);
        
        // Broadcast to all other connected clients
        for (WebSocketSession s : sessions) {
            if (s.isOpen() && !s.getId().equals(session.getId())) {
                s.sendMessage(new TextMessage(payload));
            }
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, 
                                       CloseStatus status) {
        sessions.remove(session);
        System.out.println("Client disconnected: " + session.getId());
    }
}
```

---

## 🔄 WebSocket Architecture at Scale

```
    SINGLE SERVER (simple):
    ═══════════════════════
    
    Client A ──ws──┐
    Client B ──ws──┤── WebSocket Server ──── All in memory
    Client C ──ws──┘
    
    Works for < 10K connections. All clients on same server.
    
    
    MULTIPLE SERVERS (production):
    ═════════════════════════════
    
    Problem: Client A is on Server 1, Client B is on Server 2.
    How does A's message reach B?
    
    Client A ──ws──▶ Server 1 ──publish──▶ ┌──────────┐
                                           │  Redis   │
    Client B ──ws──▶ Server 2 ◀─subscribe─ │  Pub/Sub │
                                           └──────────┘
    Client C ──ws──▶ Server 1
    Client D ──ws──▶ Server 3 ◀─subscribe─ Redis Pub/Sub
    
    Solution: Use Redis Pub/Sub (or Kafka) as a message broker.
    When Server 1 gets a message, it publishes to Redis.
    All servers subscribe to Redis and forward to their local clients.
    
    
    FULL PRODUCTION ARCHITECTURE:
    ═════════════════════════════
    
    ┌──────────┐     ┌───────────────────┐
    │ Clients  │────▶│  Load Balancer    │ (sticky sessions / IP hash)
    └──────────┘     │  (Nginx/HAProxy)  │
                     └─────────┬─────────┘
                      ┌────────┼────────┐
                      ▼        ▼        ▼
                 ┌────────┐ ┌────────┐ ┌────────┐
                 │ WS     │ │ WS     │ │ WS     │
                 │ Server │ │ Server │ │ Server │
                 │  #1    │ │  #2    │ │  #3    │
                 └───┬────┘ └───┬────┘ └───┬────┘
                     │          │          │
                     └──────────┼──────────┘
                                ▼
                     ┌──────────────────┐
                     │   Redis Pub/Sub  │ (cross-server messaging)
                     └──────────────────┘
```

---

## 📊 HTTP vs WebSocket — When to Use What

```
    ┌────────────────────┬──────────────────────┬──────────────────────┐
    │  Aspect            │  HTTP (REST)         │  WebSocket           │
    ├────────────────────┼──────────────────────┼──────────────────────┤
    │  Connection        │  New for each request│  Persistent (open)   │
    ├────────────────────┼──────────────────────┼──────────────────────┤
    │  Direction         │  Client → Server     │  Both ways ↔️        │
    │                    │  (one way)           │  (bidirectional)     │
    ├────────────────────┼──────────────────────┼──────────────────────┤
    │  Server can push?  │  ❌ No              │  ✅ Yes              │
    ├────────────────────┼──────────────────────┼──────────────────────┤
    │  Overhead per msg  │  ~200-800 bytes      │  ~2-14 bytes         │
    │                    │  (HTTP headers)      │  (frame header)      │
    ├────────────────────┼──────────────────────┼──────────────────────┤
    │  Latency           │  Higher (handshake   │  Lower (already      │
    │                    │  per request)        │  connected)          │
    ├────────────────────┼──────────────────────┼──────────────────────┤
    │  Scalability       │  Easier (stateless)  │  Harder (stateful    │
    │                    │                      │  connections)        │
    ├────────────────────┼──────────────────────┼──────────────────────┤
    │  Caching           │  ✅ Easy             │  ❌ Not applicable   │
    ├────────────────────┼──────────────────────┼──────────────────────┤
    │  Use cases         │  CRUD, forms,        │  Chat, games,        │
    │                    │  page loads, APIs    │  live feeds,         │
    │                    │                      │  notifications,      │
    │                    │                      │  collaboration       │
    └────────────────────┴──────────────────────┴──────────────────────┘
```

---

## 🏢 Real-World Examples

```
    ┌──────────────────┬───────────────────────────────────────────────┐
    │  Application     │  WebSocket Usage                             │
    ├──────────────────┼───────────────────────────────────────────────┤
    │  WhatsApp Web    │  Real-time messages synced with phone        │
    │  Slack           │  Instant messaging, typing indicators        │
    │  Discord         │  Voice/video chat, game activity, messages   │
    ├──────────────────┼───────────────────────────────────────────────┤
    │  Google Docs     │  Real-time collaborative editing             │
    │  Figma           │  Multi-user design collaboration             │
    │  Notion          │  Live page updates across users              │
    ├──────────────────┼───────────────────────────────────────────────┤
    │  Uber            │  Live driver location on map                 │
    │  Zomato/Swiggy   │  Live order tracking                        │
    ├──────────────────┼───────────────────────────────────────────────┤
    │  Robinhood       │  Live stock price tickers                    │
    │  Binance         │  Real-time crypto trading data               │
    ├──────────────────┼───────────────────────────────────────────────┤
    │  Fortnite        │  Multiplayer game state synchronization      │
    │  Chess.com       │  Real-time move updates                      │
    └──────────────────┴───────────────────────────────────────────────┘
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Using WebSockets for everything
       → Regular CRUD operations don't need persistent connections!
       ✅ Use REST for CRUD, WebSocket only for real-time features
    
    ❌ Not implementing reconnection logic
       → Connections drop (network changes, server restarts, mobile sleep)
       ✅ Client should auto-reconnect with exponential backoff
    
    ❌ Not authenticating WebSocket connections
       → Anyone can connect to ws://yourapp.com/chat !
       ✅ Validate auth token during the HTTP upgrade handshake
    
    ❌ Storing connection state only in memory
       → Server restart = all connections lost, state gone
       ✅ Use Redis for cross-server state + persistent storage
    
    ❌ No heartbeat / ping-pong
       → Dead connections stay open, wasting server resources
       ✅ Send ping every 30s, close if no pong within 10s
    
    ❌ Broadcasting to ALL users when only SOME need the message
       → 10,000 users in a room, but the message is for 2 people
       ✅ Use rooms/channels to target specific recipients
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. WebSocket = persistent, bidirectional connection. Both client    ║
║     and server can send messages ANYTIME after the handshake.        ║
║                                                                      ║
║  2. Starts as HTTP, then "upgrades" to WebSocket via 101 status.    ║
║     After that, it's a raw TCP connection with minimal overhead.     ║
║                                                                      ║
║  3. Frame overhead is 2-14 bytes vs 200-800 bytes for HTTP headers. ║
║     This makes WebSocket ideal for high-frequency messaging.         ║
║                                                                      ║
║  4. At scale, use Redis Pub/Sub or Kafka for cross-server           ║
║     message routing when clients are on different servers.           ║
║                                                                      ║
║  5. Always implement: reconnection logic, authentication,           ║
║     heartbeats (ping/pong), and room-based messaging.                ║
║                                                                      ║
║  6. Don't use WebSocket for everything — REST handles CRUD          ║
║     better. WebSocket is for real-time bidirectional features.       ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

WebSockets are powerful but come with complexity (persistent connections, state management). For simpler "server pushes data to client" scenarios, there are lighter alternatives. Next: [Chapter 3.6: Server-Sent Events (SSE) & Long Polling](./06-sse-and-long-polling.md).

---

[⬅️ Previous: gRPC](./04-grpc.md) | [⬆️ Index](../../00-INDEX.md) | [Next: SSE & Long Polling ➡️](./06-sse-and-long-polling.md)
