# Content Delivery & Media Processing Pipelines

> **What you'll learn**: How companies like Netflix, YouTube, and Instagram ingest, process, store, and deliver billions of images and videos to users worldwide — covering transcoding pipelines, adaptive bitrate streaming, thumbnail generation, CDN integration, and media optimization at planet scale.

---

## Real-Life Analogy

Imagine you're running a **global TV network** with billions of viewers:

1. A filmmaker submits a **master recording** (original high-quality video)
2. Your studio **converts it into dozens of formats** — 4K for big TVs, 1080p for laptops, 480p for slow phone connections, different languages for audio tracks
3. You **cut each version into small clips** (2-second chunks) so viewers can start watching instantly
4. You **ship copies to local stations** in every city worldwide (CDN edge nodes)
5. When someone presses play, their **local station** serves the video, not the central studio
6. If their internet gets slow, you **instantly switch** them to a lower quality version (adaptive streaming)

That's a media processing pipeline. It takes ONE upload and turns it into HUNDREDS of optimized versions, then distributes them globally so any user, on any device, on any network speed, gets the best possible experience.

---

## Core Concept Explained Step-by-Step

### The Complete Media Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│              MEDIA PROCESSING PIPELINE (End-to-End)                  │
│                                                                     │
│  ┌─────────┐    ┌──────────┐    ┌──────────────┐    ┌──────────┐ │
│  │ INGEST  │───▶│ PROCESS  │───▶│    STORE     │───▶│ DELIVER  │ │
│  │         │    │          │    │              │    │          │ │
│  │• Upload │    │• Transcode│    │• Object Store│    │• CDN     │ │
│  │• Validate│    │• Resize  │    │• Metadata DB│    │• Adaptive│ │
│  │• Queue  │    │• Thumbnail│    │• Manifest   │    │• Edge    │ │
│  │         │    │• Watermark│    │              │    │          │ │
│  └─────────┘    └──────────┘    └──────────────┘    └──────────┘ │
│                                                                     │
│       ▲              ▲                ▲                  ▲         │
│       │              │                │                  │         │
│    Upload API    Worker Farm     S3/GCS/Blob        CloudFront    │
│    (chunked)     (GPU/CPU)       + Database         /Akamai       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### What Happens When You Upload a Video to YouTube

```
YOU                     YouTube                Processing Farm         CDN
 │                        │                        │                    │
 │  Upload 4K video      │                        │                    │
 │  (2 GB file)         │                        │                    │
 │───────────────────────▶│                        │                    │
 │                        │                        │                    │
 │                        │  1. Chunk upload       │                    │
 │                        │     (resumable)        │                    │
 │                        │                        │                    │
 │                        │  2. Store original     │                    │
 │                        │     in object storage  │                    │
 │                        │                        │                    │
 │                        │  3. Queue transcode    │                    │
 │                        │     job ──────────────▶│                    │
 │                        │                        │                    │
 │                        │                        │  4. Encode to:     │
 │                        │                        │     • 2160p (4K)   │
 │                        │                        │     • 1440p        │
 │                        │                        │     • 1080p        │
 │                        │                        │     • 720p         │
 │                        │                        │     • 480p         │
 │                        │                        │     • 360p         │
 │                        │                        │     • 144p         │
 │                        │                        │                    │
 │                        │                        │  5. Generate:      │
 │                        │                        │     • Thumbnails   │
 │                        │                        │     • Preview      │
 │                        │                        │     • Subtitles    │
 │                        │                        │     • Chapters     │
 │                        │                        │                    │
 │                        │  6. Store all versions │                    │
 │                        │  ◀──────────────────────│                    │
 │                        │                        │                    │
 │                        │  7. Push to CDN ───────────────────────────▶│
 │                        │                        │                    │
 │  "Video ready!"       │                        │                    │
 │◀───────────────────────│                        │                    │
 │                        │                        │                    │
 │                        │                        │                    │
 │  VIEWER REQUESTS VIDEO:                         │                    │
 │───────────────────────▶│                        │                    │
 │                        │  "Redirect to nearest CDN"                  │
 │◀───────────────────────│                        │                    │
 │                        │                        │                    │
 │  Stream from CDN edge  │                        │                    │
 │◀────────────────────────────────────────────────────────────────────│
```

---

## Phase 1: Ingestion — Uploading Media

### Chunked / Resumable Uploads

Large files need special handling. You can't upload 2 GB in one HTTP request — it'll timeout or fail.

