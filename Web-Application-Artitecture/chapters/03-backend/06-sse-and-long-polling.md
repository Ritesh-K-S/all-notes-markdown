# Chapter 3.6: Server-Sent Events (SSE) & Long Polling

> **Level**: ⭐⭐ Intermediate  
> **What you'll learn**: Two lighter alternatives to WebSockets for server-to-client real-time communication — when you need the server to push data but don't need the client to send data back.

---

## 🧠 Real-Life Analogy: Three Ways to Get News

```
    POLLING (Short Polling):
    ════════════════════════
    You call the newspaper office every 5 minutes:
    "Any news?" → "No."
    "Any news?" → "No."
    "Any news?" → "Yes! Here's a headline."
    "Any news?" → "No."
    
    Wasteful! 90% of calls are "No."
    
    
    LONG POLLING:
    ═════════════
    You call the newspaper and they say:
    "I'll stay on the line. When there's news, I'll tell you."
    ... 30 seconds of silence ...
    "BREAKING NEWS: ..."
    *Call ends. You call again immediately.*
    
    More efficient! But you still hang up and reconnect.
    
    
    SSE (Server-Sent Events):
    ═════════════════════════
    You turn on a radio station:
    You just LISTEN. The station broadcasts whenever news happens.
    You don't need to call or ask — it just comes to you.
    
    One persistent connection. Server pushes when ready.
```

---

## 📊 Comparison: All Real-Time Approaches

```
    ┌────────────────┬───────────┬───────────┬───────────┬───────────┐
    │                │  Short    │  Long     │   SSE     │ WebSocket │
    │                │  Polling  │  Polling  │           │           │
    ├────────────────┼───────────┼───────────┼───────────┼───────────┤
    │  Direction     │ Client→   │ Client→   │ Server→   │ Both ↔️   │
    │                │ Server    │ Server    │ Client    │           │
    ├────────────────┼───────────┼───────────┼───────────┼───────────┤
    │  Connection    │ New each  │ Held open │ Held open │ Held open │
    │                │ request   │ until data│ long-term │ long-term │
    ├────────────────┼───────────┼───────────┼───────────┼───────────┤
    │  Latency       │ Up to     │ Near      │ Real-time │ Real-time │
    │                │ interval  │ real-time │           │           │
    ├────────────────┼───────────┼───────────┼───────────┼───────────┤
    │  Overhead      │ 🔴 High  │ 🟡 Medium │ 💚 Low   │ 💚 Low   │
    ├────────────────┼───────────┼───────────┼───────────┼───────────┤
    │  Complexity    │ 💚 Simple│ 🟡 Medium │ 💚 Simple│ 🔴 Complex│
    ├────────────────┼───────────┼───────────┼───────────┼───────────┤
    │  Auto-reconnect│ Built-in │ Manual    │ Built-in  │ Manual    │
    │  (browser)     │ (no need)│           │ ✅        │           │
    ├────────────────┼───────────┼───────────┼───────────┼───────────┤
    │  Browser       │ ✅ All   │ ✅ All   │ ✅ All   │ ✅ All    │
    │  support       │          │           │ (no IE)  │           │
    └────────────────┴───────────┴───────────┴───────────┴───────────┘
```

---

## 📖 Part 1: Long Polling — "Don't Hang Up Until You Have Something"

```
    HOW LONG POLLING WORKS:
    ═══════════════════════
    
    Step 1: Client sends request
    Step 2: Server HOLDS the request open (doesn't respond immediately)
    Step 3: When data is available → Server sends response
    Step 4: Client immediately sends another request
    Step 5: Repeat!
    
    ┌────────┐                              ┌────────┐
    │ Client │                              │ Server │
    └───┬────┘                              └───┬────┘
        │  GET /api/updates (request 1)         │
        │──────────────────────────────────────▶│
        │                                       │
        │         ... server WAITS ...          │  Holds connection
        │         ... 10 seconds ...            │  open until data
        │         ... 20 seconds ...            │  is available or
        │                                       │  timeout (30s)
        │         Event happens!                │
        │                                       │
        │◀───── 200 OK + {new data} ───────────│
        │                                       │
        │  GET /api/updates (request 2)         │  Client immediately
        │──────────────────────────────────────▶│  reconnects!
        │                                       │
        │         ... server WAITS again ...    │
        │                                       │
        │◀───── 200 OK + {more data} ──────────│
        │                                       │
        │  GET /api/updates (request 3)         │
        │──────────────────────────────────────▶│
        ... and so on forever ...
    
    
    VS SHORT POLLING:
    ═════════════════
    
    Short Polling:                     Long Polling:
    Client: request ──▶ Server         Client: request ──▶ Server
    Server: "no data" immediately       Server: ... waits ... waits ...
    Client: request ──▶ Server         Server: "here's data!" (event!)
    Server: "no data" immediately       Client: request ──▶ Server
    Client: request ──▶ Server         Server: ... waits ... waits ...
    Server: "here's data!"
    
    Short: 10 requests to get 1 update (9 wasted!)
    Long:   2 requests to get 1 update (efficient!)
```

