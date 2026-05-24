# TCP vs UDP — When to Use What

> **What you'll learn**: The fundamental difference between TCP and UDP, how each protocol works internally, and how to choose the right one for your application based on reliability vs speed trade-offs.

---

## Real-Life Analogy

### TCP = Sending a Package via Registered Mail

When you send a valuable package via registered mail:
1. You fill out a form (connection setup / handshake)
2. The post office assigns a tracking number (sequence numbers)
3. You get a delivery confirmation (acknowledgment)
4. If the package is lost, it's automatically resent (retransmission)
5. Packages arrive in order, even if they took different routes (ordering)
6. When done, you close the account (connection teardown)

**Reliable, ordered, but SLOW.**

### UDP = Shouting in a Crowded Room

When you shout something in a room:
1. No setup needed — just start talking (connectionless)
2. No guarantee anyone heard you (no acknowledgment)
3. If they missed it, tough luck (no retransmission)
4. Multiple messages might arrive out of order
5. You can broadcast to everyone at once (multicast)

**Fast, simple, but UNRELIABLE.**

```
TCP (Transmission Control Protocol):
┌──────────┐    Reliable, Ordered, Slow    ┌──────────┐
│  Sender  │ ══════════════════════════════▶│ Receiver │
│          │◀══════════════════════════════ │          │
│          │    "Got it!" (ACK)             │          │
└──────────┘                               └──────────┘
  Every packet is accounted for. Nothing lost.

UDP (User Datagram Protocol):
┌──────────┐    Fast, Unordered, Lossy     ┌──────────┐
│  Sender  │ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─▶│ Receiver │
│          │  "Hope you get this!"          │          │
│          │  (some packets may be lost)    │          │
└──────────┘                               └──────────┘
  Fire and forget. Speed over reliability.
```

---

## Core Concept Explained Step-by-Step

### Step 1: Where Do TCP and UDP Live?

Both TCP and UDP sit in the **Transport Layer** (Layer 4) of the network stack:

```
┌────────────────────────────────┐
│ Layer 7: Application           │ (HTTP, HTTPS, DNS, FTP)
├────────────────────────────────┤
│ Layer 4: Transport             │ ← TCP and UDP live here
├────────────────────────────────┤
│ Layer 3: Network               │ (IP - routing)
├────────────────────────────────┤
│ Layer 2: Data Link             │ (Ethernet, WiFi)
├────────────────────────────────┤
│ Layer 1: Physical              │ (cables, radio waves)
└────────────────────────────────┘

Application chooses: "Should I use TCP or UDP?"
  │
  ├── TCP → Reliable stream (like a pipe)
  └── UDP → Individual packets (like postcards)
```

### Step 2: TCP — The Reliable Protocol

TCP guarantees:
- **Reliability** — Every byte sent will be received (or an error is reported)
- **Ordering** — Data arrives in the exact order it was sent
- **Flow control** — Sender slows down if receiver is overwhelmed
- **Congestion control** — Sender backs off if the network is congested
- **Error detection** — Corrupted packets are detected and resent

**The price?** More overhead, more latency, more complexity.

### Step 3: UDP — The Fast Protocol

UDP provides:
- **Speed** — No connection setup, no waiting for acknowledgments
- **Simplicity** — Just send the data and move on
- **Low overhead** — Tiny 8-byte header vs TCP's 20-byte header
- **Broadcast/Multicast** — Send to multiple receivers at once

**The price?** No reliability, no ordering, no flow control.

### Step 4: The TCP 3-Way Handshake

Before any data is sent, TCP requires a connection setup:

```
TCP Three-Way Handshake:

  Client                          Server
    │                               │
    │──── SYN (seq=100) ───────────▶│  Step 1: "Hey, I want to connect"
    │                               │
    │◀─── SYN-ACK (seq=300,ack=101)│  Step 2: "OK! I heard you, let's connect"
    │                               │
    │──── ACK (ack=301) ───────────▶│  Step 3: "Great, confirmed!"
    │                               │
    │         Connection             │
    │        Established!            │
    │                               │
    │════ DATA ════════════════════▶│  Now data can flow
    │◀═════ DATA ══════════════════ │
    │                               │

Total time: 1.5 × Round Trip Time (RTT)
If RTT = 50ms, handshake takes ~75ms BEFORE any data is sent!
```

