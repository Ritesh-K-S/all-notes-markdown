# How Zoom Handles Millions of Video Calls

> **What you'll learn**: How Zoom delivers real-time video conferencing for 300+ million daily meeting participants with sub-200ms latency, using Selective Forwarding Units (SFUs), global media routers, and ingenious bandwidth optimization — where even 300ms of delay makes conversation feel unnatural.

---

## Real-Life Analogy

Imagine you're hosting a **conference call with 100 people in 20 different countries**, and:

- Everyone needs to **hear and see** each other with **zero noticeable delay** (less than 200ms)
- Some people have blazing fast fiber, others are on shaky mobile networks
- If you simply sent everyone's video to everyone else, a 100-person call would need **9,900 video streams** (100 × 99) — impossible
- You need to figure out **who's speaking** and show them prominently, while minimizing bandwidth for people who aren't talking
- The whole thing must work even when someone's network drops to 100 kbps

This is fundamentally different from Netflix/YouTube (where a 5-second buffer is fine). In video calls, even **300ms of delay** makes conversation feel broken. This is the **hardest real-time problem** in web applications.

---

## Core Concept Explained Step-by-Step

### Step 1: Why Video Calls Are Harder Than Video Streaming

```
┌─────────────────────────────────────────────────────────────────┐
│     VIDEO STREAMING (Netflix) vs VIDEO CALLING (Zoom)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  STREAMING (Netflix):                                            │
│  ┌──────────┐    5-30 sec buffer    ┌──────────┐               │
│  │  Server  │ ═══════════════════▶  │  Client  │               │
│  └──────────┘    (one-way, buffered) └──────────┘               │
│                                                                   │
│  • One direction (server → client)                               │
│  • Can buffer 5-30 seconds ahead                                 │
│  • Latency doesn't matter (3 second delay is fine)              │
│  • Uses TCP (reliable, ordered delivery)                         │
│                                                                   │
│  ─────────────────────────────────────────────────────────────  │
│                                                                   │
│  VIDEO CALLING (Zoom):                                           │
│  ┌──────────┐    < 200ms latency    ┌──────────┐               │
│  │ User A   │ ◀══════════════════▶  │ User B   │               │
│  └──────────┘    (bidirectional,     └──────────┘               │
│                   real-time)                                      │
│                                                                   │
│  • Bidirectional (everyone sends AND receives)                   │
│  • CANNOT buffer (adds unacceptable delay)                       │
│  • Latency must be < 150ms for natural conversation             │
│  • Uses UDP (fast, tolerates packet loss)                        │
│  • Packet loss handled by FEC, not retransmission               │
│                                                                   │
│  ┌────────────────────────────────────────────────────┐         │
│  │  Latency Perception:                               │         │
│  │  • < 150ms: Natural conversation                   │         │
│  │  • 150-300ms: Noticeable but usable               │         │
│  │  • 300-500ms: Awkward, people talk over each other │         │
│  │  • > 500ms: Unusable for conversation             │         │
│  └────────────────────────────────────────────────────┘         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: The Topology Problem — How to Connect N Participants?

```
┌─────────────────────────────────────────────────────────────────┐
│       THREE APPROACHES TO MULTI-PARTY VIDEO                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  APPROACH 1: MESH (Peer-to-Peer)                                │
│  Each person sends video to EVERY other person directly         │
│                                                                   │
│      A ◀──────▶ B          Works for: 2-4 people               │
│      │╲        ╱│          Problem: N×(N-1) streams             │
│      │  ╲    ╱  │          4 people = 12 streams                │
│      │    ╲╱    │          10 people = 90 streams (impossible!) │
│      │    ╱╲    │                                               │
│      │  ╱    ╲  │          Bandwidth per user: N-1 uploads      │
│      │╱        ╲│                                               │
│      C ◀──────▶ D                                               │
│                                                                   │
│  ─────────────────────────────────────────────────────────────  │
│                                                                   │
│  APPROACH 2: MCU (Multipoint Control Unit) — OLD approach       │
│  Server receives all streams, decodes, mixes into one           │
│  composite video, re-encodes, sends one stream back             │
│                                                                   │
│      A ──▶ ┌─────┐ ──▶ A (sees B,C,D in one frame)            │
│      B ──▶ │ MCU │ ──▶ B                                       │
│      C ──▶ │     │ ──▶ C                                       │
│      D ──▶ └─────┘ ──▶ D                                       │
│                                                                   │
│      Pro: Low bandwidth for clients (one stream in/out)         │
│      Con: HUGE server cost (decode+mix+encode = CPU-intensive)  │
│           Adds latency (encoding takes time)                    │
│           No individual layout control per client                │
│                                                                   │
│  ─────────────────────────────────────────────────────────────  │
│                                                                   │
│  APPROACH 3: SFU (Selective Forwarding Unit) — ZOOM's approach  │
│  Server receives streams and FORWARDS them selectively          │
│  NO decoding/encoding on server — just routing!                 │
│                                                                   │
│      A ──▶ ┌─────┐ ──▶ B (receives A,C,D streams separately)  │
│      B ──▶ │ SFU │ ──▶ A (receives B,C,D streams separately)  │
│      C ──▶ │     │ ──▶ ...                                     │
│      D ──▶ └─────┘                                              │
│                                                                   │
│      Pro: Server just routes (cheap, fast, low latency)         │
│           Clients choose which streams to render                 │
│           Can receive different qualities per stream             │
│      Con: Client needs more download bandwidth (N-1 streams)    │
│           (Solved by Simulcast — see below)                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: Zoom's SFU Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│              ZOOM'S SFU-BASED ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Each participant sends ONE upload stream to SFU                     │
│  SFU selectively forwards to others based on:                       │
│  • Who is speaking (active speaker gets high quality)               │
│  • Gallery view (everyone gets lower quality)                       │
│  • Receiver's bandwidth (poor connection → lower quality)           │
│                                                                      │
│  ┌────────┐                                        ┌────────┐      │
│  │User A  │──── Upload (720p) ────▶┌──────────┐──▶│User C  │      │
│  │(speaker)│                        │          │   │(viewer)│      │
│  └────────┘                        │   SFU    │   └────────┘      │
│                                    │  Server  │                     │
│  ┌────────┐                        │          │   ┌────────┐      │
│  │User B  │──── Upload (720p) ────▶│  Routes  │──▶│User D  │      │
│  │(muted) │                        │  packets │   │(poor    │      │
│  └────────┘                        │  without │   │ network)│      │
│                                    │  decode  │   └────────┘      │
│  What SFU sends:                   └──────────┘                     │
│  • To C: A's video at 720p (A is speaking)                         │
│          B's video at 180p (B is muted, small thumbnail)           │
│  • To D: A's video at 360p (D has poor bandwidth)                  │
│          B's video at 90p (minimal)                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 4: Simulcast — Multiple Quality Streams

