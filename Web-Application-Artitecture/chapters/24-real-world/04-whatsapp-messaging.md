# How WhatsApp/Messenger Handles Billions of Messages

> **What you'll learn**: How WhatsApp delivers 100+ billion messages per day with just ~50 engineers, using Erlang/BEAM VM, end-to-end encryption, and a brilliantly simple architecture that prioritizes reliability over features.

---

## Real-Life Analogy

Imagine you're running the **world's largest postal service**, but with these wild constraints:

- **2 billion people** use your service daily
- Messages must arrive **within seconds** (not days)
- Every letter is sealed in an **unbreakable envelope** — even YOU (the postal service) can't read it
- If the recipient isn't home, you **hold the letter** and deliver it the moment they return
- You need to handle **100 billion letters per day** with a team of just 50 people
- The system must **never lose a single letter**

WhatsApp pulled this off by making extremely smart engineering choices — choosing the right tools, keeping things simple, and optimizing relentlessly.

---

## Core Concept Explained Step-by-Step

### Step 1: The Connection Model

Every WhatsApp client maintains a **persistent connection** to WhatsApp's servers:

```
┌─────────────────────────────────────────────────────────────────┐
│             WHATSAPP CONNECTION MODEL                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  2 billion devices, each with a persistent TCP/WebSocket         │
│  connection to WhatsApp servers                                  │
│                                                                   │
│  ┌──────┐     ┌──────┐     ┌──────┐                             │
│  │Phone │     │Phone │     │Phone │     ... (2 billion)         │
│  │  A   │     │  B   │     │  C   │                             │
│  └──┬───┘     └──┬───┘     └──┬───┘                             │
│     │             │             │                                 │
│     │ Persistent  │ Persistent  │ Persistent                     │
│     │ Connection  │ Connection  │ Connection                     │
│     │             │             │                                 │
│     ▼             ▼             ▼                                 │
│  ┌─────────────────────────────────────────────┐                 │
│  │         WhatsApp Server Cluster              │                 │
│  │   (Erlang/BEAM - millions of connections    │                 │
│  │    per server)                               │                 │
│  └─────────────────────────────────────────────┘                 │
│                                                                   │
│  Why persistent connections?                                      │
│  • Instant delivery (no polling = no delay)                      │
│  • Low overhead (no repeated TCP handshakes)                     │
│  • Push model (server pushes messages immediately)               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: How a Message Gets Delivered

```
Alice sends "Hello!" to Bob
         │
         ▼
┌────────────────────────────────────────────────────────────────┐
│ Step 1: ENCRYPT on Alice's device                              │
│ • Message encrypted with Bob's public key                      │
│ • Only Bob's device can decrypt it                             │
│ • WhatsApp servers CANNOT read the message                     │
└──────────────────────┬─────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────┐
│ Step 2: SEND to WhatsApp server                                │
│ • Encrypted message sent over persistent connection            │
│ • Server receives: {to: Bob, payload: [encrypted_blob]}       │
└──────────────────────┬─────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────┐
│ Step 3: ROUTING                                                 │
│ • Server looks up: "Which server is Bob connected to?"         │
│ • Routes message to that specific server                       │
└──────────────────────┬─────────────────────────────────────────┘
                       │
                ┌──────┴──────┐
                │             │
           Bob ONLINE    Bob OFFLINE
                │             │
                ▼             ▼
┌────────────────────┐  ┌────────────────────────────────┐
│ Step 4a: DELIVER   │  │ Step 4b: STORE & FORWARD       │
│ • Push to Bob's    │  │ • Store in offline queue       │
│   device instantly │  │ • When Bob comes online,       │
│ • Wait for ACK     │  │   deliver all pending msgs     │
│ • Send ✓✓ to Alice │  │ • Delete from server after     │
└────────────────────┘  │   successful delivery          │
                        └────────────────────────────────┘
```

### Step 3: The Double-Check Mark System

```
┌─────────────────────────────────────────────────────────────┐
│              MESSAGE ACKNOWLEDGMENT SYSTEM                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Alice                 Server                 Bob            │
│    │                     │                     │             │
│    │──── Message ───────▶│                     │             │
│    │                     │                     │             │
│    │◀─── ✓ (sent) ──────│                     │             │
│    │     (single tick)   │                     │             │
│    │                     │──── Message ───────▶│             │
│    │                     │                     │             │
│    │                     │◀─── ACK ───────────│             │
│    │                     │     (received)      │             │
│    │◀─── ✓✓ (delivered)──│                     │             │
│    │     (double tick)   │                     │             │
│    │                     │                     │             │
│    │                     │   Bob reads message │             │
│    │                     │◀── READ receipt ────│             │
│    │◀── ✓✓ (blue ticks)─│                     │             │
│    │                     │                     │             │
│                                                              │
│  Each state transition is persisted so status survives       │
│  server restarts.                                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Step 4: Group Messaging

