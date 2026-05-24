# How Instagram Handles Photo Uploads at Scale

> **What you'll learn**: How Instagram processes 100+ million photo/video uploads daily, generates multiple resized versions, applies filters in real-time, serves images to 2 billion monthly active users, and manages one of the largest Django deployments in the world — all while maintaining sub-second upload experiences.

---

## Real-Life Analogy

Imagine you run the world's largest **photo printing and distribution service**:

- **100 million new photos arrive every day** from customers worldwide
- For each photo, you need to create **6 different sizes** (wallet, postcard, poster, billboard, etc.)
- Some customers want **filters** applied (like vintage, black-and-white, brightness adjustments)
- You need to **store** these photos forever (nobody deletes anything)
- When ANY of your **2 billion customers** wants to see a photo, you must deliver it **instantly** — no matter where they are on Earth
- Your **feed** shows each customer a personalized selection of photos from people they follow, ranked by what they'll find most interesting

Instagram handles this at a scale where they store **tens of billions of photos** and serve them billions of times per day.

---

## Core Concept Explained Step-by-Step

### Step 1: What Happens When You Upload a Photo

```
User takes photo, applies filter, writes caption, taps "Share"
         │
         ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 1: CLIENT-SIDE PROCESSING                                      │
│ • Filter applied ON DEVICE (no server needed for filters)          │
│ • Image compressed (JPEG/HEIC, quality ~80%)                       │
│ • Metadata attached (caption, location, tags, alt text)            │
│ • Photo uploaded to nearest edge/CDN endpoint                       │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 2: UPLOAD TO OBJECT STORAGE                                    │
│ • Original high-res image stored in S3-compatible storage          │
│ • Upload ID returned immediately (user sees "uploading...")        │
│ • Client gets success response in < 1 second                       │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 3: ASYNC PROCESSING PIPELINE (background)                     │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │ Generate    │  │ Content     │  │ EXIF Data   │               │
│  │ Thumbnails  │  │ Moderation  │  │ Processing  │               │
│  │ (6 sizes)   │  │ (AI scan)   │  │ (GPS, etc.) │               │
│  └─────────────┘  └─────────────┘  └─────────────┘               │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │ Face        │  │ Object      │  │ Alt Text    │               │
│  │ Detection   │  │ Recognition │  │ Generation  │               │
│  │ (tagging)   │  │ (search)    │  │ (a11y)      │               │
│  └─────────────┘  └─────────────┘  └─────────────┘               │
│                                                                     │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 4: FEED DISTRIBUTION (fan-out)                                 │
│ • Notify followers' feed services about new post                   │
│ • Push notification to close friends (if enabled)                  │
│ • Update search/explore indexes                                    │
│ • Photo appears in followers' feeds within seconds                 │
└────────────────────────────────────────────────────────────────────┘
```

### Step 2: Image Resizing — Why Multiple Versions?