```
┌─────────────────────────────────────────────────────────────────┐
│              SIMULCAST: THE BANDWIDTH SOLUTION                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Each participant sends their video at 2-3 quality levels        │
│  simultaneously ("simulcast layers"):                            │
│                                                                   │
│  User A's camera → encodes → sends to SFU:                      │
│  ┌─────────────────────────────────────────┐                    │
│  │  Layer 1 (High):   720p @ 1.5 Mbps     │                    │
│  │  Layer 2 (Medium): 360p @ 500 Kbps     │                    │
│  │  Layer 3 (Low):    180p @ 150 Kbps     │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                   │
│  SFU decides which layer to forward to each receiver:           │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Receiver      │  Sees A as    │  Gets layer  │  Reason   │  │
│  │────────────────┼───────────────┼──────────────┼──────────│  │
│  │  User B (good  │  Active       │  High (720p) │  Speaker +│  │
│  │  bandwidth)    │  speaker      │              │  fast net │  │
│  │                │               │              │           │  │
│  │  User C (good  │  Thumbnail    │  Low (180p)  │  Not      │  │
│  │  bandwidth)    │  in gallery   │              │  speaker  │  │
│  │                │               │              │           │  │
│  │  User D (slow  │  Active       │  Medium      │  Speaker  │  │
│  │  connection)   │  speaker      │  (360p)      │  but slow │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Benefit: SFU can instantly switch layers (no re-encoding)      │
│  When someone starts speaking → SFU sends their high layer      │
│  When they stop → SFU switches to their low layer               │
│  No delay from transcoding!                                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 5: Zoom's Global Network

```
┌─────────────────────────────────────────────────────────────────────┐
│              ZOOM'S GLOBAL MEDIA ROUTING                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Zoom has data centers in 18+ global locations                      │
│  Users connect to nearest media server for lowest latency           │
│                                                                      │
│  Scenario: Call between users in Mumbai, London, and San Francisco  │
│                                                                      │
│  ┌────────────┐     ┌─────────────────┐     ┌────────────┐        │
│  │ User A     │     │   Zoom DC       │     │ User C     │        │
│  │ (Mumbai)   │────▶│   Mumbai        │     │ (SF)       │        │
│  └────────────┘     └────────┬────────┘     └─────┬──────┘        │
│                              │                     │                │
│                              │  Zoom's private     │                │
│                              │  backbone network   │                │
│                              │                     │                │
│                              ▼                     ▼                │
│                       ┌─────────────────┐   ┌─────────────────┐    │
│  ┌────────────┐      │   Zoom DC       │   │   Zoom DC       │    │
│  │ User B     │─────▶│   London        │───│   US-West       │    │
│  │ (London)   │      └─────────────────┘   └─────────────────┘    │
│  └────────────┘                                                     │
│                                                                      │
│  Key insight: Media routed through Zoom's PRIVATE network           │
│  (not the public internet) between data centers.                    │
│  This gives predictable latency and avoids internet congestion.    │
│                                                                      │
│  Each user connects to their nearest DC:                            │
│  Mumbai user → Mumbai DC (20ms)                                     │
│  London user → London DC (15ms)                                     │
│  SF user → US-West DC (10ms)                                       │
│                                                                      │
│  Inter-DC latency on private backbone:                              │
│  Mumbai ↔ London: ~100ms                                           │
│  London ↔ US-West: ~70ms                                          │
│  Total end-to-end: < 200ms for any pair                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Audio Processing Pipeline