```
┌─────────────────────────────────────────────────────┐
│            RESUMABLE UPLOAD PROTOCOL                 │
│                                                     │
│  Client                          Server             │
│    │                               │                │
│    │  POST /upload/init            │                │
│    │  {filename, size, type}       │                │
│    │──────────────────────────────▶│                │
│    │                               │                │
│    │  200 OK {upload_id: "abc123"} │                │
│    │◀──────────────────────────────│                │
│    │                               │                │
│    │  PUT /upload/abc123           │                │
│    │  Content-Range: bytes 0-5MB   │                │
│    │  [chunk 1 data]              │                │
│    │──────────────────────────────▶│ ← Store chunk  │
│    │                               │                │
│    │  PUT /upload/abc123           │                │
│    │  Content-Range: bytes 5-10MB  │                │
│    │  [chunk 2 data]              │                │
│    │──────────────────────────────▶│ ← Store chunk  │
│    │                               │                │
│    │  ❌ Network failure!          │                │
│    │                               │                │
│    │  (Minutes later, resume...)   │                │
│    │                               │                │
│    │  GET /upload/abc123/status    │                │
│    │──────────────────────────────▶│                │
│    │                               │                │
│    │  200 {bytes_received: 10MB}   │                │
│    │◀──────────────────────────────│                │
│    │                               │                │
│    │  PUT /upload/abc123           │                │
│    │  Content-Range: bytes 10-15MB │                │
│    │  [chunk 3 data]              │ ← Resume!      │
│    │──────────────────────────────▶│                │
│    │                               │                │
│    │  ... continue until done ...  │                │
│    │                               │                │
│    │  POST /upload/abc123/complete │                │
│    │──────────────────────────────▶│                │
│    │                               │                │
│    │  200 {status: "processing"}   │                │
│    │◀──────────────────────────────│                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Validation at Ingestion

```
┌─────────────────────────────────────────────────────┐
│            UPLOAD VALIDATION PIPELINE               │
│                                                     │
│  1. FORMAT CHECK                                    │
│     ├── Is it a valid video container? (MP4, MOV)  │
│     ├── Is the codec supported? (H.264, VP9)       │
│     └── Is the file corrupt? (checksum/probe)      │
│                                                     │
│  2. SAFETY CHECK                                    │
│     ├── Malware scan (ClamAV / cloud scanning)     │
│     ├── Content moderation (AI - nudity/violence)  │
│     └── Copyright detection (fingerprinting)       │
│                                                     │
│  3. METADATA EXTRACTION                             │
│     ├── Duration, resolution, FPS, bitrate         │
│     ├── Audio tracks, subtitle tracks              │
│     ├── EXIF data (for photos)                     │
│     └── GPS coordinates (if present)               │
│                                                     │
│  4. QUOTA CHECK                                     │
│     ├── User's storage quota                       │
│     ├── File size limits                           │
│     └── Rate limiting (uploads per hour)           │
│                                                     │
│  IF ALL PASS → Queue for processing                │
│  IF FAIL → Return error to user with reason        │
└─────────────────────────────────────────────────────┘
```

---

## Phase 2: Processing — Transcoding & Transformation

### Video Transcoding

**Transcoding** = Converting video from one format/resolution/bitrate to another.

```
┌─────────────────────────────────────────────────────────────────┐
│                   TRANSCODING PIPELINE                           │
│                                                                 │
│  INPUT: 4K Master (H.264, 50 Mbps, 3840×2160, 60fps)          │
│                                                                 │
│            ┌─────────────────────────────────────┐              │
│            │      TRANSCODING FARM               │              │
│            │                                     │              │
│            │  ┌───────┐ ┌───────┐ ┌───────┐    │              │
│            │  │Worker1│ │Worker2│ │Worker3│    │              │
│            │  │(GPU)  │ │(GPU)  │ │(GPU)  │    │              │
│            │  └───┬───┘ └───┬───┘ └───┬───┘    │              │
│            └──────┼─────────┼─────────┼─────────┘              │
│                   │         │         │                         │
│                   ▼         ▼         ▼                         │
│                                                                 │
│  OUTPUTS:                                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Resolution │ Codec  │ Bitrate │ Use Case                 │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ 2160p (4K) │ H.265  │ 15 Mbps │ Smart TVs, fiber        │  │
│  │ 1440p      │ H.265  │ 10 Mbps │ Desktop monitors        │  │
│  │ 1080p      │ H.264  │ 5 Mbps  │ Laptops, good mobile    │  │
│  │ 720p       │ H.264  │ 2.5 Mbps│ Tablets, avg mobile     │  │
│  │ 480p       │ H.264  │ 1 Mbps  │ Slow connections        │  │
│  │ 360p       │ H.264  │ 0.5 Mbps│ Very slow connections   │  │
│  │ 144p       │ H.264  │ 0.1 Mbps│ Extremely slow          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  EACH resolution is then SEGMENTED into 2-4 second chunks:     │
│                                                                 │
│  1080p/                                                         │
│    ├── segment_001.ts  (2 seconds)                             │
│    ├── segment_002.ts  (2 seconds)                             │
│    ├── segment_003.ts  (2 seconds)                             │
│    └── ...                                                      │
│                                                                 │
│  Total outputs for a 2-hour movie:                             │
│  7 resolutions × ~3,600 segments = ~25,200 files!             │
│  + audio tracks × languages                                    │
│  + subtitle tracks                                             │
│  = Potentially 50,000+ files per video                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Adaptive Bitrate Streaming (ABR)

This is how Netflix/YouTube seamlessly adjusts quality based on your network speed:

