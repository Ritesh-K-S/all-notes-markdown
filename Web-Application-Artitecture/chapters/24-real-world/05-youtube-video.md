# How YouTube/TikTok Serves Billions of Videos

> **What you'll learn**: How YouTube handles 500+ hours of video uploaded every minute, stores over 1 billion hours of content, and serves 1 billion hours of watch time daily — while TikTok optimizes for short-form, infinite-scroll video with an AI-first recommendation engine.

---

## Real-Life Analogy

Imagine you're running a **massive video rental store** — but with these mind-bending requirements:

- **500 hours of new movies arrive every single minute** (you need to process and shelve them all)
- You have **2 billion members** who each watch different content
- Every movie needs to be available in **20 different quality levels** (VHS, DVD, Blu-ray, 4K, etc.)
- You need to instantly recommend the **perfect next movie** for each of your 2 billion members
- Your store has **branches in every city on Earth**, each with copies of the most popular movies
- Oh, and some movies are 10 seconds long (TikTok), while others are 12 hours long (YouTube)

This is the engineering challenge YouTube and TikTok solve every second of every day.

---

## Core Concept Explained Step-by-Step

### Step 1: The Upload Pipeline — Ingesting 500 Hours/Minute

```
Creator uploads a 4K video (10 GB file)
         │
         ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 1: UPLOAD & CHUNKED TRANSFER                                  │
│ • Video split into chunks (resumable upload)                       │
│ • Uploaded to nearest Google data center                           │
│ • Checksum verification (no corruption)                            │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 2: CONTENT PROCESSING PIPELINE                                │
│                                                                     │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐       │
│  │ Virus/Malware  │  │ Copyright      │  │ Content        │       │
│  │ Scan           │  │ Check          │  │ Moderation     │       │
│  │                │  │ (Content ID)   │  │ (AI + Human)   │       │
│  └────────────────┘  └────────────────┘  └────────────────┘       │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 3: TRANSCODING (The heavy computation)                        │
│                                                                     │
│  Original 4K → Encoded into MANY versions:                         │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Resolution  │  Codec   │  Bitrate    │  Use Case        │     │
│  │──────────────┼──────────┼─────────────┼──────────────────│     │
│  │  144p        │  H.264   │  100 kbps   │  Very slow 2G    │     │
│  │  240p        │  H.264   │  300 kbps   │  Slow mobile     │     │
│  │  360p        │  H.264   │  700 kbps   │  Basic mobile    │     │
│  │  480p        │  VP9     │  1.5 Mbps   │  Standard        │     │
│  │  720p        │  VP9     │  3 Mbps     │  HD              │     │
│  │  1080p       │  VP9     │  5 Mbps     │  Full HD         │     │
│  │  1440p       │  VP9     │  10 Mbps    │  2K              │     │
│  │  2160p (4K)  │  VP9/AV1 │  20 Mbps    │  4K TV           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                     │
│  Result: 1 upload → 20+ encoded files + audio tracks + subtitles  │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 4: STORAGE & CDN DISTRIBUTION                                 │
│ • All versions stored in Google's distributed storage (Colossus)  │
│ • Popular videos pre-pushed to edge caches worldwide              │
│ • Video goes LIVE — available for viewing                          │
└────────────────────────────────────────────────────────────────────┘
```

### Step 2: Watching a Video — The Playback Flow