Audio quality is actually MORE critical than video (you can tolerate pixelated video but not choppy audio):

```
┌─────────────────────────────────────────────────────────────────┐
│              AUDIO PROCESSING PIPELINE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Microphone input (raw audio)                                    │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────┐                        │
│  │  1. Noise Suppression (AI-based)     │                        │
│  │  • Remove keyboard typing            │                        │
│  │  • Remove background noise           │                        │
│  │  • Remove dog barking, baby crying   │                        │
│  │  • Deep learning model running       │                        │
│  │    locally on device                  │                        │
│  └──────────────────────┬───────────────┘                        │
│                         │                                         │
│                         ▼                                         │
│  ┌──────────────────────────────────────┐                        │
│  │  2. Acoustic Echo Cancellation (AEC) │                        │
│  │  • Prevents feedback loop            │                        │
│  │  • Speaker output → microphone pick  │                        │
│  │    up → sent back = echo             │                        │
│  │  • AEC removes the echo              │                        │
│  └──────────────────────┬───────────────┘                        │
│                         │                                         │
│                         ▼                                         │
│  ┌──────────────────────────────────────┐                        │
│  │  3. Codec Encoding (Opus codec)      │                        │
│  │  • 20ms audio frames                 │                        │
│  │  • Variable bitrate: 6-128 kbps      │                        │
│  │  • Adjusts based on network          │                        │
│  └──────────────────────┬───────────────┘                        │
│                         │                                         │
│                         ▼                                         │
│  ┌──────────────────────────────────────┐                        │
│  │  4. Forward Error Correction (FEC)   │                        │
│  │  • Add redundant data so lost packets│                        │
│  │    can be reconstructed              │                        │
│  │  • No retransmission (too slow!)     │                        │
│  └──────────────────────┬───────────────┘                        │
│                         │                                         │
│                         ▼                                         │
│  Sent over UDP to Zoom's SFU                                     │
│  (50 packets per second per audio stream)                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Handling Packet Loss and Network Jitter

```
┌─────────────────────────────────────────────────────────────────┐
│         DEALING WITH UNRELIABLE NETWORKS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Problem: UDP doesn't guarantee delivery or ordering            │
│  Packets can be lost, arrive late, or arrive out of order       │
│                                                                   │
│  Sent:     [1] [2] [3] [4] [5] [6] [7] [8]                    │
│  Received: [1] [2] [_] [4] [6] [5] [7] [8]                    │
│                   ↑ lost    ↑ reordered                          │
│                                                                   │
│  Solutions employed:                                              │
│                                                                   │
│  1. JITTER BUFFER (client-side)                                  │
│  ┌──────────────────────────────────────────┐                   │
│  │  Collect packets, wait 20-60ms,          │                   │
│  │  reorder, then play in sequence          │                   │
│  │                                          │                   │
│  │  Trade-off: More buffer = smoother       │                   │
│  │             but adds latency             │                   │
│  │  Adaptive: Buffer grows/shrinks with     │                   │
│  │            network conditions            │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                   │
│  2. FORWARD ERROR CORRECTION (FEC)                               │
│  ┌──────────────────────────────────────────┐                   │
│  │  Send redundant data alongside real data │                   │
│  │  If packet 3 is lost, reconstruct it     │                   │
│  │  from redundancy in packets 2 and 4      │                   │
│  │                                          │                   │
│  │  Overhead: 20-50% extra bandwidth        │                   │
│  │  But: No retransmission delay!           │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                   │
│  3. PACKET LOSS CONCEALMENT (for audio)                          │
│  ┌──────────────────────────────────────────┐                   │
│  │  If a packet is lost and FEC can't help: │                   │
│  │  • Repeat last packet (sounds OK for     │                   │
│  │    20ms of repeated audio)               │                   │
│  │  • Or interpolate between surrounding    │                   │
│  │    packets                               │                   │
│  │  • Human ear tolerates 1-2% loss well   │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                   │
│  4. ADAPTIVE BITRATE (for video)                                 │
│  ┌──────────────────────────────────────────┐                   │
│  │  High packet loss / low bandwidth?       │                   │
│  │  → Reduce video quality (720p → 360p)    │                   │
│  │  → Reduce framerate (30fps → 15fps)      │                   │
│  │  → Prioritize audio over video           │                   │
│  │  → In extreme cases: video off, audio only│                  │
│  └──────────────────────────────────────────┘                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Active Speaker Detection