```
┌─────────────────────────────────────────────────────────────────┐
│           ADAPTIVE BITRATE STREAMING (HLS / DASH)               │
│                                                                 │
│  MANIFEST FILE (master.m3u8 or manifest.mpd):                  │
│  ┌─────────────────────────────────────────────────────┐       │
│  │ #EXTM3U                                            │       │
│  │ #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360      │
│  │ 360p/playlist.m3u8                                 │       │
│  │ #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720   │
│  │ 720p/playlist.m3u8                                 │       │
│  │ #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080  │
│  │ 1080p/playlist.m3u8                                │       │
│  │ #EXT-X-STREAM-INF:BANDWIDTH=15000000,RESOLUTION=3840x2160 │
│  │ 4k/playlist.m3u8                                   │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
│  PLAYER BEHAVIOR:                                               │
│                                                                 │
│  Time    Network Speed    Quality Selected                      │
│  ─────   ─────────────    ────────────────                      │
│  0:00    10 Mbps          1080p ████████████████                │
│  0:30    ↓ drops to 3     720p  ████████████                    │
│  1:00    ↓ drops to 1     480p  ████████                        │
│  1:30    ↑ recovers 8     1080p ████████████████                │
│  2:00    ↑ rises to 20    4K    ████████████████████████        │
│                                                                 │
│  HOW IT WORKS:                                                  │
│  1. Player downloads manifest (list of all quality levels)     │
│  2. Player starts with low quality (fast start)                │
│  3. Measures download speed of each segment                    │
│  4. If speed > next quality's bitrate → upgrade               │
│  5. If speed < current quality's bitrate → downgrade          │
│  6. Because segments are only 2-4s, switches are seamless     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Image Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│           IMAGE PROCESSING PIPELINE (Instagram-style)            │
│                                                                 │
│  ORIGINAL UPLOAD (12 MP, 4032×3024, 8 MB JPEG)                 │
│                    │                                            │
│                    ▼                                            │
│  ┌─────────────────────────────────────────┐                   │
│  │         IMAGE PROCESSING                 │                   │
│  │                                         │                   │
│  │  1. Strip EXIF (remove GPS, camera info)│                   │
│  │  2. Auto-orient (fix rotation)          │                   │
│  │  3. Color space → sRGB                  │                   │
│  │  4. Generate variants:                  │                   │
│  │                                         │                   │
│  │     ┌──────────┬────────┬─────────┐    │                   │
│  │     │  SIZE    │ PIXELS │  USE    │    │                   │
│  │     ├──────────┼────────┼─────────┤    │                   │
│  │     │ original │ 4032px │ zoom    │    │                   │
│  │     │ large    │ 1080px │ feed    │    │                   │
│  │     │ medium   │ 640px  │ grid    │    │                   │
│  │     │ thumb    │ 150px  │ avatar  │    │                   │
│  │     │ micro    │ 50px   │ notif.  │    │                   │
│  │     └──────────┴────────┴─────────┘    │                   │
│  │                                         │                   │
│  │  5. Convert formats:                    │                   │
│  │     • JPEG (fallback, universal)       │                   │
│  │     • WebP (30% smaller, most browsers)│                   │
│  │     • AVIF (50% smaller, modern only)  │                   │
│  │                                         │                   │
│  │  6. Quality optimization:               │                   │
│  │     • Perceptual quality (SSIM-based)  │                   │
│  │     • Progressive JPEG (loads blurry → sharp)              │
│  │     • Lazy loading placeholder (tiny blur)│                │
│  │                                         │                   │
│  └─────────────────────────────────────────┘                   │
│                    │                                            │
│                    ▼                                            │
│  STORED: 5 sizes × 3 formats = 15 files per photo             │
│  ORIGINAL: 8 MB → OPTIMIZED: ~200 KB average served           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Queue-Based Processing Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│          DISTRIBUTED MEDIA PROCESSING ARCHITECTURE                   │
│                                                                     │
│  ┌──────────┐     ┌─────────────┐     ┌──────────────────────┐    │
│  │  Upload  │────▶│  Message    │────▶│   Worker Pool        │    │
│  │  API     │     │  Queue      │     │                      │    │
│  │          │     │  (SQS/      │     │  ┌────────────────┐  │    │
│  │  • Auth  │     │   Kafka)    │     │  │ Worker 1 (GPU) │  │    │
│  │  • Chunk │     │             │     │  │ Transcoding    │  │    │
│  │  • Store │     │  Jobs:      │     │  └────────────────┘  │    │
│  │    raw   │     │  ┌───────┐  │     │  ┌────────────────┐  │    │
│  └──────────┘     │  │transc.│  │     │  │ Worker 2 (GPU) │  │    │
│                   │  ├───────┤  │     │  │ Transcoding    │  │    │
│                   │  │thumb  │  │     │  └────────────────┘  │    │
│                   │  ├───────┤  │     │  ┌────────────────┐  │    │
│                   │  │waterma│  │     │  │ Worker 3 (CPU) │  │    │
│                   │  └───────┘  │     │  │ Thumbnails     │  │    │
│                   └─────────────┘     │  └────────────────┘  │    │
│                                       │  ┌────────────────┐  │    │
│                                       │  │ Worker 4 (CPU) │  │    │
│                                       │  │ Metadata/AI    │  │    │
│                                       │  └────────────────┘  │    │
│                                       └──────────┬───────────┘    │
│                                                  │                 │
│                                                  ▼                 │
│                                       ┌──────────────────┐        │
│                                       │  Object Storage  │        │
│                                       │  (S3/GCS)        │        │
│                                       │                  │        │
│                                       │  /videos/{id}/   │        │
│                                       │    original.mp4  │        │
│                                       │    1080p/        │        │
│                                       │    720p/         │        │
│                                       │    manifest.m3u8 │        │
│                                       │                  │        │
│                                       │  /images/{id}/   │        │
│                                       │    original.jpg  │        │
│                                       │    large.webp    │        │
│                                       │    thumb.webp    │        │
│                                       └────────┬─────────┘        │
│                                                │                   │
│                                                ▼                   │
│                                       ┌──────────────────┐        │
│                                       │      CDN         │        │
│                                       │  (CloudFront/    │        │
│                                       │   Akamai/        │        │
│                                       │   Fastly)        │        │
│                                       └──────────────────┘        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### CDN Delivery with Origin Shield

```
┌─────────────────────────────────────────────────────────────────┐
│                CDN DELIVERY ARCHITECTURE                         │
│                                                                 │
│  User in Tokyo        User in NYC         User in London       │
│       │                    │                    │               │
│       ▼                    ▼                    ▼               │
│  ┌─────────┐         ┌─────────┐         ┌─────────┐         │
│  │Edge Node│         │Edge Node│         │Edge Node│         │
│  │ Tokyo   │         │  NYC    │         │ London  │         │
│  │         │         │         │         │         │         │
│  │ Cache:  │         │ Cache:  │         │ Cache:  │         │
│  │ HIT? ──▶ serve   │ HIT? ──▶ serve   │ HIT? ──▶ serve   │
│  │ MISS?   │         │ MISS?   │         │ MISS?   │         │
│  └────┬────┘         └────┬────┘         └────┬────┘         │
│       │                    │                    │               │
│       └────────────────────┼────────────────────┘               │
│                            │                                    │
│                            ▼                                    │
│                   ┌─────────────────┐                          │
│                   │  ORIGIN SHIELD  │  (Regional cache layer)  │
│                   │  (reduces hits  │                          │
│                   │   to origin)    │                          │
│                   │                 │                          │
│                   │  Only 1 request │                          │
│                   │  to origin even │                          │
│                   │  if 100 edges   │                          │
│                   │  miss at once   │                          │
│                   └────────┬────────┘                          │
│                            │                                    │
│                            ▼                                    │
│                   ┌─────────────────┐                          │
│                   │    ORIGIN       │                          │
│                   │  (S3 Bucket)    │                          │
│                   │                 │                          │
│                   │  All processed  │                          │
│                   │  media files    │                          │
│                   └─────────────────┘                          │
│                                                                 │
│  RESULT: 95%+ of requests served from edge (< 20ms latency)   │
│          Origin only hit for first request or cache expiry     │
└─────────────────────────────────────────────────────────────────┘
```

### On-the-Fly Image Transformation

Instead of pre-generating all sizes, some systems transform images **on request**:

```
┌──────────────────────────────────────────────────────────┐
│        ON-THE-FLY IMAGE TRANSFORMATION                   │
│                                                          │
│  Request URL encodes transformations:                    │
│                                                          │
│  https://images.myapp.com/photo-123/w=400,h=300,f=webp │
│                                      ▲     ▲       ▲    │
│                                      │     │       │    │
│                                   width  height  format  │
│                                                          │
│  FLOW:                                                   │
│                                                          │
│  Client ──▶ CDN Edge                                    │
│              │                                           │
│              ├── Cache HIT? → Serve immediately          │
│              │                                           │
│              └── Cache MISS? → Image Processing Service  │
│                                      │                   │
│                                      ├── Fetch original  │
│                                      │   from S3        │
│                                      │                   │
│                                      ├── Apply transforms│
│                                      │   (resize, crop, │
│                                      │    format convert)│
│                                      │                   │
│                                      ├── Cache result   │
│                                      │   at CDN edge    │
│                                      │                   │
│                                      └── Return to user │
│                                                          │
│  TOOLS: Imgproxy, Thumbor, Cloudinary, imgix             │
│  ADVANTAGE: Only generate sizes that are actually needed │
│  RISK: First request is slow (cold cache)               │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Video Transcoding Pipeline with FFmpeg