```
User clicks on a video
         │
         ▼
┌──────────────────────────────────────────────────────────────────┐
│ 1. API REQUEST → YouTube Backend (Google Cloud)                   │
│    Returns: video metadata, CDN URL, available qualities         │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│ 2. VIDEO PLAYER INITIALIZATION                                    │
│    • Checks available bandwidth                                   │
│    • Selects initial quality (usually 480p to start fast)        │
│    • Requests first few chunks from CDN                           │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│ 3. ADAPTIVE STREAMING (DASH / HLS)                                │
│                                                                    │
│    Video file structure:                                          │
│    ┌────────┬────────┬────────┬────────┬────────┐               │
│    │Chunk 1 │Chunk 2 │Chunk 3 │Chunk 4 │Chunk 5 │ ...           │
│    │ 2-5 sec│ 2-5 sec│ 2-5 sec│ 2-5 sec│ 2-5 sec│               │
│    └────────┴────────┴────────┴────────┴────────┘               │
│                                                                    │
│    Each chunk available in ALL quality levels                      │
│    Player can switch quality between ANY two chunks               │
│                                                                    │
│    Chunk 1 (480p) → Chunk 2 (720p) → Chunk 3 (1080p) → ...     │
│    (bandwidth improved, so quality ramped up)                      │
└──────────────────────────────────────────────────────────────────┘
```

### Step 3: YouTube's CDN — Google's Global Edge Network

```
┌─────────────────────────────────────────────────────────────────────┐
│               YOUTUBE CDN ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│              ┌─────────────────────────┐                             │
│              │    Origin Storage       │                             │
│              │   (Google Colossus)     │                             │
│              │   ALL videos stored     │                             │
│              └───────────┬─────────────┘                             │
│                          │                                           │
│         Popular videos pushed to edge during off-peak                │
│                          │                                           │
│       ┌──────────────────┼──────────────────────┐                   │
│       ▼                  ▼                      ▼                   │
│  ┌──────────┐     ┌──────────┐          ┌──────────┐              │
│  │ Edge POP │     │ Edge POP │          │ Edge POP │              │
│  │ Mumbai   │     │ New York │          │ London   │              │
│  │          │     │          │          │          │              │
│  │ Cache:   │     │ Cache:   │          │ Cache:   │              │
│  │ Popular  │     │ Popular  │          │ Popular  │              │
│  │ videos   │     │ videos   │          │ videos   │              │
│  │ in India │     │ in US    │          │ in UK    │              │
│  └────┬─────┘     └────┬─────┘          └────┬─────┘              │
│       │                 │                     │                     │
│       ▼                 ▼                     ▼                     │
│  ┌─────────┐     ┌─────────┐          ┌─────────┐                 │
│  │ Users   │     │ Users   │          │ Users   │                 │
│  │ India   │     │ US      │          │ UK      │                 │
│  └─────────┘     └─────────┘          └─────────┘                 │
│                                                                      │
│  Cache HIT (popular video): Served from local edge (~10ms)          │
│  Cache MISS (rare video): Fetched from origin (~100-200ms)          │
│                                                                      │
│  YouTube has 100+ edge locations across 70+ countries               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 4: TikTok's Recommendation Engine — The For You Page

TikTok's architecture differs significantly — it's **recommendation-first**:

```
┌─────────────────────────────────────────────────────────────────────┐
│           TIKTOK "FOR YOU" PAGE ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  User opens TikTok → Infinite scroll of personalized videos        │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │              SIGNALS COLLECTED                            │      │
│  │                                                          │      │
│  │  • Watch time (most important: did you watch 100%?)     │      │
│  │  • Re-watches (watched twice = STRONG signal)           │      │
│  │  • Shares (shared = very high interest)                 │      │
│  │  • Comments, likes, follows                             │      │
│  │  • Scroll speed (scrolled fast past = not interested)   │      │
│  │  • Video completion rate                                │      │
│  │  • Account age, device type, location, time of day      │      │
│  └──────────────────────────────────────────────────────────┘      │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │         RECOMMENDATION MODEL                             │      │
│  │                                                          │      │
│  │  Two-tower architecture:                                │      │
│  │                                                          │      │
│  │  ┌────────────┐              ┌────────────────┐         │      │
│  │  │ User Tower │              │ Content Tower  │         │      │
│  │  │            │              │                │         │      │
│  │  │ User       │   Dot       │ Video          │         │      │
│  │  │ embeddings │ ◄─Product──▶│ embeddings     │         │      │
│  │  │ (interests,│   (Score)   │ (visual, audio,│         │      │
│  │  │  behavior) │              │  text, tags)   │         │      │
│  │  └────────────┘              └────────────────┘         │      │
│  │                                                          │      │
│  │  Result: Score for every video for this user            │      │
│  │  Top N videos served in the feed                        │      │
│  └──────────────────────────────────────────────────────────┘      │
│                          │                                          │
│                          ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │         DIVERSIFICATION & BUSINESS RULES                 │      │
│  │                                                          │      │
│  │  • Don't show same creator twice in a row               │      │
│  │  • Mix in "exploration" videos (discover new interests) │      │
│  │  • Insert ads at natural positions                      │      │
│  │  • Content policy enforcement                           │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### YouTube's Transcoding at Scale