```
┌─────────────────────────────────────────────────────────────────┐
│         ACTIVE SPEAKER DETECTION                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  With 20+ participants, bandwidth is precious.                  │
│  Solution: Only send high-quality video for active speakers.    │
│                                                                   │
│  Detection methods:                                              │
│  1. Audio energy level (who's talking loudest?)                  │
│  2. Voice Activity Detection (VAD) — ML-based                   │
│  3. Debouncing (don't switch for brief sounds like "um")        │
│                                                                   │
│  Bandwidth allocation in a 20-person meeting:                   │
│                                                                   │
│  ┌─────────────────────────────────────────────┐                │
│  │  Active speaker (1-2 people):               │                │
│  │  • 720p or 1080p video                      │                │
│  │  • 1.5-3 Mbps per stream                    │                │
│  │                                             │                │
│  │  Recent speakers (2-3 people):              │                │
│  │  • 360p video                               │                │
│  │  • 500 Kbps per stream                      │                │
│  │                                             │                │
│  │  Everyone else (15+ people):                │                │
│  │  • 180p or 90p thumbnails                   │                │
│  │  • 50-150 Kbps per stream                   │                │
│  │  • OR: Video paused, show avatar            │                │
│  │                                             │                │
│  │  Total download: ~5-8 Mbps (manageable)     │                │
│  │  vs. all at 720p: ~30 Mbps (impossible!)    │                │
│  └─────────────────────────────────────────────┘                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Jitter Buffer for Audio Packets

```python
# Simplified jitter buffer: reorders packets and plays them smoothly
import time
import heapq
from dataclasses import dataclass, field
from typing import Optional, List