### Step 5: UDP — No Handshake

```
UDP Communication:

  Client                          Server
    │                               │
    │──── Data Packet 1 ───────────▶│  Just send immediately!
    │──── Data Packet 2 ───────────▶│  No waiting!
    │──── Data Packet 3 ───────────▶│  Fire and forget!
    │                               │
    │  (Packet 2 might be lost,     │
    │   Packet 3 might arrive       │
    │   before Packet 1)            │

Total setup time: 0ms
Data starts flowing immediately.
```

### Step 6: Head-of-Line Blocking (TCP's Weakness)

```
TCP Problem — Head-of-Line Blocking:

Sent:     [Pkt 1] [Pkt 2] [Pkt 3] [Pkt 4] [Pkt 5]
                      ↑
                   LOST! (network dropped it)

Received: [Pkt 1] [???] [Pkt 3] [Pkt 4] [Pkt 5]
                          ↑
           TCP HOLDS these — won't deliver to app
           until Pkt 2 is retransmitted and received!

Result: Everything stalls waiting for ONE lost packet.

UDP doesn't have this problem:
  App gets Pkt 1, 3, 4, 5 immediately.
  Pkt 2 is just... gone. App handles it.
```

---

## How It Works Internally

### TCP Internal Mechanisms

```
TCP Segment Header (20 bytes minimum):
┌─────────────────────────────────────────────────────────┐
│ Source Port (16 bits) │ Destination Port (16 bits)      │
├─────────────────────────────────────────────────────────┤
│              Sequence Number (32 bits)                   │
├─────────────────────────────────────────────────────────┤
│           Acknowledgment Number (32 bits)                │
├─────────────────────────────────────────────────────────┤
│ Data   │ Reserved │ Flags      │ Window Size (16 bits)  │
│ Offset │  (3 bit) │ (9 bits)   │ (flow control)         │
│ (4 bit)│          │ SYN ACK FIN│                        │
├─────────────────────────────────────────────────────────┤
│ Checksum (16 bits)     │ Urgent Pointer (16 bits)       │
├─────────────────────────────────────────────────────────┤
│              Options (variable, 0-40 bytes)              │
└─────────────────────────────────────────────────────────┘
```

**Key TCP Mechanisms:**

1. **Sequence Numbers** — Every byte gets a number, ensuring order
2. **Acknowledgments (ACKs)** — Receiver tells sender what it got
3. **Sliding Window** — Sender can have multiple unacked packets in flight
4. **Retransmission Timer** — If ACK doesn't come back, resend
5. **Congestion Window (cwnd)** — Limits sending rate to avoid overwhelming the network

```
TCP Sliding Window:

Sender's view:
  [Sent+ACKd] [Sent, waiting ACK] [Can send] [Cannot send yet]
  ├──────────┤├──────────────────┤├─────────┤├───────────────┤
  Already      "In flight"        Window      Blocked (wait
  confirmed    (up to window      allows      for ACKs)
               size packets)      these

Window size adapts:
  • Slow Start: window = 1, 2, 4, 8, 16... (exponential growth)
  • After loss: window drops (congestion avoidance)
  • Steady state: window grows linearly
```

### UDP Internal Structure

```
UDP Datagram Header (8 bytes only!):
┌─────────────────────────────────────────────┐
│ Source Port (16 bits) │ Dest Port (16 bits) │
├─────────────────────────────────────────────┤
│ Length (16 bits)      │ Checksum (16 bits)  │
└─────────────────────────────────────────────┘

That's it! No sequence numbers, no ACKs, no window.
Maximum simplicity = maximum speed.

Max UDP payload: 65,507 bytes (but practical limit ~1400 bytes due to MTU)
```

### TCP Connection Teardown (4-Way)