```python
import subprocess
import json
import boto3
from pathlib import Path

def probe_video(input_path: str) -> dict:
    """Extract metadata from video file using ffprobe."""
    cmd = [
        'ffprobe', '-v', 'quiet',
        '-print_format', 'json',
        '-show_streams', '-show_format',
        input_path
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    return json.loads(result.stdout)

def transcode_video(input_path: str, output_path: str, 
                    resolution: str, bitrate: str):
    """Transcode video to specific resolution and bitrate."""
    cmd = [
        'ffmpeg', '-i', input_path,
        '-vf', f'scale={resolution}',       # Resize
        '-b:v', bitrate,                     # Video bitrate
        '-c:v', 'libx264',                   # H.264 codec
        '-preset', 'medium',                 # Encoding speed/quality trade-off
        '-c:a', 'aac',                       # Audio codec
        '-b:a', '128k',                      # Audio bitrate
        '-movflags', '+faststart',           # Enable streaming
        '-y', output_path                    # Overwrite output
    ]
    subprocess.run(cmd, check=True)

def create_hls_segments(input_path: str, output_dir: str):
    """Split video into HLS segments for adaptive streaming."""
    Path(output_dir).mkdir(parents=True, exist_ok=True)
    cmd = [
        'ffmpeg', '-i', input_path,
        '-hls_time', '4',                    # 4-second segments
        '-hls_list_size', '0',               # Include all segments in playlist
        '-hls_segment_filename', f'{output_dir}/segment_%04d.ts',
        f'{output_dir}/playlist.m3u8'
    ]
    subprocess.run(cmd, check=True)

def generate_thumbnail(input_path: str, output_path: str, time: str = '00:00:05'):
    """Extract a thumbnail frame from video."""
    cmd = [
        'ffmpeg', '-i', input_path,
        '-ss', time,                         # Seek to timestamp
        '-vframes', '1',                     # Extract 1 frame
        '-vf', 'scale=640:-1',              # Width 640, auto height
        '-q:v', '2',                         # JPEG quality
        '-y', output_path
    ]
    subprocess.run(cmd, check=True)

# --- COMPLETE PIPELINE ---
def process_upload(video_id: str, input_path: str):
    """Full processing pipeline for an uploaded video."""
    s3 = boto3.client('s3')
    bucket = 'my-app-media'
    
    # Define output variants
    variants = [
        {'res': '1920:1080', 'bitrate': '5000k', 'label': '1080p'},
        {'res': '1280:720',  'bitrate': '2500k', 'label': '720p'},
        {'res': '854:480',   'bitrate': '1000k', 'label': '480p'},
        {'res': '640:360',   'bitrate': '500k',  'label': '360p'},
    ]
    
    for variant in variants:
        # 1. Transcode
        output = f'/tmp/{video_id}_{variant["label"]}.mp4'
        transcode_video(input_path, output, variant['res'], variant['bitrate'])
        
        # 2. Create HLS segments
        hls_dir = f'/tmp/{video_id}/{variant["label"]}'
        create_hls_segments(output, hls_dir)
        
        # 3. Upload segments to S3
        for file in Path(hls_dir).iterdir():
            s3.upload_file(
                str(file), bucket,
                f'videos/{video_id}/{variant["label"]}/{file.name}',
                ExtraArgs={'ContentType': 'video/MP2T'}
            )
    
    # 4. Generate thumbnail
    thumb_path = f'/tmp/{video_id}_thumb.jpg'
    generate_thumbnail(input_path, thumb_path)
    s3.upload_file(thumb_path, bucket, f'videos/{video_id}/thumbnail.jpg')
    
    # 5. Create master manifest
    master_manifest = generate_master_manifest(video_id, variants)
    s3.put_object(
        Bucket=bucket,
        Key=f'videos/{video_id}/master.m3u8',
        Body=master_manifest,
        ContentType='application/vnd.apple.mpegurl'
    )

def generate_master_manifest(video_id: str, variants: list) -> str:
    """Generate HLS master playlist pointing to all quality levels."""
    lines = ['#EXTM3U']
    bandwidth_map = {'1080p': 5000000, '720p': 2500000, 
                     '480p': 1000000, '360p': 500000}
    
    for v in variants:
        lines.append(
            f'#EXT-X-STREAM-INF:BANDWIDTH={bandwidth_map[v["label"]]},'
            f'RESOLUTION={v["res"].replace(":", "x")}'
        )
        lines.append(f'{v["label"]}/playlist.m3u8')
    
    return '\n'.join(lines)
```