```
┌─────────────────────────────────────────────────────────────────┐
│              GROUP MESSAGE DELIVERY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Alice sends "Hi everyone!" to Group (Alice, Bob, Charlie, Dave) │
│                                                                   │
│  Approach: Fan-out on SENDER side (not server)                   │
│                                                                   │
│  Alice's device:                                                  │
│  ┌──────────────────────────────────────────────┐                │
│  │  Encrypts message 3 times:                   │                │
│  │  • Once with Bob's key      → encrypted_B   │                │
│  │  • Once with Charlie's key  → encrypted_C   │                │
│  │  • Once with Dave's key     → encrypted_D   │                │
│  └──────────────────────────────────────────────┘                │
│                      │                                            │
│                      ▼                                            │
│  Server receives 3 encrypted messages, routes each:              │
│                                                                   │
│  encrypted_B ──────▶ Bob's server ──────▶ Bob                    │
│  encrypted_C ──────▶ Charlie's server ──▶ Charlie                │
│  encrypted_D ──────▶ Dave's server ─────▶ Dave                   │
│                                                                   │
│  NOTE: For large groups (up to 1024 members), WhatsApp uses      │
│  Signal Protocol's "Sender Keys" — sender creates one group      │
│  key, encrypts once, server fans out to all members.             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Why Erlang? The Secret Weapon

WhatsApp chose **Erlang/OTP** (running on BEAM VM) — a language designed for telecom systems in the 1980s. This was their unfair advantage:

| Erlang Feature | Benefit for WhatsApp |
|---------------|---------------------|
| **Lightweight processes** | 2 million connections per server (each connection = one Erlang process, ~2KB each) |
| **Preemptive scheduling** | No single connection can starve others |
| **Hot code loading** | Deploy new code without disconnecting users |
| **"Let it crash" philosophy** | Individual process crashes don't affect the system |
| **Pattern matching** | Elegant protocol handling |
| **Distributed by design** | Processes communicate across machines seamlessly |

```
┌─────────────────────────────────────────────────────────────────┐
│         ERLANG/BEAM: WHY IT'S PERFECT FOR MESSAGING              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Single WhatsApp Server:                                         │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                BEAM Virtual Machine                       │    │
│  │                                                         │    │
│  │  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐      │    │
│  │  │P1│ │P2│ │P3│ │P4│ │P5│ │P6│ │P7│ │P8│ │P9│ ...   │    │
│  │  └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘      │    │
│  │                                                         │    │
│  │  Each P = one user connection (~2KB memory)             │    │
│  │  2,000,000+ processes per server                        │    │
│  │  Each process handles ONE user's messages               │    │
│  │                                                         │    │
│  │  If P5 crashes → only that user affected               │    │
│  │  Supervisor restarts P5 automatically                   │    │
│  │  All other processes unaffected                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Compare to Java/C++:                                            │
│  • Java thread: ~1MB stack → 1000 threads = 1GB                 │
│  • Erlang process: ~2KB → 2 million processes = 4GB              │
│  • That's 2000x more concurrent users per GB of RAM!             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### End-to-End Encryption (Signal Protocol)