```
Connection Closing:

  Client                          Server
    │                               │
    │──── FIN ─────────────────────▶│  "I'm done sending"
    │                               │
    │◀─── ACK ─────────────────────│  "OK, noted"
    │                               │
    │◀─── FIN ─────────────────────│  "I'm done too"
    │                               │
    │──── ACK ─────────────────────▶│  "OK, goodbye"
    │                               │
    │    TIME_WAIT (2×MSL)          │  Client waits ~60s
    │    (prevents late packets     │  before fully closing
    │     from being confused       │
    │     with new connections)     │
```

### TCP vs UDP — Packet Overhead Comparison

```
Sending "Hello" (5 bytes) over the network:

TCP total overhead:
  IP Header:  20 bytes
  TCP Header: 20 bytes
  Data:        5 bytes
  ─────────────────────
  Total:      45 bytes  (only 11% is actual data!)
  + 3-way handshake (3 packets before data even starts)

UDP total overhead:
  IP Header:  20 bytes
  UDP Header:  8 bytes
  Data:        5 bytes
  ─────────────────────
  Total:      33 bytes  (15% is actual data, no setup needed)
```

---

## Code Examples

### Python — TCP Server and Client

```python
# tcp_server.py — Reliable, ordered communication
import socket
import threading

def handle_client(conn, addr):
    """Handle a connected TCP client."""
    print(f"[TCP] Client connected: {addr}")
    
    while True:
        # TCP guarantees: all data arrives, in order
        data = conn.recv(1024)
        if not data:
            break
        
        message = data.decode('utf-8')
        print(f"[TCP] Received from {addr}: {message}")
        
        # Send acknowledgment back
        response = f"Server received: {message}"
        conn.send(response.encode('utf-8'))
    
    conn.close()
    print(f"[TCP] Client disconnected: {addr}")

# Create TCP socket
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  # SOCK_STREAM = TCP
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('0.0.0.0', 9000))
server.listen(5)  # Backlog queue of 5

print("[TCP] Server listening on port 9000")

while True:
    conn, addr = server.accept()  # Blocks until client connects (3-way handshake)
    threading.Thread(target=handle_client, args=(conn, addr)).start()
```

```python
# tcp_client.py
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(('localhost', 9000))  # 3-way handshake happens here

# Send messages — TCP guarantees delivery and order
messages = ["Hello", "How are you?", "Goodbye"]
for msg in messages:
    client.send(msg.encode('utf-8'))
    response = client.recv(1024)
    print(f"Server said: {response.decode('utf-8')}")

client.close()  # 4-way teardown
```

### Python — UDP Server and Client

```python
# udp_server.py — Fast, fire-and-forget communication
import socket

# Create UDP socket
server = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)  # SOCK_DGRAM = UDP
server.bind(('0.0.0.0', 9001))

print("[UDP] Server listening on port 9001")

while True:
    # No accept() needed — UDP is connectionless
    # Each recvfrom() gets one datagram
    data, addr = server.recvfrom(1024)
    message = data.decode('utf-8')
    print(f"[UDP] Received from {addr}: {message}")
    
    # Optionally respond (but no guarantee client gets it!)
    response = f"Got: {message}"
    server.sendto(response.encode('utf-8'), addr)
```

```python
# udp_client.py
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# No connect() needed — just start sending!
messages = ["Ping 1", "Ping 2", "Ping 3"]
for msg in messages:
    client.sendto(msg.encode('utf-8'), ('localhost', 9001))
    
    # Try to receive response (might not arrive!)
    client.settimeout(1.0)  # Don't wait forever
    try:
        data, _ = client.recvfrom(1024)
        print(f"Response: {data.decode('utf-8')}")
    except socket.timeout:
        print(f"No response for: {msg} (packet likely lost)")

client.close()
```

### Java — TCP vs UDP Comparison

```java
// TCPServer.java — Reliable stream communication
import java.io.*;
import java.net.*;

public class TCPServer {
    public static void main(String[] args) throws IOException {
        ServerSocket server = new ServerSocket(9000);
        System.out.println("[TCP] Listening on port 9000");
        
        while (true) {
            // accept() blocks until 3-way handshake completes
            Socket client = server.accept();
            System.out.println("[TCP] Client connected: " + client.getRemoteSocketAddress());
            
            // TCP provides a stream — data arrives reliably, in order
            BufferedReader in = new BufferedReader(
                new InputStreamReader(client.getInputStream()));
            PrintWriter out = new PrintWriter(client.getOutputStream(), true);
            
            String line;
            while ((line = in.readLine()) != null) {
                System.out.println("[TCP] Received: " + line);
                out.println("ACK: " + line);  // Guaranteed delivery
            }
            client.close();
        }
    }
}
```