### Java — Image Processing Service with Thumbnailator

```java
import net.coobird.thumbnailator.Thumbnails;
import net.coobird.thumbnailator.geometry.Positions;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.*;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.*;
import java.nio.file.*;
import java.util.List;
import java.util.Map;

public class ImageProcessingWorker {

    private final S3Client s3 = S3Client.create();
    private final SqsClient sqs = SqsClient.create();
    private final String queueUrl = "https://sqs.us-east-1.amazonaws.com/123/image-jobs";
    private final String bucket = "my-app-media";

    // Image size variants to generate
    private static final List<ImageVariant> VARIANTS = List.of(
        new ImageVariant("large", 1080, 1080, 0.85),
        new ImageVariant("medium", 640, 640, 0.80),
        new ImageVariant("thumb", 150, 150, 0.75),
        new ImageVariant("micro", 50, 50, 0.70)
    );

    public void pollAndProcess() {
        while (true) {
            // 1. Poll SQS for new image processing jobs
            var messages = sqs.receiveMessage(ReceiveMessageRequest.builder()
                .queueUrl(queueUrl)
                .maxNumberOfMessages(10)
                .waitTimeSeconds(20)  // Long polling
                .build()).messages();

            for (Message msg : messages) {
                processImage(msg);
                // Delete message after successful processing
                sqs.deleteMessage(DeleteMessageRequest.builder()
                    .queueUrl(queueUrl)
                    .receiptHandle(msg.receiptHandle())
                    .build());
            }
        }
    }

    private void processImage(Message msg) {
        // Parse job: {imageId: "abc", key: "uploads/raw/photo.jpg"}
        String imageId = extractField(msg.body(), "imageId");
        String sourceKey = extractField(msg.body(), "key");

        // 2. Download original from S3
        Path tempOriginal = Path.of("/tmp", imageId + "_original.jpg");
        s3.getObject(
            GetObjectRequest.builder().bucket(bucket).key(sourceKey).build(),
            tempOriginal
        );

        // 3. Generate each variant
        for (ImageVariant variant : VARIANTS) {
            Path output = Path.of("/tmp", imageId + "_" + variant.name + ".webp");
            
            try {
                Thumbnails.of(tempOriginal.toFile())
                    .size(variant.width, variant.height)
                    .crop(Positions.CENTER)        // Center crop to exact size
                    .outputFormat("webp")          // Modern format (30% smaller)
                    .outputQuality(variant.quality)
                    .toFile(output.toFile());

                // 4. Upload variant to S3
                s3.putObject(
                    PutObjectRequest.builder()
                        .bucket(bucket)
                        .key(String.format("images/%s/%s.webp", imageId, variant.name))
                        .contentType("image/webp")
                        .cacheControl("public, max-age=31536000") // Cache 1 year
                        .build(),
                    output
                );
            } catch (IOException e) {
                System.err.println("Failed to process variant: " + variant.name);
            }
        }

        // 5. Generate blur placeholder (tiny 20px image, base64 encoded)
        generateBlurPlaceholder(tempOriginal, imageId);
    }

    private void generateBlurPlaceholder(Path original, String imageId) {
        try {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            Thumbnails.of(original.toFile())
                .size(20, 20)
                .outputFormat("jpeg")
                .outputQuality(0.3)
                .toOutputStream(baos);
            
            // Store as base64 in metadata DB for instant loading
            String base64 = java.util.Base64.getEncoder().encodeToString(baos.toByteArray());
            // saveToDatabase(imageId, "blur_placeholder", base64);
        } catch (IOException e) {
            // Non-critical, skip placeholder
        }
    }

    record ImageVariant(String name, int width, int height, double quality) {}
}
```