### Long Polling — Python Server

```python
"""
Long Polling server using Flask.
Holds the request open until data is available.
"""
from flask import Flask, jsonify, request
import time
import threading
import queue

app = Flask(__name__)

# Message queue for pending updates
update_queues = {}  # client_id → queue

@app.route('/api/updates', methods=['GET'])
def long_poll():
    """Hold connection open until data is available or timeout."""
    client_id = request.args.get('client_id', 'default')
    timeout = int(request.args.get('timeout', 30))  # 30s default
    
    # Create a queue for this client if not exists
    if client_id not in update_queues:
        update_queues[client_id] = queue.Queue()
    
    q = update_queues[client_id]
    
    try:
        # BLOCK here until data arrives or timeout
        data = q.get(timeout=timeout)
        return jsonify({"status": "update", "data": data}), 200
    except queue.Empty:
        # Timeout — no data available, client should reconnect
        return jsonify({"status": "timeout"}), 200

@app.route('/api/send', methods=['POST'])
def send_update():
    """Push an update to all waiting clients."""
    data = request.get_json()
    for client_id, q in update_queues.items():
        q.put(data)  # Unblocks the waiting long-poll request!
    return jsonify({"sent_to": len(update_queues)}), 200
```

### Long Polling — Java Server

```java
/**
 * Long Polling with Spring Boot using DeferredResult.
 * DeferredResult releases the servlet thread while waiting.
 */
@RestController
public class LongPollController {

    // Store pending requests per client
    private final Map<String, DeferredResult<ResponseEntity<?>>> 
        pendingRequests = new ConcurrentHashMap<>();

    @GetMapping("/api/updates")
    public DeferredResult<ResponseEntity<?>> longPoll(
            @RequestParam String clientId) {
        
        // Create a deferred result with 30s timeout
        DeferredResult<ResponseEntity<?>> result = 
            new DeferredResult<>(30000L);
        
        result.onTimeout(() -> 
            result.setResult(ResponseEntity.ok(
                Map.of("status", "timeout"))));
        
        result.onCompletion(() -> 
            pendingRequests.remove(clientId));
        
        pendingRequests.put(clientId, result);
        return result;  // Thread is RELEASED, not blocked!
    }

    @PostMapping("/api/send")
    public ResponseEntity<?> sendUpdate(@RequestBody Map<String, Object> data) {
        // Resolve ALL pending long-poll requests
        for (var entry : pendingRequests.entrySet()) {
            entry.getValue().setResult(
                ResponseEntity.ok(Map.of("status", "update", "data", data))
            );
        }
        return ResponseEntity.ok(Map.of("sent_to", pendingRequests.size()));
    }
}
```

---

## 📖 Part 2: Server-Sent Events (SSE) — "One-Way Radio Broadcast"

SSE is a **native browser API** where the server sends a stream of events to the client over a single, long-lived HTTP connection.

```
    HOW SSE WORKS:
    ══════════════
    
    ┌────────┐                              ┌────────┐
    │ Client │                              │ Server │
    └───┬────┘                              └───┬────┘
        │  GET /api/events                      │
        │  Accept: text/event-stream            │
        │──────────────────────────────────────▶│
        │                                       │
        │  HTTP/1.1 200 OK                      │
        │  Content-Type: text/event-stream      │
        │  Connection: keep-alive               │
        │                                       │
        │◀── data: {"price": 150.50}\n\n ──────│  Event 1
        │                                       │
        │     ... 2 seconds later ...           │
        │                                       │
        │◀── data: {"price": 151.20}\n\n ──────│  Event 2
        │                                       │
        │     ... 5 seconds later ...           │
        │                                       │
        │◀── data: {"price": 149.80}\n\n ──────│  Event 3
        │                                       │
        │     ... connection stays open ...     │
        │     Server pushes whenever it wants   │
        │                                       │
    
    
    SSE MESSAGE FORMAT:
    ═══════════════════
    
    event: price-update           ← Event type (optional)
    id: 42                        ← Event ID (for reconnection)
    retry: 3000                   ← Reconnect after 3s if disconnected
    data: {"symbol": "AAPL",     ← The actual data
    data:  "price": 150.50}       ← Multi-line data
                                  ← Blank line = end of event
    
    event: notification
    id: 43
    data: {"text": "Order shipped!"}
    
    (blank line between events)
```

### SSE — Python Server