YouTube transcodes **500 hours of video every minute**. This requires enormous compute:

```
┌─────────────────────────────────────────────────────────────────┐
│           YOUTUBE TRANSCODING PIPELINE (Cosmos)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Input: One 4K video (10 GB, 10 minutes)                        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Step 1: Split into segments (GOP-aligned)               │    │
│  │                                                         │    │
│  │  [Seg 1: 0-2s] [Seg 2: 2-4s] [Seg 3: 4-6s] ...       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Step 2: Parallel transcode (distributed MapReduce)      │    │
│  │                                                         │    │
│  │  Worker 1: Seg 1 → 144p, 240p, 360p, 480p...          │    │
│  │  Worker 2: Seg 2 → 144p, 240p, 360p, 480p...          │    │
│  │  Worker 3: Seg 3 → 144p, 240p, 360p, 480p...          │    │
│  │  ... (thousands of workers in parallel)                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Step 3: Reassemble & package                            │    │
│  │                                                         │    │
│  │  • DASH manifest (for web/Android)                      │    │
│  │  • HLS manifest (for iOS/Safari)                        │    │
│  │  • Generate thumbnails (AI-selected best frames)        │    │
│  │  • Extract audio tracks (multi-language if available)   │    │
│  │  • Auto-generate subtitles (speech-to-text ML)          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Total output: 1 video × 8 qualities × 2 codecs = 16+ files    │
│  + audio + subtitles + thumbnails = 20-30 files per upload      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Content ID — Copyright Detection at Scale

```
┌─────────────────────────────────────────────────────────────────┐
│              CONTENT ID SYSTEM                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  How YouTube detects copyrighted content in uploaded videos:     │
│                                                                   │
│  ┌────────────────────────────────────────────────────┐          │
│  │  Reference Database                                 │          │
│  │  • 100M+ reference files from copyright owners     │          │
│  │  • Audio fingerprints + video fingerprints          │          │
│  │  • Stored as compact hashes (~few KB per hour)      │          │
│  └─────────────────────────┬──────────────────────────┘          │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────┐          │
│  │  For each uploaded video:                           │          │
│  │                                                    │          │
│  │  1. Generate audio fingerprint                     │          │
│  │  2. Generate video fingerprint (visual features)   │          │
│  │  3. Compare against 100M+ references              │          │
│  │  4. If match found:                               │          │
│  │     • Block video, OR                             │          │
│  │     • Allow but monetize for copyright owner, OR  │          │
│  │     • Allow with tracking                         │          │
│  └────────────────────────────────────────────────────┘          │
│                                                                   │
│  Processing: Every minute of uploaded video scanned             │
│  against every second of 100M+ reference files                  │
│  Accuracy: 99.7%+ (handles pitch shifts, speed changes)         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### YouTube's Data Storage — The Scale is Insane