```java
// UDPServer.java — Fast datagram communication
import java.net.*;

public class UDPServer {
    public static void main(String[] args) throws Exception {
        DatagramSocket socket = new DatagramSocket(9001);
        byte[] buffer = new byte[1024];
        System.out.println("[UDP] Listening on port 9001");
        
        while (true) {
            // No connection — just receive individual datagrams
            DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
            socket.receive(packet);  // Gets one datagram at a time
            
            String message = new String(packet.getData(), 0, packet.getLength());
            System.out.println("[UDP] Received: " + message 
                + " from " + packet.getSocketAddress());
            
            // Send response (fire and forget — might not arrive!)
            byte[] response = ("Got: " + message).getBytes();
            DatagramPacket reply = new DatagramPacket(
                response, response.length, packet.getSocketAddress());
            socket.send(reply);
        }
    }
}
```

---

## Infrastructure Examples

### When Real Systems Use TCP vs UDP

```
┌──────────────────────────────────────────────────────────────┐
│              PROTOCOL USAGE IN REAL SYSTEMS                    │
├────────────────────────────┬─────────────────────────────────┤
│         TCP                │           UDP                    │
├────────────────────────────┼─────────────────────────────────┤
│ HTTP/HTTPS (web browsing)  │ DNS queries (port 53)           │
│ REST APIs                  │ Video streaming (RTP)           │
│ gRPC                       │ Voice calls (VoIP/SIP)          │
│ Database connections       │ Online gaming (game state)      │
│ SSH                        │ IoT sensor data                 │
│ SMTP (email)               │ NTP (time sync)                 │
│ FTP (file transfer)        │ DHCP (IP assignment)            │
│ WebSockets (after upgrade) │ mDNS (discovery)               │
│ Kafka (broker connections) │ QUIC (HTTP/3 - UDP + custom)   │
│ Redis (client commands)    │ StatsD (metrics collection)    │
│ PostgreSQL connections     │ Multicast video (IPTV)         │
└────────────────────────────┴─────────────────────────────────┘
```

### Nginx Configuration for TCP vs UDP Proxying

```nginx
# Nginx stream module — proxy TCP and UDP traffic

stream {
    # TCP load balancing (databases, Redis, etc.)
    upstream postgres_cluster {
        server 10.0.1.10:5432;
        server 10.0.1.11:5432;
        server 10.0.1.12:5432 backup;
    }
    
    server {
        listen 5432;                    # TCP proxy for PostgreSQL
        proxy_pass postgres_cluster;
        proxy_connect_timeout 5s;
        proxy_timeout 3600s;            # Long-lived DB connections
    }
    
    # UDP load balancing (DNS, game servers, etc.)
    upstream dns_servers {
        server 10.0.2.10:53;
        server 10.0.2.11:53;
    }
    
    server {
        listen 53 udp;                  # UDP proxy for DNS
        proxy_pass dns_servers;
        proxy_timeout 5s;
        proxy_responses 1;              # Expect 1 response per request
    }
}
```

### Docker — Game Server (UDP) + API Server (TCP)

```yaml
# docker-compose.yml — Mixed TCP/UDP services
version: '3.8'

services:
  # REST API — uses TCP (reliable, ordered)
  api-server:
    image: myapp-api:latest
    ports:
      - "8080:8080/tcp"    # TCP for HTTP API
    
  # Game server — uses UDP (fast, real-time)
  game-server:
    image: myapp-game:latest
    ports:
      - "7777:7777/udp"    # UDP for game state updates
      - "7778:7778/tcp"    # TCP for chat/login (needs reliability)
    
  # DNS server — uses UDP (fast lookups)
  dns-server:
    image: coredns/coredns:latest
    ports:
      - "53:53/udp"        # UDP for DNS queries
      - "53:53/tcp"        # TCP for large DNS responses (zone transfers)
```