```python
"""
SSE (Server-Sent Events) server using Flask.
Streams live stock prices to connected clients.
"""
from flask import Flask, Response
import json
import time
import random

app = Flask(__name__)

def generate_stock_events():
    """Generator that yields SSE-formatted events."""
    event_id = 0
    price = 150.00
    
    while True:
        # Simulate price changes
        price += random.uniform(-2.0, 2.0)
        price = round(max(0, price), 2)
        event_id += 1
        
        # Format as SSE (MUST follow this exact format!)
        event = f"id: {event_id}\n"
        event += f"event: price-update\n"
        event += f"data: {json.dumps({'symbol': 'AAPL', 'price': price})}\n"
        event += f"\n"  # Blank line = end of event
        
        yield event
        time.sleep(1)  # Send update every second

@app.route('/api/stock-stream')
def stock_stream():
    """SSE endpoint — streams events to the client."""
    return Response(
        generate_stock_events(),
        mimetype='text/event-stream',  # THIS IS THE KEY!
        headers={
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive',
            'X-Accel-Buffering': 'no'  # Disable Nginx buffering
        }
    )

if __name__ == '__main__':
    app.run(port=8000, threaded=True)
```

### SSE — JavaScript Client (Browser)

```javascript
// Browser-side SSE client — built into the browser!
// No library needed. The EventSource API handles everything.

const eventSource = new EventSource('/api/stock-stream');

// Listen for specific event types
eventSource.addEventListener('price-update', (event) => {
    const data = JSON.parse(event.data);
    console.log(`${data.symbol}: $${data.price}`);
    
    // Update UI
    document.getElementById('price').textContent = `$${data.price}`;
});

// Listen for any event (default)
eventSource.onmessage = (event) => {
    console.log('Message:', event.data);
};

// Auto-reconnection is BUILT IN! 🎉
// If connection drops, browser reconnects automatically.
// It sends the last event ID so the server can resume.

eventSource.onerror = (error) => {
    console.log('Connection lost. Auto-reconnecting...');
    // Browser handles reconnection automatically!
};

// Close connection when done
// eventSource.close();
```

### SSE — Java Server (Spring Boot)

```java
/**
 * SSE with Spring Boot using SseEmitter.
 * Streams live notifications to connected clients.
 */
@RestController
public class SseController {

    private final List<SseEmitter> emitters = 
        new CopyOnWriteArrayList<>();

    @GetMapping("/api/notifications/stream")
    public SseEmitter streamNotifications() {
        SseEmitter emitter = new SseEmitter(0L); // No timeout
        emitters.add(emitter);
        
        emitter.onCompletion(() -> emitters.remove(emitter));
        emitter.onTimeout(() -> emitters.remove(emitter));
        emitter.onError(e -> emitters.remove(emitter));
        
        return emitter;
    }

    // Called when a new notification should be sent
    public void sendNotification(String message) {
        List<SseEmitter> deadEmitters = new ArrayList<>();
        
        for (SseEmitter emitter : emitters) {
            try {
                emitter.send(SseEmitter.event()
                    .name("notification")
                    .data(Map.of("text", message, 
                                 "time", Instant.now().toString())));
            } catch (IOException e) {
                deadEmitters.add(emitter);
            }
        }
        emitters.removeAll(deadEmitters);
    }
}
```

---

## 🔄 SSE vs WebSocket — Decision Guide

```
    ┌──────────────────────┬────────────────────┬──────────────────────┐
    │  Feature             │  SSE               │  WebSocket           │
    ├──────────────────────┼────────────────────┼──────────────────────┤
    │  Direction           │  Server → Client   │  Both ways ↔️        │
    │                      │  (one-way)         │  (bidirectional)     │
    ├──────────────────────┼────────────────────┼──────────────────────┤
    │  Protocol            │  HTTP/1.1 or HTTP/2│  WS:// (custom)     │
    ├──────────────────────┼────────────────────┼──────────────────────┤
    │  Auto-reconnect      │  ✅ Built into     │  ❌ Must implement  │
    │                      │  browser API       │  manually            │
    ├──────────────────────┼────────────────────┼──────────────────────┤
    │  Event IDs / resume  │  ✅ Built-in       │  ❌ Must implement  │
    ├──────────────────────┼────────────────────┼──────────────────────┤
    │  Data format         │  Text only         │  Text + Binary       │
    ├──────────────────────┼────────────────────┼──────────────────────┤
    │  HTTP/2 multiplexing │  ✅ Yes            │  ❌ No              │
    ├──────────────────────┼────────────────────┼──────────────────────┤
    │  Firewall friendly   │  ✅ Standard HTTP  │  🟡 May be blocked  │
    ├──────────────────────┼────────────────────┼──────────────────────┤
    │  Max connections     │  6 per domain      │  Unlimited           │
    │  (HTTP/1.1)          │  (browser limit)   │                      │
    ├──────────────────────┼────────────────────┼──────────────────────┤
    │  Complexity          │  💚 Very simple    │  🔴 More complex    │
    └──────────────────────┴────────────────────┴──────────────────────┘
    
    
    USE SSE WHEN:                         USE WEBSOCKET WHEN:
    ═════════════                         ═══════════════════
    ✅ Server pushes data to client       ✅ Client AND server send data
    ✅ Live notifications                 ✅ Chat applications
    ✅ Stock tickers / price updates      ✅ Multiplayer games
    ✅ Live news feeds                    ✅ Collaborative editing
    ✅ Build/deploy progress updates      ✅ Low-latency bidirectional
    ✅ Simple event streaming             ✅ Binary data transfer
    
    USE LONG POLLING WHEN:
    ═════════════════════
    ✅ SSE not supported (older browsers)
    ✅ Need to work through strict proxies
    ✅ Simple fallback mechanism
```