```
┌─────────────────────────────────────────────────────────────────┐
│              YOUTUBE STORAGE NUMBERS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  • 800 million+ videos on the platform                          │
│  • 500 hours uploaded every minute = 720,000 hours/day          │
│  • Each video stored in ~20 versions                            │
│  • Estimated total storage: 1+ exabyte (1,000 petabytes)        │
│                                                                   │
│  Storage strategy:                                               │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  HOT TIER (SSD/RAM Cache)                           │        │
│  │  • Top 1% most-watched videos                       │        │
│  │  • Viral content, trending, new releases            │        │
│  │  • ~80% of all views served from here               │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  WARM TIER (HDD)                                    │        │
│  │  • Videos with moderate viewership                  │        │
│  │  • ~15% of views                                    │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────┐        │
│  │  COLD TIER (Tape/Archive)                           │        │
│  │  • Rarely watched videos (long tail)                │        │
│  │  • Still accessible but with slightly higher latency │        │
│  │  • ~5% of views but ~70% of stored content          │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                   │
│  Power law: 1% of videos account for 80% of views              │
│  (But ALL videos must remain accessible — that's the promise)    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Video Chunking for Adaptive Streaming

```python
# Simplified DASH-style video chunking for adaptive streaming
import os
from dataclasses import dataclass
from typing import List, Dict

@dataclass
class VideoChunk:
    segment_number: int
    start_time_sec: float
    duration_sec: float
    quality: str
    file_path: str
    size_bytes: int

class AdaptiveStreamingManifest:
    """
    Generates a DASH-like manifest for adaptive bitrate streaming.
    YouTube uses this to let players switch quality mid-stream.
    """
    
    QUALITY_PROFILES = {
        "144p":  {"width": 256,  "height": 144,  "bitrate_kbps": 100},
        "360p":  {"width": 640,  "height": 360,  "bitrate_kbps": 700},
        "720p":  {"width": 1280, "height": 720,  "bitrate_kbps": 3000},
        "1080p": {"width": 1920, "height": 1080, "bitrate_kbps": 5000},
        "4K":    {"width": 3840, "height": 2160, "bitrate_kbps": 20000},
    }
    
    def __init__(self, video_id: str, duration_sec: float, chunk_duration: float = 4.0):
        self.video_id = video_id
        self.duration = duration_sec
        self.chunk_duration = chunk_duration
        self.chunks: Dict[str, List[VideoChunk]] = {}
    
    def generate_manifest(self) -> dict:
        """Generate manifest listing all available chunks at all qualities."""
        manifest = {
            "video_id": self.video_id,
            "duration": self.duration,
            "representations": []
        }
        
        num_chunks = int(self.duration / self.chunk_duration) + 1
        
        for quality, profile in self.QUALITY_PROFILES.items():
            segments = []
            for i in range(num_chunks):
                start = i * self.chunk_duration
                # URL pattern: /video/{id}/{quality}/segment_{n}.mp4
                url = f"/video/{self.video_id}/{quality}/seg_{i}.m4s"
                segments.append({"number": i, "start": start, "url": url})
            
            manifest["representations"].append({
                "quality": quality,
                "width": profile["width"],
                "height": profile["height"],
                "bitrate_kbps": profile["bitrate_kbps"],
                "segments": segments
            })
        
        return manifest

# Usage: Generate manifest for a 10-minute video
manifest = AdaptiveStreamingManifest("abc123", duration_sec=600)
result = manifest.generate_manifest()
# Player uses this to know which URLs to fetch for each quality/chunk
```

### Java — Video Upload with Resumable Chunks

```java
import java.security.MessageDigest;
import java.util.*;

/**
 * Simplified resumable upload handler — similar to YouTube's upload API.
 * Videos are uploaded in chunks so that network interruptions
 * don't require re-uploading the entire file.
 */
public class ResumableUploadHandler {
    private static final int CHUNK_SIZE = 5 * 1024 * 1024; // 5MB chunks
    
    // uploadId → upload state
    private final Map<String, UploadState> uploads = new HashMap<>();

    public String initiateUpload(String userId, String filename, long totalSize) {
        String uploadId = UUID.randomUUID().toString();
        int totalChunks = (int) Math.ceil((double) totalSize / CHUNK_SIZE);
        
        uploads.put(uploadId, new UploadState(
            uploadId, userId, filename, totalSize, totalChunks
        ));
        
        return uploadId;  // Client uses this to send chunks
    }