```
┌─────────────────────────────────────────────────────────────────┐
│          END-TO-END ENCRYPTION (Signal Protocol)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Key Exchange (happens once when Alice and Bob first chat):      │
│                                                                   │
│  Alice                    Server                    Bob           │
│    │                        │                        │            │
│    │── Upload public keys ─▶│◀── Upload public keys ─│            │
│    │   (Identity + Signed   │   (Identity + Signed   │            │
│    │    Pre-key + One-time) │    Pre-key + One-time) │            │
│    │                        │                        │            │
│    │◀─ Fetch Bob's keys ────│                        │            │
│    │                        │                        │            │
│    │  Alice computes shared │                        │            │
│    │  secret using X3DH     │                        │            │
│    │  (Extended Triple       │                        │            │
│    │   Diffie-Hellman)      │                        │            │
│    │                        │                        │            │
│    │── Encrypted msg ──────▶│──── Encrypted msg ────▶│            │
│    │   (server can't read)  │    (server can't read) │            │
│    │                        │                        │            │
│                                                                   │
│  Ongoing messages use Double Ratchet:                            │
│  • New encryption key for EVERY message                          │
│  • Forward secrecy: compromising one key doesn't                 │
│    reveal past messages                                           │
│  • Future secrecy: compromising one key doesn't                  │
│    reveal future messages                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Message Storage Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│              MESSAGE STORAGE PHILOSOPHY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  WhatsApp's key principle: STORE AS LITTLE AS POSSIBLE           │
│                                                                   │
│  ┌─────────────────────────────────────────────────┐             │
│  │  ON SERVER:                                      │             │
│  │  • Messages stored ONLY until delivered         │             │
│  │  • Once recipient gets the message → DELETED    │             │
│  │  • Offline messages kept max 30 days            │             │
│  │  • Server never has plaintext (E2E encrypted)   │             │
│  └─────────────────────────────────────────────────┘             │
│                                                                   │
│  ┌─────────────────────────────────────────────────┐             │
│  │  ON DEVICE:                                      │             │
│  │  • Full message history stored locally           │             │
│  │  • SQLite database on phone                     │             │
│  │  • Backup to Google Drive/iCloud (encrypted)    │             │
│  │  • Chat history is on YOUR device, not server   │             │
│  └─────────────────────────────────────────────────┘             │
│                                                                   │
│  Database: Mnesia (Erlang's built-in distributed DB)            │
│  → Later migrated to custom solution on top of                   │
│    FreeBSD + custom file formats for offline message queues      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Server Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│            WHATSAPP SERVER ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│                  ┌──────────────────┐                             │
│                  │   Load Balancer   │                             │
│                  │   (by phone #)    │                             │
│                  └────────┬─────────┘                             │
│                           │                                       │
│        ┌──────────────────┼──────────────────┐                   │
│        ▼                  ▼                  ▼                   │
│  ┌───────────┐     ┌───────────┐     ┌───────────┐             │
│  │ Chat      │     │ Chat      │     │ Chat      │             │
│  │ Server 1  │     │ Server 2  │     │ Server N  │             │
│  │           │     │           │     │           │             │
│  │ Erlang    │     │ Erlang    │     │ Erlang    │             │
│  │ 2M conns  │     │ 2M conns  │     │ 2M conns  │             │
│  └─────┬─────┘     └─────┬─────┘     └─────┬─────┘             │
│        │                  │                  │                   │
│        └──────────────────┼──────────────────┘                   │
│                           │                                       │
│                           ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              OFFLINE MESSAGE STORE                        │    │
│  │              (Mnesia / Custom storage)                    │    │
│  │              Messages deleted after delivery             │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Routing: User A is on Server 1, User B on Server 3             │
│  Server 1 routes message directly to Server 3 (Erlang           │
│  distribution protocol — processes talk across nodes)            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Store-and-Forward Message Queue

```python
# Simplified WhatsApp-style offline message queue
import time
from collections import defaultdict, deque
from dataclasses import dataclass
from typing import Optional, Dict, Deque
from threading import Lock

@dataclass
class Message:
    msg_id: str
    sender: str
    recipient: str
    encrypted_payload: bytes  # E2E encrypted — server can't read
    timestamp: float
    delivered: bool = False

class MessageRouter:
    """
    Simplified WhatsApp message routing with store-and-forward.
    Messages stored only until delivered, then deleted.
    """
    
    def __init__(self):
        # user_id → connection handler (None if offline)
        self.connections: Dict[str, Optional[object]] = {}
        # user_id → queue of pending messages
        self.offline_queue: Dict[str, Deque[Message]] = defaultdict(deque)
        self.lock = Lock()
        self.MAX_OFFLINE_DAYS = 30
    
    def user_connected(self, user_id, connection):
        """User comes online — deliver all pending messages."""
        self.connections[user_id] = connection
        
        # Flush offline queue
        while self.offline_queue[user_id]:
            msg = self.offline_queue[user_id].popleft()
            self._deliver(connection, msg)
            # Message delivered → delete from server (WhatsApp principle)
    
    def user_disconnected(self, user_id):
        """User goes offline."""
        self.connections[user_id] = None
    
    def route_message(self, message: Message) -> str:
        """Route message: deliver immediately or queue for later."""
        recipient_conn = self.connections.get(message.recipient)
        
        if recipient_conn:
            # Recipient online → deliver immediately
            self._deliver(recipient_conn, message)
            return "delivered"  # ✓✓ double tick
        else:
            # Recipient offline → store for later
            self.offline_queue[message.recipient].append(message)
            return "stored"  # ✓ single tick (sent to server)
    
    def _deliver(self, connection, message: Message):
        """Push message to connected client."""
        # In real WhatsApp: send over persistent TCP connection
        connection.send(message.encrypted_payload)
        message.delivered = True