@dataclass(order=True)
class AudioPacket:
    sequence_number: int
    timestamp_ms: float = field(compare=False)
    audio_data: bytes = field(compare=False, repr=False)
    received_at: float = field(compare=False, default=0)

class AdaptiveJitterBuffer:
    """
    Zoom-style adaptive jitter buffer.
    Collects out-of-order packets, waits briefly, 
    then plays them in correct order.
    
    Trade-off: Larger buffer = smoother audio, but more latency.
    """
    
    def __init__(self, target_delay_ms: float = 40):
        self.buffer: List[AudioPacket] = []  # Min-heap by sequence number
        self.target_delay_ms = target_delay_ms
        self.last_played_seq = -1
        self.jitter_estimate_ms = 20  # Running estimate of network jitter
    
    def receive_packet(self, packet: AudioPacket):
        """Packet arrives from network (possibly out of order)."""
        packet.received_at = time.time() * 1000
        
        # Discard if too old (already played past this sequence)
        if packet.sequence_number <= self.last_played_seq:
            return  # Late packet — discard
        
        # Add to buffer (heap maintains order)
        heapq.heappush(self.buffer, packet)
        
        # Update jitter estimate (for adaptive buffer sizing)
        self._update_jitter(packet)
    
    def get_next_frame(self) -> Optional[AudioPacket]:
        """Called every 20ms by audio playback thread."""
        if not self.buffer:
            return None  # Buffer empty — output silence (concealment)
        
        # Check if oldest packet has waited long enough
        oldest = self.buffer[0]
        wait_time = (time.time() * 1000) - oldest.received_at
        
        if wait_time >= self.target_delay_ms:
            # Ready to play — pop from buffer
            packet = heapq.heappop(self.buffer)
            self.last_played_seq = packet.sequence_number
            return packet
        
        return None  # Not ready yet — wait for more packets to arrive
    
    def _update_jitter(self, packet: AudioPacket):
        """Adapt buffer delay based on observed network jitter."""
        # In production: exponential moving average of inter-packet timing
        # If jitter increases → grow buffer (smoother but more delay)
        # If jitter decreases → shrink buffer (less delay)
        self.target_delay_ms = max(20, min(100, self.jitter_estimate_ms * 2))

# Usage
buffer = AdaptiveJitterBuffer(target_delay_ms=40)

# Packets arrive out of order (network reordering)
buffer.receive_packet(AudioPacket(seq=1, timestamp_ms=0, audio_data=b"..."))
buffer.receive_packet(AudioPacket(seq=3, timestamp_ms=40, audio_data=b"..."))
buffer.receive_packet(AudioPacket(seq=2, timestamp_ms=20, audio_data=b"..."))