    public UploadResult uploadChunk(String uploadId, int chunkNumber, 
                                     byte[] data, String checksum) {
        UploadState state = uploads.get(uploadId);
        if (state == null) return UploadResult.error("Unknown upload");
        
        // Verify chunk integrity
        String computedChecksum = computeMD5(data);
        if (!computedChecksum.equals(checksum)) {
            return UploadResult.error("Checksum mismatch — resend chunk");
        }
        
        // Store chunk (in production: write to distributed storage)
        state.receivedChunks.add(chunkNumber);
        state.bytesReceived += data.length;
        
        // Check if upload is complete
        if (state.receivedChunks.size() == state.totalChunks) {
            // Trigger transcoding pipeline
            triggerTranscoding(state);
            uploads.remove(uploadId);
            return UploadResult.complete("Upload complete — processing started");
        }
        
        return UploadResult.partial(
            "Chunk " + chunkNumber + "/" + state.totalChunks + " received"
        );
    }

    public int getResumePosition(String uploadId) {
        // Client asks: "Where did I leave off?" (after network failure)
        UploadState state = uploads.get(uploadId);
        return state != null ? state.receivedChunks.size() : 0;
    }

    private void triggerTranscoding(UploadState state) {
        // In production: publish event to transcoding queue
        System.out.println("Transcoding started for: " + state.filename);
    }

    private String computeMD5(byte[] data) {
        try {
            return Base64.getEncoder().encodeToString(
                MessageDigest.getInstance("MD5").digest(data));
        } catch (Exception e) { return ""; }
    }

    static class UploadState {
        String uploadId, userId, filename;
        long totalSize, bytesReceived;
        int totalChunks;
        Set<Integer> receivedChunks = new TreeSet<>();
        
        UploadState(String id, String user, String file, long size, int chunks) {
            this.uploadId = id; this.userId = user; this.filename = file;
            this.totalSize = size; this.totalChunks = chunks;
        }
    }

    record UploadResult(String status, String message) {
        static UploadResult complete(String msg) { return new UploadResult("complete", msg); }
        static UploadResult partial(String msg) { return new UploadResult("partial", msg); }
        static UploadResult error(String msg) { return new UploadResult("error", msg); }
    }
}
```

---

## Infrastructure Examples

### YouTube's Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Storage** | Colossus (GFS v2) | Video file storage (exabytes) |
| **CDN** | Google Global Cache (GGC) | Edge caching in ISPs |
| **Database** | Bigtable, Spanner | Metadata, comments, analytics |
| **Cache** | Memcache / custom | Hot video metadata |
| **Transcoding** | Custom (Borg-scheduled) | Video encoding at scale |
| **Streaming** | DASH + HLS | Adaptive bitrate delivery |
| **ML** | TensorFlow on TPUs | Recommendations, content moderation |
| **Search** | Custom (Google infrastructure) | Video search |
| **Analytics** | Dremel / BigQuery | Creator analytics, ads |

### TikTok's Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **CDN** | Custom + third-party (Akamai, CloudFront) | Global video delivery |
| **Cloud** | Multi-cloud (AWS, GCP, own DCs) | Compute and storage |
| **Database** | MySQL (sharded), Redis | User data, video metadata |
| **Cache** | Redis clusters | Feed pre-computation |
| **ML** | Custom GPU clusters | Recommendation engine |
| **Storage** | Object storage | Video files |
| **Messaging** | Kafka | Event streaming |
| **Processing** | Apache Flink | Real-time analytics |

---

## Real-World Example

### YouTube — Handling Viral Videos

When a video goes viral (millions of views in hours):

```
┌─────────────────────────────────────────────────────────────────┐
│              HANDLING VIRAL CONTENT                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Normal video: 100 views/day                                     │
│  Viral video: 10,000,000 views in first hour                    │
│                                                                   │
│  YouTube's response (automatic):                                 │
│                                                                   │
│  T+0 min: Video uploaded, transcoded, stored at origin           │
│  T+5 min: First 1000 views → stays on origin servers            │
│  T+30 min: 100K views → promoted to regional edge caches        │
│  T+60 min: 1M views → pushed to ALL global edge caches          │
│  T+2 hrs: 10M views → multiple copies at each edge              │
│                                                                   │
│  The CDN AUTOMATICALLY detects popularity and caches more        │
│  aggressively. No human intervention needed.                     │
│                                                                   │
│  ┌─────────────────────────────────────────────────┐             │
│  │  Popularity Detection Algorithm:                 │             │
│  │                                                 │             │
│  │  views_per_minute > threshold_1 → cache at edge │             │
│  │  views_per_minute > threshold_2 → multi-copy    │             │
│  │  views_per_minute > threshold_3 → priority QoS  │             │
│  └─────────────────────────────────────────────────┘             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### TikTok — Cold Start Problem