### QUIC — The Best of Both Worlds (UDP + Reliability)

```
QUIC Protocol (used by HTTP/3):
Built ON TOP of UDP, but adds reliability features.

┌─────────────────────────────────────────────────┐
│              Why not just use TCP?                │
│                                                  │
│  Problem with TCP:                               │
│  • 3-way handshake = 1.5 RTT delay             │
│  • Head-of-line blocking (one lost packet       │
│    blocks ALL streams)                           │
│  • Can't be updated (baked into OS kernels)     │
│  • TLS handshake adds another RTT              │
│                                                  │
│  QUIC's solution:                               │
│  • 0-RTT connection (for repeat connections)    │
│  • Multiple independent streams (no HOL)        │
│  • Built-in encryption (TLS 1.3 inside QUIC)   │
│  • Runs in user space (easy to update)          │
│  • Built on UDP (works through existing NATs)   │
└─────────────────────────────────────────────────┘

TCP Connection:
  Client ──SYN──▶ Server
  Client ◀──SYN-ACK── Server
  Client ──ACK──▶ Server
  Client ──TLS ClientHello──▶ Server
  Client ◀──TLS ServerHello── Server
  Client ──Data──▶ Server           Total: 3 RTTs!

QUIC Connection (0-RTT resume):
  Client ──Data──▶ Server           Total: 0 RTTs!
  (uses cached encryption keys from previous session)
```

---

## Real-World Example

### Online Gaming — Why UDP is King

```
Fortnite / PUBG / Valorant — Game Server Architecture:

                    UDP (game state: 60 updates/sec)
Player 1 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─▶ ┌──────────────┐
Player 2 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─▶ │ Game Server  │
Player 3 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─▶ │              │
                                          │ • Position   │
Player 1 ◀─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │ • Health     │
Player 2 ◀─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │ • Actions    │
Player 3 ◀─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │ • Physics    │
                                          └──────────────┘
                    TCP (login, chat, purchases)
Player 1 ═══════════════════════════════▶ ┌──────────────┐
                                          │ Auth Server  │
                                          │ Chat Server  │
                                          └──────────────┘

Why UDP for game state:
• Player position from 50ms ago is WORTHLESS — need latest
• If a packet is lost, just send the NEW position
• TCP would retransmit old position — causing lag/stutter
• 60 updates/sec means you get ~17ms per frame
• Can't afford TCP's retransmission delay (100-300ms)

Why TCP for chat/auth:
• "You killed Player3" message MUST arrive
• Login credentials MUST not be lost
• Purchase transactions MUST be reliable
```

### Zoom/Google Meet — Real-Time Video

```
Video Call Protocol Stack:

Audio/Video data:
  ┌──────┐    ┌──────┐    ┌──────────┐
  │ App  │───▶│ RTP  │───▶│   UDP    │  ← Real-time media
  │      │    │(Real-│    │          │    Lost frame? Skip it.
  │      │    │ time │    │          │    Old frame? Discard.
  │      │    │Proto)│    │          │
  └──────┘    └──────┘    └──────────┘

Signaling (call setup):
  ┌──────┐    ┌──────┐    ┌──────────┐
  │ App  │───▶│ SIP/ │───▶│   TCP    │  ← Control messages
  │      │    │WebRTC│    │          │    "Join call" MUST arrive
  │      │    │Signal│    │          │    "Hang up" MUST arrive
  └──────┘    └──────┘    └──────────┘

Real numbers for a 1080p video call:
• Video: ~2-4 Mbps over UDP (drops frames rather than buffer)
• Audio: ~100 Kbps over UDP (20ms packets)
• Signaling: ~1 Kbps over TCP/WebSocket
```

### Netflix — TCP for Streaming (Buffered Video)