```

### Java — Lightweight Connection Manager (Erlang-Inspired)

```java
import java.util.concurrent.*;
import java.util.Map;

/**
 * Demonstrates WhatsApp's per-user connection model.
 * Each user gets their own lightweight handler (like an Erlang process).
 * In real WhatsApp, this is done with Erlang's 2KB lightweight processes.
 */
public class ConnectionManager {
    // Each user mapped to their virtual connection process
    private final ConcurrentHashMap<String, UserSession> sessions = 
        new ConcurrentHashMap<>();
    
    // Offline message store (in production: distributed storage)
    private final ConcurrentHashMap<String, ConcurrentLinkedQueue<byte[]>> 
        offlineQueues = new ConcurrentHashMap<>();

    public void onUserConnect(String userId, Object socket) {
        UserSession session = new UserSession(userId, socket);
        sessions.put(userId, session);
        
        // Deliver all offline messages immediately
        ConcurrentLinkedQueue<byte[]> pending = offlineQueues.remove(userId);
        if (pending != null) {
            while (!pending.isEmpty()) {
                byte[] msg = pending.poll();
                session.deliver(msg);
                // Message delivered → removed from server
            }
        }
    }

    public void onUserDisconnect(String userId) {
        sessions.remove(userId);
    }

    public DeliveryStatus routeMessage(String recipientId, byte[] encryptedPayload) {
        UserSession recipientSession = sessions.get(recipientId);
        
        if (recipientSession != null && recipientSession.isActive()) {
            // Online → deliver now
            recipientSession.deliver(encryptedPayload);
            return DeliveryStatus.DELIVERED;  // ✓✓
        } else {
            // Offline → queue
            offlineQueues.computeIfAbsent(recipientId, 
                k -> new ConcurrentLinkedQueue<>()).add(encryptedPayload);
            return DeliveryStatus.QUEUED;  // ✓
        }
    }

    enum DeliveryStatus { QUEUED, DELIVERED, READ }
    
    static class UserSession {
        private final String userId;
        private final Object socket;
        
        UserSession(String userId, Object socket) {
            this.userId = userId;
            this.socket = socket;
        }
        
        boolean isActive() { return socket != null; }
        void deliver(byte[] data) { /* send over socket */ }
    }
}
```

---

## Infrastructure Examples

### WhatsApp's Remarkably Lean Stack

| Component | Technology | Notes |
|-----------|-----------|-------|
| **Language** | Erlang/OTP (BEAM VM) | Core messaging servers |
| **OS** | FreeBSD | Not Linux! FreeBSD chosen for network stack performance |
| **Database** | Mnesia → Custom | Erlang's built-in DB for routing tables |
| **Offline Store** | Custom (file-based) | Messages stored until delivery |
| **Protocol** | Custom binary (XMPP-derived) | Minimal overhead per message |
| **Encryption** | Signal Protocol (libsignal) | End-to-end on device |
| **Media Storage** | Not stored on device - separate flow | Images/videos expire after download |

### Hardware Efficiency

```
┌─────────────────────────────────────────────────────────────────┐
│        WHATSAPP: EFFICIENCY AT SCALE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  At time of Facebook acquisition (2014):                         │
│  • 450 million monthly active users                              │
│  • 72 billion messages/day                                       │
│  • ~50 engineers total                                           │
│  • <1000 servers                                                 │
│                                                                   │
│  Per-server metrics:                                             │
│  • 2+ million TCP connections per server                         │
│  • ~500K messages/sec per server                                │
│  • Using standard commodity hardware                             │
│                                                                   │
│  Compare to alternatives:                                        │
│  ┌──────────────┬──────────────┬────────────────┐               │
│  │ Framework    │ Connections  │ Memory/conn    │               │
│  ├──────────────┼──────────────┼────────────────┤               │
│  │ Java (Netty) │ ~100K-500K   │ ~10-50 KB      │               │
│  │ Node.js      │ ~100K-500K   │ ~10-20 KB      │               │
│  │ Go           │ ~500K-1M     │ ~4-8 KB        │               │
│  │ Erlang/BEAM  │ ~2M+         │ ~2 KB          │               │
│  └──────────────┴──────────────┴────────────────┘               │
│                                                                   │
│  This is why 50 engineers could serve 450M users.               │
│  The right tool for the right job.                               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### WhatsApp's Architecture Principles