How TikTok recommends videos to brand-new users (no history):

1. **First 5 videos**: Show globally popular content (safe bet)
2. **Watch signals**: User watches 3 seconds of video A, skips B, watches all of C
3. **After 10 videos**: System already has enough signal to start personalizing
4. **After 50 videos**: Feed is highly personalized (TikTok's "addiction" kicks in)

TikTok's advantage: Short videos (15-60s) generate feedback signals **10x faster** than YouTube's long videos. This is why TikTok personalizes faster.

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Better Approach |
|---------|---------------|-----------------|
| Storing only one version of each video | Users on slow connections get terrible experience | Transcode into 8+ quality levels for adaptive streaming |
| Transcoding synchronously during upload | User waits minutes/hours before video is available | Async pipeline: accept upload immediately, transcode in background |
| Using general-purpose CDN for video | Not optimized for large file streaming | Dedicated video CDN with range-request support and edge caching |
| Single monolithic recommendation model | Can't iterate quickly, slow to adapt | Multiple specialized models combined (retrieval → ranking → re-ranking) |
| No resumable upload support | Users on mobile lose progress when network drops | Chunked resumable uploads with server-side state |
| Treating all videos equally in storage | Wastes expensive storage on rarely-watched content | Tiered storage: hot (SSD) → warm (HDD) → cold (archive) |

---

## When to Use / When NOT to Use

### When to Build a YouTube-Like Architecture
- Serving user-generated video at scale (millions of videos)
- Diverse device/bandwidth requirements (phone to TV)
- Need for adaptive streaming (varying network conditions)
- Content discovery through recommendations is core product
- Long-tail content matters (ALL videos accessible, not just popular ones)

### When NOT to Build This
- Live streaming only (different architecture — lower latency)
- Short-form only with limited catalog → Simpler approach works
- Enterprise video hosting (< 10K videos) → Use Vimeo/Wistia/Mux
- Real-time communication → WebRTC architecture (see Zoom chapter)
- Budget constraints → Use AWS MediaConvert + CloudFront or Cloudflare Stream

---

## Key Takeaways

1. **500 hours uploaded per minute** requires a massively parallel transcoding pipeline — each video is split into segments and processed across thousands of workers simultaneously
2. **Adaptive bitrate streaming (DASH/HLS)** lets players switch quality mid-stream without buffering — videos are stored as 2-5 second chunks at multiple quality levels
3. **Tiered storage** is essential: 1% of videos get 80% of views. Keep hot content on SSDs near users; archive the long tail to cold storage.
4. **YouTube's CDN** places cache servers inside ISPs globally (Google Global Cache) — 90% of views served without crossing the internet
5. **TikTok's recommendation engine** is the product itself — the "For You" page uses a two-tower neural network model trained on watch-time signals, with feedback loops 10x faster than long-form video
6. **Content ID** scans every upload against 100M+ reference files using audio/video fingerprinting — a search engine over multimedia
7. **Resumable chunked uploads** are critical for user experience — network drops shouldn't mean re-uploading a 10GB file from scratch

---

## What's Next?

Next, we'll explore [How Twitter/X Handles the Firehose of Tweets](./06-twitter-timeline.md) — understanding how they handle 500 million tweets per day, build personalized timelines for hundreds of millions of users, and manage the "thundering herd" problem during breaking news events.