```
Wait... Netflix uses TCP? Not UDP?

Netflix streaming:
  Client ═══TCP═══▶ CDN Server (HTTPS/TCP)

Why TCP works for Netflix (but not gaming):
1. Video is BUFFERED — client has 30-60 seconds of buffer
2. A lost packet causes a tiny pause (barely noticeable with buffer)
3. TCP's congestion control adapts video quality automatically
4. HTTPS (TCP) works through all firewalls/NATs
5. No real-time requirement — 2 seconds of delay is fine

Why UDP wouldn't help Netflix:
• They NEED all frames to arrive (reliable delivery)
• Buffering hides any retransmission delay
• HTTP byte-range requests let you skip ahead (seek)
• TCP's adaptive bitrate algorithms work perfectly
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Using TCP for real-time game state | Head-of-line blocking causes lag spikes | Use UDP + application-level sequencing |
| Using UDP for file transfers | Lost packets = corrupted files | Use TCP or implement your own reliability |
| Not handling UDP packet loss in app | App crashes or shows stale data | Implement sequence numbers and timeouts |
| Assuming UDP packets arrive in order | Network can reorder packets | Add sequence numbers, handle out-of-order |
| Using TCP for metrics/telemetry (StatsD) | Connection overhead for fire-and-forget data | Use UDP — losing 1 metric out of 1000 is fine |
| Large UDP packets (> MTU 1500 bytes) | IP fragmentation — all fragments must arrive or entire datagram is lost | Keep UDP payloads under 1400 bytes |
| Not implementing heartbeats on TCP | Dead connections go undetected for hours | Use TCP keepalive or application-level pings |
| Assuming TCP is "slow" and UDP is "fast" | TCP with proper tuning (window scaling, Nagle off) can saturate bandwidth | Profile before switching protocols |

---

## When to Use / When NOT to Use

### Use TCP When:

| Scenario | Why TCP |
|----------|---------|
| Web APIs (REST, gRPC) | Every request/response must arrive intact |
| File transfers | Cannot afford to lose a single byte |
| Database connections | Queries and results must be reliable |
| Email (SMTP) | Messages must not disappear |
| Chat messaging | "I love you" must not become "I you" |
| Financial transactions | Money transfer MUST be reliable |
| WebSockets | Reliable bidirectional after upgrade |

### Use UDP When:

| Scenario | Why UDP |
|----------|---------|
| Real-time gaming | Old position data is worthless |
| Live video/audio | Dropped frame is better than buffering |
| DNS lookups | Single small request/response, speed matters |
| IoT sensor readings | Sending 100/sec, losing 1 is fine |
| Service discovery | Broadcast/multicast needed |
| VoIP phone calls | Latency > reliability for voice |
| Metrics collection | Losing 1 data point out of 10000 is OK |

### Decision Flowchart:

```
Do you need EVERY packet to arrive?
├── YES → Use TCP
│         └── Can you tolerate 1+ RTT setup delay?
│             ├── YES → TCP is fine
│             └── NO → Use QUIC (UDP-based with 0-RTT)
│
└── NO → Is real-time speed critical?
          ├── YES → Use UDP
          │         └── Do you need ordering/some reliability?
          │             ├── YES → UDP + custom protocol (like QUIC, RTP)
          │             └── NO → Raw UDP is fine
          │
          └── NO → Just use TCP (simpler, works everywhere)
```

---

## Key Takeaways

1. **TCP** = reliable, ordered, connection-oriented. Use for anything where data MUST arrive (APIs, databases, file transfers).
2. **UDP** = fast, unordered, connectionless. Use when speed matters more than reliability (gaming, live video, DNS).
3. TCP's **3-way handshake** adds 1.5 RTT before any data flows. UDP has zero setup time.
4. TCP's **head-of-line blocking** means one lost packet stalls everything. UDP doesn't have this problem.
5. **QUIC** (HTTP/3) is the best of both worlds — built on UDP but adds reliability, encryption, and multiple streams without head-of-line blocking.
6. Most web applications use **TCP** (via HTTP). Only specialized real-time applications need UDP.
7. You can use **both in the same system** — TCP for control plane (login, chat), UDP for data plane (game state, video frames).

---

## What's Next?

Next, we'll explore **VPC, Subnets & Network Security Groups** — how cloud providers create isolated, secure network environments for your applications, and how to architect network boundaries in production.

→ [04-vpc-subnets.md](./04-vpc-subnets.md)