---

## Infrastructure Examples

### AWS Media Processing Architecture (Terraform)

```hcl
# S3 Bucket for media storage
resource "aws_s3_bucket" "media" {
  bucket = "myapp-media-prod"
}

# CloudFront CDN distribution
resource "aws_cloudfront_distribution" "media_cdn" {
  origin {
    domain_name = aws_s3_bucket.media.bucket_regional_domain_name
    origin_id   = "S3-media"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.media.cloudfront_access_identity_path
    }
  }

  enabled = true
  
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-media"

    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 86400      # 1 day
    max_ttl                = 31536000   # 1 year
    compress               = true       # Gzip/Brotli
  }

  # Serve from nearest edge location
  price_class = "PriceClass_All"

  restrictions {
    geo_restriction { restriction_type = "none" }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

# SQS Queue for processing jobs
resource "aws_sqs_queue" "transcode_jobs" {
  name                       = "video-transcode-jobs"
  visibility_timeout_seconds = 900  # 15 min (long processing)
  message_retention_seconds  = 86400
  
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.transcode_dlq.arn
    maxReceiveCount     = 3  # Retry 3 times, then DLQ
  })
}

# Dead Letter Queue for failed jobs
resource "aws_sqs_queue" "transcode_dlq" {
  name = "video-transcode-dlq"
}

# ECS Task Definition for transcoding workers (GPU)
resource "aws_ecs_task_definition" "transcoder" {
  family                   = "video-transcoder"
  requires_compatibilities = ["EC2"]  # GPU needs EC2, not Fargate
  
  container_definitions = jsonencode([{
    name  = "transcoder"
    image = "myapp/transcoder:latest"
    
    resourceRequirements = [{
      type  = "GPU"
      value = "1"
    }]
    
    environment = [
      { name = "SQS_QUEUE_URL", value = aws_sqs_queue.transcode_jobs.id },
      { name = "S3_BUCKET", value = aws_s3_bucket.media.id },
    ]
    
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"  = "/ecs/transcoder"
        "awslogs-region" = "us-east-1"
      }
    }
  }])
}

# Auto-scale workers based on queue depth
resource "aws_appautoscaling_policy" "transcode_scaling" {
  name               = "transcode-queue-scaling"
  service_namespace  = "ecs"
  resource_id        = "service/media-cluster/transcoder"
  scalable_dimension = "ecs:service:DesiredCount"
  policy_type        = "TargetTrackingScaling"

  target_tracking_scaling_policy_configuration {
    customized_metric_specification {
      metric_name = "ApproximateNumberOfMessagesVisible"
      namespace   = "AWS/SQS"
      statistic   = "Average"
      dimensions {
        name  = "QueueName"
        value = aws_sqs_queue.transcode_jobs.name
      }
    }
    target_value = 5  # Scale up when > 5 messages waiting
  }
}
```

### Nginx Configuration for Media Serving

```nginx
# Media server with on-the-fly image resizing
server {
    listen 443 ssl http2;
    server_name media.myapp.com;

    # Image resizing using nginx image_filter module
    location ~ ^/images/(?<image_id>[^/]+)/(?<width>\d+)x(?<height>\d+)\.(?<format>jpg|webp|avif)$ {
        # Try cache first
        proxy_cache media_cache;
        proxy_cache_valid 200 30d;
        proxy_cache_key "$image_id-$width-$height-$format";
        
        # Resize on cache miss
        image_filter resize $width $height;
        image_filter_jpeg_quality 85;
        image_filter_webp_quality 80;
        image_filter_buffer 10M;
        
        # Fetch original from S3
        proxy_pass https://myapp-media.s3.amazonaws.com/images/$image_id/original.jpg;
        
        # Cache headers for browser
        add_header Cache-Control "public, max-age=31536000, immutable";
        add_header Vary "Accept";
    }

    # Video streaming (HLS)
    location /videos/ {
        # Proxy to S3 with caching
        proxy_pass https://myapp-media.s3.amazonaws.com/videos/;
        proxy_cache video_cache;
        proxy_cache_valid 200 7d;
        
        # CORS for video player
        add_header Access-Control-Allow-Origin "*";
        add_header Access-Control-Allow-Methods "GET, HEAD, OPTIONS";
        
        # Byte-range requests (essential for video seeking)
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_range;
        
        # Cache control
        add_header Cache-Control "public, max-age=604800"; # 7 days
    }

    # Signed URLs for premium content
    location /premium/ {
        secure_link $arg_token,$arg_expires;
        secure_link_md5 "$uri$arg_expires MY_SECRET_KEY";
        
        if ($secure_link = "") { return 403; }
        if ($secure_link = "0") { return 410; } # Expired
        
        proxy_pass https://myapp-media.s3.amazonaws.com/premium/;
    }
}
```