```
┌─────────────────────────────────────────────────────────────────┐
│         IMAGE VERSIONS GENERATED PER UPLOAD                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Original: 4032 × 3024 (iPhone 14 Pro, ~8 MB HEIC)             │
│                                                                   │
│  Generated versions:                                             │
│  ┌──────────────────────────────────────────────────────┐       │
│  │  Size Name      │  Dimensions  │  Use Case           │       │
│  │─────────────────┼──────────────┼─────────────────────│       │
│  │  thumbnail_tiny │  150 × 150   │  User avatar, grid  │       │
│  │  thumbnail      │  320 × 320   │  Feed grid (3-col)  │       │
│  │  medium         │  640 × 640   │  Feed (mobile)      │       │
│  │  large          │  1080 × 1080 │  Feed (tablet/web)  │       │
│  │  full           │  1440 × 1440 │  Zoomed view        │       │
│  │  original       │  4032 × 3024 │  Download/backup    │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                   │
│  Why? Serving a 8 MB original to someone viewing a               │
│  150px thumbnail wastes 99% of bandwidth!                        │
│                                                                   │
│  Each version stored separately in object storage.               │
│  CDN serves the RIGHT size based on device/screen.               │
│                                                                   │
│  Storage per photo: ~1.5 MB total (all versions compressed)      │
│  Total storage: billions of photos × 1.5 MB = PETABYTES         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: The Feed — Ranked, Not Chronological

```
┌─────────────────────────────────────────────────────────────────────┐
│              INSTAGRAM FEED ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  User opens Instagram → "Show me my feed"                           │
│         │                                                            │
│         ▼                                                            │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │  CANDIDATE GENERATION                                     │      │
│  │  Gather recent posts from:                               │      │
│  │  • People you follow (last 48-72 hours)                  │      │
│  │  • Suggested posts (from Explore system)                 │      │
│  │  • Ads (from ads auction)                                │      │
│  │  Result: ~500 candidate posts                            │      │
│  └──────────────────────────┬───────────────────────────────┘      │
│                             │                                        │
│                             ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │  RANKING MODEL (ML)                                       │      │
│  │                                                          │      │
│  │  For each post, predict probability of:                  │      │
│  │  • P(like)     — will you like it?                      │      │
│  │  • P(comment)  — will you comment?                      │      │
│  │  • P(share)    — will you share/save?                   │      │
│  │  • P(linger)   — will you pause and look at it?        │      │
│  │  • P(see_more) — will you tap "more" on caption?       │      │
│  │                                                          │      │
│  │  Score = w1×P(like) + w2×P(comment) + w3×P(share)       │      │
│  │        + w4×P(linger) + w5×P(see_more)                  │      │
│  │        - penalty_for_low_quality                         │      │
│  └──────────────────────────┬───────────────────────────────┘      │
│                             │                                        │
│                             ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │  POST-PROCESSING                                          │      │
│  │  • Diversity: Don't show 5 posts from same person        │      │
│  │  • Freshness: Boost newer posts slightly                 │      │
│  │  • Type variety: Mix photos, videos, carousels, reels   │      │
│  │  • Insert ads at natural positions                       │      │
│  │  • Show unseen posts before seen ones                    │      │
│  └──────────────────────────────────────────────────────────┘      │
│                             │                                        │
│                             ▼                                        │
│  User sees ~20-30 posts (loads more on scroll)                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 4: CDN — Serving Images Globally