# Playback retrieves in correct order after buffer delay
frame = buffer.get_next_frame()  # Returns seq=1 (after 40ms wait)
```

### Java — SFU Routing Logic

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Simplified Selective Forwarding Unit (SFU) routing logic.
 * Decides WHICH video layer to forward to WHICH participant
 * based on speaking status and receiver bandwidth.
 */
public class SFURouter {
    
    enum VideoLayer { HIGH, MEDIUM, LOW }
    
    // Meeting state
    private final Map<String, ParticipantState> participants = new ConcurrentHashMap<>();
    private volatile String activeSpeakerId = null;
    
    record ParticipantState(
        String participantId,
        boolean isSpeaking,
        long availableBandwidthKbps,
        boolean videoEnabled
    ) {}

    /**
     * Determine which video layer each participant should receive
     * for a given sender. Called on every video frame (~30fps per sender).
     */
    public Map<String, VideoLayer> routeVideo(String senderId) {
        Map<String, VideoLayer> routing = new HashMap<>();
        
        for (ParticipantState receiver : participants.values()) {
            if (receiver.participantId().equals(senderId)) continue; // Don't send to self
            if (!receiver.videoEnabled()) continue; // Receiver has video off
            
            VideoLayer layer = selectLayer(senderId, receiver);
            routing.put(receiver.participantId(), layer);
        }
        
        return routing;
    }

    private VideoLayer selectLayer(String senderId, ParticipantState receiver) {
        boolean senderIsSpeaking = senderId.equals(activeSpeakerId);
        long bandwidth = receiver.availableBandwidthKbps();
        
        // Rule 1: If receiver has very low bandwidth → always LOW
        if (bandwidth < 500) return VideoLayer.LOW;
        
        // Rule 2: Active speaker gets HIGH quality (if receiver can handle it)
        if (senderIsSpeaking && bandwidth > 1500) return VideoLayer.HIGH;
        if (senderIsSpeaking) return VideoLayer.MEDIUM;
        
        // Rule 3: Non-speakers get LOW (thumbnail in gallery view)
        if (participants.size() > 5) return VideoLayer.LOW;
        
        // Rule 4: Small meeting (≤5) → everyone gets MEDIUM
        return VideoLayer.MEDIUM;
    }

    public void updateSpeaker(String participantId) {
        this.activeSpeakerId = participantId;
        // In production: Notify all SFU nodes to switch layers
        // This triggers instant quality boost for active speaker
    }

    public void addParticipant(String id, long bandwidthKbps) {
        participants.put(id, new ParticipantState(id, false, bandwidthKbps, true));
    }

    public static void main(String[] args) {
        SFURouter sfu = new SFURouter();
        sfu.addParticipant("alice", 5000);  // 5 Mbps
        sfu.addParticipant("bob", 1000);    // 1 Mbps (slow)
        sfu.addParticipant("charlie", 3000); // 3 Mbps
        sfu.updateSpeaker("alice");
        
        Map<String, VideoLayer> routing = sfu.routeVideo("alice");
        // bob gets MEDIUM (alice is speaking, but bob has low bandwidth)
        // charlie gets HIGH (alice is speaking, charlie has good bandwidth)
        routing.forEach((id, layer) -> 
            System.out.printf("Send alice's video to %s at %s quality%n", id, layer));
    }
}
```

---

## Infrastructure Examples

### Zoom's Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Media Servers** | Custom C++ SFUs | Video/audio routing |
| **Signaling** | Custom (WebSocket) | Call setup, participant management |
| **Protocol** | Custom over UDP (not WebRTC) | Low-latency media transport |
| **Codec (Video)** | H.264, VP8, AV1 | Video compression |
| **Codec (Audio)** | Opus | Audio compression |
| **Network** | Private global backbone | Predictable inter-DC latency |
| **Cloud** | Hybrid (own DCs + AWS/Oracle) | Elastic capacity |
| **AI** | Custom ML | Noise suppression, virtual backgrounds |
| **Recording** | Cloud storage (AWS S3 equivalent) | Meeting recordings |

### Scale Numbers

```
┌─────────────────────────────────────────────────────────────────┐
│              ZOOM BY THE NUMBERS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  • 300+ million daily meeting participants (peak during COVID)  │
│  • 3.3 trillion annualized meeting minutes                      │
│  • Meetings support up to 1,000 video participants              │
│  • Webinars support up to 50,000 attendees                      │
│  • 18+ global data center locations                             │
│  • < 150ms average audio latency                                │
│  • Handles 30-50 packets/second per audio stream                │
│  • Handles 30 frames/second per video stream                    │
│                                                                   │
│  During COVID peak (April 2020):                                 │
│  • Went from 10M to 300M daily participants in 3 months!       │
│  • Scaled 30x in 90 days                                        │
│  • Used Oracle Cloud + AWS to handle overflow                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### How Zoom Scaled 30x During COVID

```
┌─────────────────────────────────────────────────────────────────┐
│         ZOOM'S COVID SCALING STORY                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  December 2019: 10 million daily participants                   │
│  March 2020:    200 million daily participants                  │
│  April 2020:    300 million daily participants                  │
│                                                                   │
│  Challenge: 30x traffic in 90 days                              │
│                                                                   │
│  How they did it:                                                │
│                                                                   │
│  1. HYBRID CLOUD: Added Oracle Cloud + AWS capacity             │
│     • Own data centers: handle base load                        │
│     • Cloud: handle overflow (burst capacity)                   │
│                                                                   │
│  2. GEOGRAPHIC EXPANSION: New DC regions activated              │
│     • Reduced latency for newly remote workers worldwide       │
│                                                                   │
│  3. ARCHITECTURE ADVANTAGE: SFU model scales linearly           │
│     • MCU: CPU cost = O(N²) per meeting (decode + mix)         │
│     • SFU: CPU cost = O(N) per meeting (just forwarding)       │
│     • SFU is 10-100x cheaper at scale                          │
│                                                                   │
│  4. BANDWIDTH MANAGEMENT: Aggressive quality adaptation         │
│     • Reduced default video quality during peak hours           │
│     • Free users limited to 40 min (reduces sustained load)    │
│                                                                   │
│  5. ENGINEERING DECISIONS MADE YEARS EARLIER:                    │
│     • Custom protocol (not WebRTC) = more control               │
│     • Own media servers (not third-party) = faster scaling     │
│     • Global private backbone = predictable performance         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Zoom vs. WebRTC-Based Solutions (Google Meet)