1. **No Ads, No Games, No Gimmicks** — Focus purely on messaging reliability
2. **Delete messages after delivery** — Minimal server storage
3. **End-to-end encryption by default** — Server is just a router, not a reader
4. **Single phone number = identity** — No usernames, no passwords to remember
5. **Keep the team small** — 50 engineers forced simple, maintainable solutions

### How Multi-Device Works (WhatsApp Web/Desktop)

```
┌─────────────────────────────────────────────────────────────────┐
│              MULTI-DEVICE ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  OLD MODEL (before 2021):                                        │
│  Phone is master → Web/Desktop mirror via phone                  │
│  Problem: Phone must be online for Web to work                   │
│                                                                   │
│  NEW MODEL (2021+):                                              │
│  Each device is an independent client with its own keys          │
│                                                                   │
│  ┌────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ Phone  │  │ Desktop  │  │  Web     │  │ Tablet   │         │
│  │  Key1  │  │  Key2    │  │  Key3    │  │  Key4    │         │
│  └───┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
│      │             │             │             │                  │
│      └─────────────┴──────┬──────┴─────────────┘                │
│                           │                                      │
│                           ▼                                      │
│            ┌──────────────────────────┐                          │
│            │  WhatsApp Server          │                          │
│            │  Sends message to ALL     │                          │
│            │  linked devices           │                          │
│            │  (fan-out per device)     │                          │
│            └──────────────────────────┘                          │
│                                                                   │
│  Sender encrypts message N times (once per recipient device)     │
│  Each device can independently receive/decrypt                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Handling Network Unreliability

WhatsApp was designed for developing markets (India, Brazil, etc.) where networks are unreliable:

- **Tiny message overhead**: Custom binary protocol (not JSON/XML) — saves bytes
- **Compression**: Messages compressed before sending
- **Aggressive reconnection**: Reconnects instantly when network returns
- **Offline-first**: App works fully offline, syncs when connected
- **Low-bandwidth mode**: Reduces media quality automatically

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Better Approach |
|---------|---------------|-----------------|
| Using HTTP for every message | Huge overhead per message (headers, handshake) | Persistent TCP/WebSocket connection |
| Storing all messages permanently on server | Massive storage costs, privacy concerns | Store only until delivered, then delete |
| Using heavyweight threads per connection | Can't scale to millions of connections | Lightweight processes (Erlang) or async I/O |
| Server-side message decryption | Security risk, legal liability | End-to-end encryption; server is just a router |
| Polling for new messages | Wastes bandwidth, adds latency | Push-based delivery over persistent connections |
| Ignoring unreliable networks | Users in developing markets have flaky connectivity | Design for offline-first with store-and-forward |

---

## When to Use / When NOT to Use

### When to Use This Architecture
- Building a messaging system for millions of concurrent users
- Reliability and delivery guarantees are critical
- Privacy/encryption is a requirement
- You need to handle users with unreliable network connections
- Simple, focused functionality (messaging, not a social platform)

### When NOT to Use This Architecture
- Building a social feed (different access patterns — fan-out on read)
- You need message search across all conversations (WhatsApp doesn't server-side search)
- Rich collaborative features (use something like Slack's architecture instead)
- Team chat with channels and integrations (different problem domain)
- Small scale (< 100K users) → Use Firebase, Pusher, or Socket.io

---

## Key Takeaways

1. **Erlang/BEAM is WhatsApp's superpower** — 2M+ connections per server with only 2KB per process, enabling 50 engineers to serve 2 billion users
2. **Store nothing permanently** — Messages exist on the server only until delivered, then deleted. History lives on your device.
3. **End-to-end encryption (Signal Protocol)** means the server is just a routing layer — it literally cannot read your messages
4. **Persistent connections + push delivery** gives instant message delivery without polling overhead
5. **Offline-first design** with store-and-forward handles unreliable networks gracefully (crucial for WhatsApp's developing-market user base)
6. **FreeBSD over Linux** — WhatsApp chose FreeBSD for its superior network stack performance with millions of concurrent connections
7. **Keep the team small and the architecture simple** — constraints breed innovation. 50 engineers built what most companies need 5000 for.

---

## What's Next?

Next, we'll explore [How YouTube/TikTok Serves Billions of Videos](./05-youtube-video.md) — understanding how they handle 500+ hours of video uploaded every minute, process and store petabytes of content, and serve billions of views daily.