```
┌─────────────────────────────────────────────────────────────────┐
│              IMAGE SERVING ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Client requests: GET /p/abc123/media/?size=medium               │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────┐                        │
│  │  CDN Edge (Facebook/Meta CDN)        │                        │
│  │  • Is this image in edge cache? ─── YES ──▶ Return (1ms)     │
│  │                     │                                          │
│  │                    NO                                          │
│  │                     │                                          │
│  └─────────────────────┼────────────────┘                        │
│                        │                                          │
│                        ▼                                          │
│  ┌──────────────────────────────────────┐                        │
│  │  Origin Shield (regional cache)       │                        │
│  │  • Is it in regional cache? ──── YES ──▶ Return + cache edge │
│  │                     │                                          │
│  │                    NO                                          │
│  └─────────────────────┼────────────────┘                        │
│                        │                                          │
│                        ▼                                          │
│  ┌──────────────────────────────────────┐                        │
│  │  Object Storage (Origin)             │                        │
│  │  • Fetch from storage                │                        │
│  │  • Cache at shield + edge            │                        │
│  │  • Return to user                    │                        │
│  └──────────────────────────────────────┘                        │
│                                                                   │
│  Cache hit rate: 90%+ (most popular images served from edge)    │
│  Unique photos requested daily: Billions                        │
│  Total stored: Tens of billions of images                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Instagram's Tech Stack — A Django Success Story

Instagram is famous for being one of the largest **Python/Django** deployments in the world:

```
┌─────────────────────────────────────────────────────────────────────┐
│              INSTAGRAM'S ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                    CLIENT APPS                             │      │
│  │     iOS App     │    Android App     │    Web App         │      │
│  └──────────────────────────┬───────────────────────────────┘      │
│                             │                                        │
│                             ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │            LOAD BALANCER (Meta's LB)                      │      │
│  └──────────────────────────┬───────────────────────────────┘      │
│                             │                                        │
│                             ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │         DJANGO WEB SERVERS (hundreds of instances)        │      │
│  │         Python 3.x + Django + uWSGI                      │      │
│  │         The world's largest Django deployment!           │      │
│  └─────────────┬───────────┬────────────┬───────────────────┘      │
│                │           │            │                            │
│       ┌────────┘    ┌──────┘     ┌──────┘                          │
│       ▼             ▼            ▼                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐                   │
│  │PostgreSQL│ │ Cassandra│ │     Redis         │                   │
│  │(sharded) │ │(feed,    │ │   (cache,         │                   │
│  │          │ │ activity)│ │    sessions,       │                   │
│  │Users,    │ │          │ │    counters)       │                   │
│  │posts,    │ │          │ │                    │                   │
│  │follows   │ │          │ │                    │                   │
│  └──────────┘ └──────────┘ └──────────────────┘                   │
│       │                          │                                  │
│       │          ┌───────────────┘                                  │
│       ▼          ▼                                                  │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │              STORAGE & MEDIA LAYER                         │      │
│  │                                                          │      │
│  │  • Object Storage (Photos, Videos) — Petabytes          │      │
│  │  • CDN (Facebook/Meta Global CDN)                        │      │
│  │  • Media Processing (resize, transcode)                  │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │              ASYNC TASK LAYER                              │      │
│  │                                                          │      │
│  │  • Celery (task queue for Python)                        │      │
│  │  • RabbitMQ / Kafka (message broker)                     │      │
│  │  • Tasks: resize images, send notifications, update     │      │
│  │    counters, run ML models, fan-out to feeds             │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### PostgreSQL Sharding — How Instagram Scales Their Primary DB

```
┌─────────────────────────────────────────────────────────────────┐
│              INSTAGRAM'S POSTGRESQL SHARDING                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Strategy: Shard by user_id (all data for one user on same shard)│
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Shard 1 (user_ids 0-999999)                             │   │
│  │  • Users table rows for these users                      │   │
│  │  • Posts table rows for these users                      │   │
│  │  • Comments made BY these users                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Shard 2 (user_ids 1000000-1999999)                      │   │
│  │  • Users table rows for these users                      │   │
│  │  • Posts table rows for these users                      │   │
│  │  • Comments made BY these users                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ... thousands of shards ...                                     │
│                                                                   │
│  Benefits:                                                        │
│  • User's profile + posts + activity all on same shard          │
│  • No cross-shard queries for common operations                  │
│  • Each shard: ~1TB, fits in memory with replicas               │
│                                                                   │
│  ID Generation (Instagram's custom approach):                    │
│  • 41 bits: timestamp (ms since custom epoch)                   │
│  • 13 bits: shard ID (which logical shard)                      │
│  • 10 bits: auto-increment within shard                         │
│  (Similar to Twitter's Snowflake — see Chapter 24.6)            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### The Explore Page — Content Discovery

```
┌─────────────────────────────────────────────────────────────────┐
│              EXPLORE PAGE ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Challenge: Show interesting content from people you             │
│  DON'T follow (discovery, not just your network)                │
│                                                                   │
│  ┌────────────────────────────────────────────────────┐         │
│  │  Step 1: SEED ACCOUNTS                              │         │
│  │  Find accounts similar to ones you engage with     │         │
│  │  "Users who liked posts from X also liked Y"       │         │
│  └─────────────────────────┬──────────────────────────┘         │
│                            │                                     │
│                            ▼                                     │
│  ┌────────────────────────────────────────────────────┐         │
│  │  Step 2: CANDIDATE POSTS                            │         │
│  │  Get top posts from seed accounts + topics          │         │
│  │  Filter: recent, high engagement, not seen before  │         │
│  │  ~10,000 candidates                                │         │
│  └─────────────────────────┬──────────────────────────┘         │
│                            │                                     │
│                            ▼                                     │
│  ┌────────────────────────────────────────────────────┐         │
│  │  Step 3: FIRST-PASS RANKING (lightweight model)    │         │
│  │  Score all 10,000 candidates quickly               │         │
│  │  Keep top 500                                      │         │
│  └─────────────────────────┬──────────────────────────┘         │
│                            │                                     │
│                            ▼                                     │
│  ┌────────────────────────────────────────────────────┐         │
│  │  Step 4: SECOND-PASS RANKING (heavy model)         │         │
│  │  Deep neural network scores top 500                │         │
│  │  Consider: visual content, user history, engagement│         │
│  │  Keep top 50 for display                           │         │
│  └─────────────────────────┬──────────────────────────┘         │
│                            │                                     │
│                            ▼                                     │
│  ┌────────────────────────────────────────────────────┐         │
│  │  Step 5: INTEGRITY FILTER                           │         │
│  │  Remove borderline content                         │         │
│  │  Remove spam, misinformation, policy violations    │         │
│  │  Final: ~25 posts displayed on first load          │         │
│  └────────────────────────────────────────────────────┘         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Image Processing Pipeline with Celery

```python
# Instagram-style image processing pipeline using Celery (async tasks)
from PIL import Image
import io
import hashlib
from dataclasses import dataclass
from typing import Dict, List, Tuple

@dataclass
class ImageVersion:
    name: str
    max_width: int
    max_height: int
    quality: int  # JPEG quality (1-100)

class ImageProcessingPipeline:
    """
    Instagram's image processing: upload → generate multiple sizes → store.
    In production: runs as Celery tasks on hundreds of workers.
    """
    
    VERSIONS: List[ImageVersion] = [
        ImageVersion("thumbnail_tiny", 150, 150, 70),
        ImageVersion("thumbnail", 320, 320, 75),
        ImageVersion("medium", 640, 640, 80),
        ImageVersion("large", 1080, 1080, 85),
        ImageVersion("full", 1440, 1440, 90),
    ]
    
    def __init__(self, storage_client):
        self.storage = storage_client
    
    def process_upload(self, image_bytes: bytes, post_id: str) -> Dict[str, str]:
        """
        Main entry point: process uploaded image.
        Returns dict of {version_name: storage_url}.
        In production: This is a Celery task (@app.task decorator).
        """
        # Generate content hash (for deduplication)
        content_hash = hashlib.sha256(image_bytes).hexdigest()[:16]
        
        # Open original image
        img = Image.open(io.BytesIO(image_bytes))
        
        # Store original
        original_path = f"photos/{post_id}/original.jpg"
        self.storage.put(original_path, image_bytes)
        
        # Generate all versions
        urls = {"original": original_path}
        for version in self.VERSIONS:
            resized_bytes = self._resize_image(img, version)
            path = f"photos/{post_id}/{version.name}.jpg"
            self.storage.put(path, resized_bytes)
            urls[version.name] = path
        
        # Trigger downstream tasks (async, non-blocking)
        self._extract_metadata(image_bytes, post_id)
        self._run_content_moderation(image_bytes, post_id)
        self._detect_faces(image_bytes, post_id)
        self._generate_alt_text(image_bytes, post_id)
        
        return urls
    
    def _resize_image(self, img: Image.Image, version: ImageVersion) -> bytes:
        """Resize maintaining aspect ratio, then center-crop to square."""
        # Resize so smallest dimension matches target
        ratio = max(version.max_width / img.width, version.max_height / img.height)
        new_size = (int(img.width * ratio), int(img.height * ratio))
        resized = img.resize(new_size, Image.LANCZOS)
        
        # Center crop to exact target size
        left = (resized.width - version.max_width) // 2
        top = (resized.height - version.max_height) // 2
        cropped = resized.crop((left, top, left + version.max_width, 
                                top + version.max_height))
        
        # Encode as JPEG
        buffer = io.BytesIO()
        cropped.save(buffer, format="JPEG", quality=version.quality, optimize=True)
        return buffer.getvalue()
    
    def _extract_metadata(self, image_bytes, post_id):
        """Extract EXIF: camera, GPS, timestamp (strip GPS for privacy)."""
        pass  # Celery task
    
    def _run_content_moderation(self, image_bytes, post_id):
        """AI content moderation: nudity, violence, spam detection."""
        pass  # Celery task → ML model inference
    
    def _detect_faces(self, image_bytes, post_id):
        """Face detection for photo tagging suggestions."""
        pass  # Celery task → face recognition model
    
    def _generate_alt_text(self, image_bytes, post_id):
        """Auto-generate accessibility alt text using vision AI."""
        pass  # Celery task → image captioning model
```

### Java — Feed Ranking with Pre-computed Scores

```java
import java.util.*;
import java.util.stream.Collectors;

/**
 * Simplified Instagram feed ranking.
 * Combines pre-computed ML scores with business rules.
 */
public class FeedRanker {
    
    record FeedCandidate(
        String postId,
        String authorId,
        long timestampMs,
        double mlScore,      // Pre-computed engagement prediction
        String contentType,  // "photo", "video", "carousel", "reel"
        boolean isAd
    ) {}

    /**
     * Rank candidates for a specific user's feed.
     * Instagram uses a deep neural network; this shows the concept.
     */
    public List<FeedCandidate> rankFeed(String userId, List<FeedCandidate> candidates) {
        
        // Step 1: Score each candidate
        List<ScoredPost> scored = candidates.stream()
            .map(c -> new ScoredPost(c, computeFinalScore(userId, c)))
            .collect(Collectors.toList());
        
        // Step 2: Sort by score (highest first)
        scored.sort((a, b) -> Double.compare(b.score, a.score));
        
        // Step 3: Apply diversity rules
        List<FeedCandidate> diversified = applyDiversity(scored);
        
        // Step 4: Insert ads at natural positions
        return insertAds(diversified, candidates.stream()
            .filter(FeedCandidate::isAd).collect(Collectors.toList()));
    }

    private double computeFinalScore(String userId, FeedCandidate candidate) {
        double score = candidate.mlScore();  // Base ML prediction
        
        // Recency boost (newer posts score higher)
        long ageHours = (System.currentTimeMillis() - candidate.timestampMs()) / 3_600_000;
        double recencyMultiplier = Math.max(0.5, 1.0 - (ageHours * 0.02));
        score *= recencyMultiplier;
        
        // Content type boost (Reels get priority in 2024)
        if (candidate.contentType().equals("reel")) score *= 1.3;
        if (candidate.contentType().equals("video")) score *= 1.1;
        
        return score;
    }

    private List<FeedCandidate> applyDiversity(List<ScoredPost> scored) {
        // Rule: No more than 2 posts from same author in top 10
        List<FeedCandidate> result = new ArrayList<>();
        Map<String, Integer> authorCount = new HashMap<>();
        
        for (ScoredPost sp : scored) {
            String author = sp.candidate.authorId();
            int count = authorCount.getOrDefault(author, 0);
            if (count < 2) {
                result.add(sp.candidate);
                authorCount.put(author, count + 1);
            }
            if (result.size() >= 30) break;
        }
        return result;
    }

    private List<FeedCandidate> insertAds(List<FeedCandidate> feed, List<FeedCandidate> ads) {
        // Insert ad every 5-7 posts
        List<FeedCandidate> withAds = new ArrayList<>();
        int adIdx = 0;
        for (int i = 0; i < feed.size(); i++) {
            withAds.add(feed.get(i));
            if ((i + 1) % 6 == 0 && adIdx < ads.size()) {
                withAds.add(ads.get(adIdx++));
            }
        }
        return withAds;
    }

    record ScoredPost(FeedCandidate candidate, double score) {}
}
```

---

## Infrastructure Examples

### Instagram's Complete Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Language** | Python (Django), C++ (performance-critical) | Web app, media processing |
| **Web Framework** | Django (heavily customized) | API and business logic |
| **Task Queue** | Celery + RabbitMQ | Async image processing |
| **Primary DB** | PostgreSQL (sharded, 1000s of shards) | Users, posts, relationships |
| **Feed Storage** | Cassandra | High-write feed data |
| **Cache** | Redis + Memcached (TAO/Meta cache) | Feed, counters, sessions |
| **Object Storage** | Meta's internal (like S3) | Photos, videos |
| **CDN** | Meta's global CDN | Image/video delivery |
| **Search** | Elasticsearch | User search, hashtags |
| **ML** | PyTorch (internal models) | Feed ranking, content moderation |
| **Messaging** | Kafka | Event streaming between services |

### Scale Numbers

```
┌─────────────────────────────────────────────────────────────────┐
│              INSTAGRAM BY THE NUMBERS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  • 2+ billion monthly active users                              │
│  • 100+ million photos/videos uploaded daily                    │
│  • 500+ million daily active Stories users                      │
│  • 2+ billion Reels plays per day                              │
│  • 4.2 billion likes per day                                    │
│  • 95 million photos/videos shared per day                      │
│                                                                   │
│  Infrastructure:                                                  │
│  • Thousands of PostgreSQL shards                               │
│  • Petabytes of photo/video storage                             │
│  • Largest Django deployment in the world                       │
│  • Hundreds of millions of Redis operations/sec                 │
│  • Processing: 100M+ images through ML models daily             │
│                                                                   │
│  Performance:                                                     │
│  • Photo upload: < 2 seconds (user perception)                  │
│  • Feed load: < 500ms (P95)                                    │
│  • Image serving from CDN: < 50ms (cache hit)                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### Instagram's Migration Story — From Startup to Meta Scale

```
┌─────────────────────────────────────────────────────────────────┐
│         INSTAGRAM'S EVOLUTION                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  2010 (Launch): 2 engineers, 1 Django server                    │
│  ─────────────────────────────────────────                       │
│  • Single EC2 instance                                           │
│  • PostgreSQL (single instance)                                  │
│  • Photos stored on Amazon S3                                    │
│  • 25,000 users on day 1                                        │
│                                                                   │
│  2011: 10M users, still tiny team                               │
│  ─────────────────────────────────                               │
│  • Added Redis for caching                                      │
│  • PostgreSQL replicas for reads                                 │
│  • Still on AWS                                                  │
│  • ~5 engineers                                                  │
│                                                                   │
│  2012: Acquired by Facebook (1B users by 2018)                  │
│  ───────────────────────────────────────────                     │
│  • Migrated to Facebook infrastructure                          │
│  • PostgreSQL sharding implemented                              │
│  • TAO (Facebook's cache layer) adopted                         │
│  • Moved to Facebook's custom CDN                               │
│                                                                   │
│  2020+: 2B users, full Meta integration                         │
│  ────────────────────────────────────────                        │
│  • Thousands of PostgreSQL shards                               │
│  • ML everywhere (feed, explore, ads, moderation)              │
│  • Reels (competing with TikTok)                               │
│  • Still Django! (with heavy optimization)                     │
│                                                                   │
│  Key insight: They KEPT Python/Django even at 2B users.         │
│  They optimized it (Cinder — Meta's custom Python runtime)      │
│  rather than rewriting in Go/Java/C++.                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### How Instagram Handles a Celebrity Post

When a celebrity (50M followers) posts a photo:

1. **Upload & process**: Same as anyone else (~1 second)
2. **Fan-out**: NOT pushed to all 50M follower feeds immediately
3. **Pull-based for celebrities**: Similar to Twitter's hybrid approach (see Chapter 24.6)
4. **CDN pre-warming**: Photo pushed to CDN edges in anticipation of high traffic
5. **Like counter**: Redis counter, eventually consistent (OK to show 1.2M instead of 1,200,037)
6. **Engagement spike**: Auto-scales serving capacity for this specific post

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Better Approach |
|---------|---------------|-----------------|
| Serving original resolution to all devices | 8 MB photo to a 150px thumbnail slot wastes bandwidth | Generate multiple sizes; serve appropriate version per device |
| Synchronous image processing | User waits 10+ seconds while server resizes, moderates, etc. | Store original immediately, process async (Celery/workers) |
| Single database for everything at scale | PostgreSQL can't handle 2B users in one instance | Shard by user_id; keep user's data co-located |
| No CDN for images | Every image request hits your origin server | Use CDN; 90%+ of image requests should be cache hits |
| Running ML models in the request path | Adds 500ms+ to upload/feed time | Pre-compute rankings offline; serve from cache |
| Not stripping EXIF GPS data | Exposing users' exact location = serious privacy violation | Strip GPS coordinates during upload processing |

---

## When to Use / When NOT to Use

### When to Use This Architecture
- User-generated image/video content at scale
- Feed/timeline-based product (social network, content platform)
- Need for multiple image sizes across diverse devices
- Content discovery (Explore page) is important
- Billions of daily image serves

### When NOT to Use This Architecture
- Small image hosting (< 1M images) → Use Cloudinary, Imgix, or S3 + CloudFront directly
- No feed/social component → Simple CDN + storage is enough
- Enterprise document management → Different access patterns
- Real-time image processing (like Photoshop) → Different architecture entirely
- Video-first platform → See YouTube/TikTok architecture (Chapter 24.5)

---

## Key Takeaways

1. **Multiple image sizes** are non-negotiable at scale: Generate 5-6 versions of every upload and serve the right one for each device/context. Serving originals everywhere wastes 90% of bandwidth.
2. **Async processing** is the key to fast uploads: Store the original immediately (< 1 second), then resize, moderate, and index in background workers (Celery).
3. **Django scales to 2 billion users** — Instagram proves you don't NEED to rewrite in Go/Java. Optimize your runtime (Meta's Cinder Python), shard your database, and cache aggressively.
4. **PostgreSQL sharding by user_id** keeps all of a user's data co-located: their posts, comments, and relationships on the same shard = no cross-shard queries for common operations.
5. **Feed ranking with ML** replaced chronological ordering: predict P(engagement) for each candidate post and rank accordingly. This is now the standard for all social platforms.
6. **CDN cache hit rate > 90%** means most image requests never reach your origin. The power law applies: a small % of photos get the vast majority of views.
7. **Content moderation at upload time** using ML ensures policy-violating content is caught before it reaches any feed — this runs on every single upload (100M+ per day).

---

## What's Next?

Congratulations! You've completed **Part 24: Real-World System Designs**. You now have a deep understanding of how the world's most demanding applications are architected at planet scale.

Continue to [Part 25: Design Patterns & Best Practices for Large Systems](../25-design-patterns/01-twelve-factor-app.md) to learn the patterns and methodologies that enable these systems to evolve and maintain quality over time.