| Aspect | Zoom | Google Meet (WebRTC) |
|--------|------|---------------------|
| Protocol | Custom over UDP | Standard WebRTC |
| SFU | Custom C++ | Google's internal media servers |
| Noise suppression | On-device ML | On-device ML (recently added) |
| Network | Private backbone | Google's private backbone |
| Max participants | 1000 video | 500 video |
| Platform | Native apps + web | Web-first (browser-based) |

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Better Approach |
|---------|---------------|-----------------|
| Using TCP for real-time media | Retransmission adds 100ms+ delay per lost packet | Use UDP + FEC (accept some loss, reconstruct what you can) |
| MCU approach at scale | CPU cost is O(N²) — economically unfeasible for large meetings | SFU approach — just forward packets, no decode/encode |
| Fixed video quality for all | Users on 500 Kbps get buffering; users on 50 Mbps get blurry video | Simulcast + adaptive layer selection per receiver |
| Not implementing jitter buffer | Out-of-order packets → glitchy audio/video | Adaptive jitter buffer (trade latency for smoothness) |
| Sending all video streams at full quality | 20-person meeting × 720p = 30 Mbps download (impossible for most) | Active speaker detection + thumbnail quality for non-speakers |
| Public internet only | Internet routing is unpredictable; latency spikes are common | Private backbone between DCs for predictable inter-region latency |

---

## When to Use / When NOT to Use

### When to Build a Zoom-Like Architecture
- Real-time communication with < 200ms latency requirement
- Large group video calls (5+ participants)
- You need full control over media quality and routing
- Building a product where video/audio IS the core product
- Operating at scale (millions of concurrent meetings)

### When NOT to Build This
- 1-on-1 video calls only → WebRTC peer-to-peer is sufficient
- Pre-recorded video → Use streaming architecture (Netflix model)
- Audio-only podcast → Much simpler; standard WebRTC or SIP
- Small scale (< 10K concurrent users) → Use Twilio, Agora, or Daily.co
- Budget/team constraints → Use WebRTC + open-source SFU (mediasoup, Janus)

---

## Key Takeaways

1. **SFU (Selective Forwarding Unit)** is the architecture of choice for modern video conferencing — it routes packets without decoding, making it O(N) not O(N²) cost per meeting
2. **Simulcast** lets each sender upload 2-3 quality layers simultaneously; the SFU picks which layer to forward to each receiver based on bandwidth and speaking status
3. **UDP over TCP** is mandatory for real-time: retransmission delays (TCP) are worse than lost packets (UDP + FEC can reconstruct)
4. **Adaptive jitter buffer** smooths out network variability — bigger buffer = smoother but more delay; it auto-adjusts based on conditions
5. **Active speaker detection** saves massive bandwidth — only the current speaker gets high-quality video forwarded; everyone else gets thumbnails
6. **Private backbone networks** give Zoom/Google Meet predictable inter-DC latency that the public internet can't guarantee
7. **Audio quality > Video quality** in perception — users tolerate pixelated video but NOT choppy audio. Always prioritize audio bandwidth.

---

## What's Next?

Next, we'll explore [How Instagram Handles Photo Uploads at Scale](./10-instagram-photos.md) — understanding how they process billions of photo uploads, generate multiple resized versions, apply filters, and serve images to 2 billion users from a global CDN.