---

## Real-World Example

### How Netflix Processes and Delivers Video

```
┌─────────────────────────────────────────────────────────────────────┐
│                    NETFLIX VIDEO PIPELINE                            │
│                                                                     │
│  INGEST (from studios):                                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Original Master: 4K HDR, ProRes 4444, ~100 GB per hour    │   │
│  │  Delivered via: Aspera (high-speed file transfer)           │   │
│  │  Stored in: S3 (multiple regions, durability)              │   │
│  └──────────────────────────────┬──────────────────────────────┘   │
│                                 │                                   │
│  ENCODE (their own tool: "Cosmos"):                                │
│  ┌──────────────────────────────▼──────────────────────────────┐   │
│  │                                                             │   │
│  │  PER-TITLE ENCODING:                                        │   │
│  │  Instead of fixed bitrate ladders, Netflix analyzes         │   │
│  │  EACH video's complexity and creates custom encoding:       │   │
│  │                                                             │   │
│  │  Action Movie (complex):     Animated Show (simple):        │   │
│  │  1080p → 8 Mbps needed      1080p → 3 Mbps is enough      │   │
│  │  720p  → 4 Mbps             720p  → 1.5 Mbps              │   │
│  │                                                             │   │
│  │  Result: 30% bandwidth savings across catalog!             │   │
│  │                                                             │   │
│  │  Encodes to: H.264, H.265, VP9, AV1 (newest, best)        │   │
│  │  ~1,200 files per title (all resolutions × codecs × audio) │   │
│  │                                                             │   │
│  └──────────────────────────────┬──────────────────────────────┘   │
│                                 │                                   │
│  DISTRIBUTE (Open Connect CDN):                                    │
│  ┌──────────────────────────────▼──────────────────────────────┐   │
│  │                                                             │   │
│  │  Netflix doesn't use traditional CDNs (Akamai/CloudFront)  │   │
│  │  They built their OWN: "Open Connect"                       │   │
│  │                                                             │   │
│  │  • 17,000+ servers in ISPs worldwide                       │   │
│  │  • Placed INSIDE internet providers (Comcast, Jio, etc.)   │   │
│  │  • Pre-loaded with predicted popular content               │   │
│  │  • During off-peak hours, fill with tomorrow's releases    │   │
│  │                                                             │   │
│  │  ┌─────┐    ┌─────────────────────────────┐               │   │
│  │  │ You │───▶│ Your ISP's Netflix Box      │               │   │
│  │  └─────┘    │ (Open Connect Appliance)     │               │   │
│  │             │ Data never leaves your ISP!  │               │   │
│  │             └─────────────────────────────┘               │   │
│  │                                                             │   │
│  │  Result: 95% of Netflix traffic served from ISP-local      │   │
│  │          servers, NOT from Netflix's data centers           │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  PLAYBACK:                                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  1. Device requests manifest from Netflix API               │   │
│  │  2. Manifest contains URLs to local Open Connect server    │   │
│  │  3. Device downloads segments (DASH format, 4-sec chunks)  │   │
│  │  4. ABR algorithm picks quality based on:                  │   │
│  │     • Measured throughput                                  │   │
│  │     • Buffer level                                         │   │
│  │     • Device capability                                    │   │
│  │     • User's data plan                                     │   │
│  │  5. Seamlessly switches quality mid-stream                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### How Instagram Handles 500M+ Photo Uploads/Day

```
┌─────────────────────────────────────────────────────────────────┐
│              INSTAGRAM IMAGE PIPELINE                            │
│                                                                 │
│  Upload → Process → Store → Serve                              │
│                                                                 │
│  SCALE: 500+ million photos/day                                │
│  STORAGE: Exabytes of images across Facebook/Meta infra        │
│                                                                 │
│  KEY OPTIMIZATIONS:                                             │
│                                                                 │
│  1. PROGRESSIVE JPEG                                           │
│     First: blurry full image loads instantly (tiny)            │
│     Then: full quality loads                                   │
│     User perceives instant loading                            │
│                                                                 │
│  2. CONTENT-AWARE COMPRESSION                                  │
│     AI detects: "This is a face → keep face sharp,            │
│                  compress background more aggressively"         │
│     Result: 20-30% smaller files, same perceived quality      │
│                                                                 │
│  3. FORMAT NEGOTIATION                                         │
│     Client: "I support AVIF, WebP, JPEG"                      │
│     Server: Sends smallest supported format                   │
│     AVIF: 50% smaller than JPEG                               │
│     WebP: 30% smaller than JPEG                               │
│                                                                 │
│  4. SHARDING BY USER ID                                        │
│     Images stored in sharded object storage                   │
│     Key: hash(user_id) / image_id                             │
│     Evenly distributes load across storage nodes              │
│                                                                 │
│  5. HAYSTACK (Custom Object Store)                             │
│     Meta built their own object store optimized for photos    │
│     Problem: Too many small files → too many disk seeks       │
│     Solution: Pack thousands of photos into one large file    │
│     One disk seek → serve any photo from the packed file      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Not using chunked uploads** | Large uploads fail on flaky networks; user loses progress | Implement resumable/chunked upload protocol |
| **Processing synchronously** | User waits minutes for transcoding to finish | Queue jobs; return immediately; notify when done |
| **Single output format** | WebP not supported on old Safari; AVIF not supported everywhere | Generate multiple formats; use `<picture>` tag or content negotiation |
| **No CDN** | Every request hits origin; slow for distant users | Put CDN in front of all static media |
| **Storing originals only** | Must resize on every request (expensive CPU, slow) | Pre-generate common sizes; cache transformed variants |
| **Fixed bitrate encoding** | Simple cartoons waste bandwidth; complex scenes look bad | Use per-title/per-scene encoding (like Netflix) |
| **No processing timeout/retry** | Stuck job blocks the queue forever | Set visibility timeout; use DLQ for failed jobs |
| **Serving from application server** | App server CPU wasted on file serving | Serve directly from S3/CDN; use pre-signed URLs |