---

## 🏢 Real-World Examples

```
    ┌──────────────────┬─────────────┬──────────────────────────────────┐
    │  Application     │  Technology │  Use Case                        │
    ├──────────────────┼─────────────┼──────────────────────────────────┤
    │  Facebook Feed   │  Long Poll  │  Originally used long polling    │
    │                  │             │  for news feed updates           │
    ├──────────────────┼─────────────┼──────────────────────────────────┤
    │  Twitter/X       │  SSE        │  Live timeline updates,          │
    │                  │             │  streaming API                   │
    ├──────────────────┼─────────────┼──────────────────────────────────┤
    │  GitHub          │  SSE        │  CI/CD build logs streaming,     │
    │                  │             │  notification bell               │
    ├──────────────────┼─────────────┼──────────────────────────────────┤
    │  ChatGPT         │  SSE        │  Streaming AI responses          │
    │                  │             │  token by token!                 │
    ├──────────────────┼─────────────┼──────────────────────────────────┤
    │  Vercel          │  SSE        │  Deployment log streaming        │
    ├──────────────────┼─────────────┼──────────────────────────────────┤
    │  Trello          │  Long Poll  │  Board updates across users      │
    │                  │  + WebSocket│  (migrated to WS later)          │
    └──────────────────┴─────────────┴──────────────────────────────────┘
    
    Fun fact: ChatGPT streams responses using SSE!
    That's why you see words appearing one by one.
```

---

## ⚠️ Common Mistakes / Pitfalls

```
    ❌ Using SSE when you need client-to-server messaging
       → SSE is one-way only (server → client)
       ✅ Use WebSocket for bidirectional communication
    
    ❌ Not disabling Nginx/proxy buffering for SSE
       → Proxy buffers events, client gets them in batches instead of real-time!
       ✅ Add X-Accel-Buffering: no header or proxy_buffering off in Nginx
    
    ❌ Opening too many SSE connections from one browser
       → HTTP/1.1 limits to 6 connections per domain!
       ✅ Use HTTP/2 (multiplexing) or share one SSE stream for all event types
    
    ❌ Long polling without timeout
       → Connection held forever, wasting server resources
       ✅ Set a timeout (30s), return empty response, client reconnects
    
    ❌ Not sending Last-Event-ID on SSE reconnect
       → Client misses events that happened during disconnection
       ✅ Browser sends Last-Event-ID automatically; server must handle it
    
    ❌ Using short polling with very short intervals
       → Polling every 100ms = 600 requests/minute = DDoS on your own server!
       ✅ Use long polling or SSE instead for near-real-time needs
```

---

## 🔑 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  1. Long Polling = client sends request, server holds it open       ║
║     until data is available, then client reconnects. Near real-time.║
║                                                                      ║
║  2. SSE = server pushes a stream of events over a single HTTP       ║
║     connection. Built-in browser API with auto-reconnect.           ║
║                                                                      ║
║  3. SSE is simpler than WebSocket and works over standard HTTP.     ║
║     Perfect when you only need server-to-client data push.          ║
║                                                                      ║
║  4. ChatGPT, GitHub, Twitter all use SSE for streaming updates.     ║
║     It's production-proven and reliable.                             ║
║                                                                      ║
║  5. Use SSE for: notifications, live feeds, streaming AI responses. ║
║     Use WebSocket for: chat, games, collaborative editing.          ║
║     Use Long Polling for: simplest fallback, legacy browser support.║
║                                                                      ║
║  6. Always disable proxy/Nginx buffering for SSE endpoints.         ║
║     Always set timeouts for long polling.                            ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## What's Next?

We've covered how clients talk to servers (REST, GraphQL, gRPC, WebSocket, SSE). Now let's go deeper into the server itself — how does a server handle MULTIPLE requests at the same time? Next: [Chapter 3.7: Thread Models](./07-thread-models.md).

---

[⬅️ Previous: WebSockets](./05-websockets.md) | [⬆️ Index](../../00-INDEX.md) | [Next: Thread Models ➡️](./07-thread-models.md)