### The Thundering Herd on New Content

```
PROBLEM:
──────────────────────────────────────────
Taylor Swift posts a new photo on Instagram.
10 million fans open the app simultaneously.
Photo was JUST uploaded → CDN cache is EMPTY.

All 10 million requests hit origin simultaneously!

                CDN Edge (cache MISS)
                     │
                     │ × 10,000,000 requests
                     ▼
              ┌──────────────┐
              │   ORIGIN     │  ← CRUSHED
              │   SERVER     │
              └──────────────┘

SOLUTION: Request coalescing / cache lock
──────────────────────────────────────────
CDN Edge: "1 million requests for same URL arrived"
          "I'll send ONLY ONE request to origin"
          "Hold all other requests until I get the response"
          "Then serve the cached response to all 1 million"

                CDN Edge (coalescing)
                     │
                     │ × 1 request
                     ▼
              ┌──────────────┐
              │   ORIGIN     │  ← Only 1 request!
              │   SERVER     │
              └──────────────┘
                     │
                     │ response cached
                     ▼
              Serve to all 10M from cache
```

---

## When to Use / When NOT to Use

### ✅ Build a Media Processing Pipeline When:

- Your app handles **user-uploaded images or videos** (social media, e-commerce, CMS)
- You need to serve media in **multiple sizes/formats** for different devices
- You have **global users** who need low-latency media delivery
- Content needs **post-processing** (watermarks, thumbnails, transcoding)
- You're building a **streaming platform** (live or on-demand)

### ❌ When Simple Storage is Enough:

- Internal tools with < 100 users (just serve from app server)
- PDFs/documents that don't need transformation
- Very small scale (< 1000 uploads/day) — simple S3 + CloudFront is fine
- When a managed service (Cloudinary, imgix, Mux) can handle it for you

### Build vs Buy Decision

```
┌─────────────────────────────────────────────────────────────┐
│              BUILD vs BUY DECISION MATRIX                    │
│                                                             │
│                       SCALE                                 │
│              Low ◀────────────────▶ High                    │
│                                                             │
│         Low  ┌───────────────┬───────────────┐             │
│              │  Just use S3  │ Cloudinary /   │             │
│   COMPLEXITY │  + CloudFront │ imgix / Mux    │             │
│              │  (no pipeline)│ (managed)      │             │
│              ├───────────────┼───────────────┤             │
│         High │  Cloudinary / │ Build custom   │             │
│              │  managed SaaS │ pipeline       │             │
│              │  (cheaper     │ (Netflix,      │             │
│              │   than build) │  YouTube,      │             │
│              │               │  Instagram)    │             │
│              └───────────────┴───────────────┘             │
│                                                             │
│  MANAGED SERVICES:                                          │
│  • Images: Cloudinary, imgix, Fastly Image Optimizer       │
│  • Video:  Mux, Cloudflare Stream, AWS MediaConvert        │
│  • CDN:    CloudFront, Fastly, Cloudflare, Akamai          │
│                                                             │
│  BUILD YOUR OWN WHEN:                                       │
│  • Scale > millions of uploads/day                         │
│  • Need custom encoding logic (per-title encoding)         │
│  • Cost of managed service > engineering team cost         │
│  • Need full control over quality/format decisions         │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

- A **media processing pipeline** transforms raw uploads into optimized, multi-format, multi-resolution variants stored in object storage and delivered via CDN.
- Use **chunked/resumable uploads** for large files — network failures should never lose upload progress.
- **Transcoding** is CPU/GPU intensive and must be done **asynchronously** via job queues (SQS, Kafka) with auto-scaling worker pools.
- **Adaptive Bitrate Streaming** (HLS/DASH) splits videos into small segments at multiple quality levels, letting players switch seamlessly based on network conditions.
- **CDN + Origin Shield** serves 95%+ of media requests from edge nodes near users, protecting your origin from traffic spikes.
- **On-the-fly image transformation** (Imgproxy, Thumbor) is simpler to operate but has cold-cache latency; **pre-generation** is better for known sizes at high scale.
- Netflix's **Open Connect** pushes content INTO ISP networks, so video literally never traverses the internet — it's served from a box inside your ISP.

---

## What's Next?

This concludes Part 22: Storage & File Systems. In the next part, we'll explore [Part 23: Networking Deep Dive — Proxies, Tunnels & Protocols](../23-networking/01-forward-vs-reverse-proxy.md), starting with the differences between forward proxies and reverse proxies and how they fit into web application architecture.
