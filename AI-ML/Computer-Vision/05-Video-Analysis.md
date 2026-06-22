# Chapter 05: Video Analysis — Action Recognition, Video Classification, Optical Flow & Tracking

---

## Table of Contents
1. [Video Fundamentals](#1-video-fundamentals)
2. [Optical Flow](#2-optical-flow)
3. [Object Tracking](#3-object-tracking)
4. [Action Recognition](#4-action-recognition)
5. [Video Classification](#5-video-classification)
6. [Temporal Modeling Architectures](#6-temporal-modeling-architectures)
7. [Video Transformers](#7-video-transformers)
8. [Real-World Applications & Deployment](#8-real-world-applications--deployment)
9. [Common Mistakes](#9-common-mistakes)
10. [Interview Questions](#10-interview-questions)
11. [Quick Reference](#11-quick-reference)

---

## 1. Video Fundamentals

### What It Is
A video is simply a sequence of images (called **frames**) displayed rapidly one after another. Think of it like a flip book — each page is a still image, but flipping quickly creates the illusion of motion.

### Why It Matters
- Video is the richest source of visual data — it contains spatial AND temporal information
- Surveillance, autonomous driving, sports analytics, medical imaging all rely on video understanding
- Over 80% of internet traffic is video content (Cisco, 2023)

### How It Works

```
Video = Sequence of Frames over Time

Frame 1    Frame 2    Frame 3    Frame 4    ...    Frame N
┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐         ┌──────┐
│      │   │      │   │      │   │      │         │      │
│ 🚗  │   │  🚗 │   │   🚗│   │    🚗│         │     🚗│
│      │   │      │   │      │   │      │         │      │
└──────┘   └──────┘   └──────┘   └──────┘         └──────┘
  t=0       t=1/30     t=2/30     t=3/30            t=N/30

Time axis →
```

**Key Concepts:**
- **FPS (Frames Per Second):** How many frames are shown per second (24fps for cinema, 30fps for TV, 60fps for gaming)
- **Resolution:** Size of each frame (1920×1080 = Full HD)
- **Temporal Dimension:** The "time" axis that makes video different from images
- **Spatial Dimension:** The height × width of each frame (same as images)

### Code Example — Reading and Processing Video

```python
import cv2
import numpy as np

# Open a video file (or camera with 0)
cap = cv2.VideoCapture('input_video.mp4')

# Get video properties
fps = cap.get(cv2.CAP_PROP_FPS)           # Frames per second
width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))   # Frame width
height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT)) # Frame height
total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))  # Total frames

print(f"Video: {width}x{height} @ {fps}fps, {total_frames} frames")
print(f"Duration: {total_frames/fps:.2f} seconds")

# Read frames one by one
frames = []
while cap.isOpened():
    ret, frame = cap.read()  # ret=True if frame read successfully
    if not ret:
        break
    frames.append(frame)

cap.release()
print(f"Loaded {len(frames)} frames with shape {frames[0].shape}")
```

```python
# Pro Tip: Sample frames at intervals for efficiency
def sample_frames(video_path, num_frames=16):
    """Sample evenly-spaced frames from a video."""
    cap = cv2.VideoCapture(video_path)
    total = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    
    # Calculate indices to sample
    indices = np.linspace(0, total - 1, num_frames, dtype=int)
    
    sampled = []
    for idx in indices:
        cap.set(cv2.CAP_PROP_POS_FRAMES, idx)  # Seek to frame
        ret, frame = cap.read()
        if ret:
            sampled.append(frame)
    
    cap.release()
    return np.array(sampled)  # Shape: (num_frames, H, W, C)

# Usage
clip = sample_frames('video.mp4', num_frames=16)
print(f"Sampled clip shape: {clip.shape}")
```

---

## 2. Optical Flow

### What It Is
Optical flow is the apparent motion of objects (or pixels) between consecutive frames. Imagine drawing tiny arrows on every pixel showing where that pixel "moved to" in the next frame.

### Why It Matters
- Foundation for motion estimation, video stabilization, and action recognition
- Used in autonomous driving to understand scene dynamics
- Enables slow-motion video creation (frame interpolation)
- Critical for video compression (MPEG uses motion estimation)

### How It Works

**The Core Assumption (Brightness Constancy):**
A pixel's intensity doesn't change between frames — it just moves.

$$I(x, y, t) = I(x + \Delta x, y + \Delta y, t + \Delta t)$$

Using Taylor expansion:

$$I_x u + I_y v + I_t = 0$$

Where:
- $I_x, I_y$ = spatial gradients (how intensity changes in x and y)
- $I_t$ = temporal gradient (how intensity changes over time)
- $u, v$ = optical flow (horizontal and vertical velocity)

```
Frame t                Frame t+1              Optical Flow
┌──────────────┐      ┌──────────────┐       ┌──────────────┐
│              │      │              │       │   → → →      │
│    ●         │  →   │       ●      │  =    │    ↗ → →     │
│              │      │              │       │              │
└──────────────┘      └──────────────┘       └──────────────┘

Each arrow shows the displacement of that pixel
```

**Two Main Approaches:**

| Method | Type | Speed | Accuracy | Use Case |
|--------|------|-------|----------|----------|
| Lucas-Kanade | Sparse | Fast | Good for features | Tracking key points |
| Farneback | Dense | Medium | Good overall | Full motion field |
| RAFT (Deep) | Dense | Slow | State-of-art | Research, accuracy-critical |
| FlowNet | Dense | Fast | Good | Real-time applications |

### Code Examples

```python
import cv2
import numpy as np

# ============================================
# SPARSE Optical Flow (Lucas-Kanade)
# Tracks specific points across frames
# ============================================

cap = cv2.VideoCapture('video.mp4')

# Read first frame
ret, old_frame = cap.read()
old_gray = cv2.cvtColor(old_frame, cv2.COLOR_BGR2GRAY)

# Detect good features to track (Shi-Tomasi corners)
feature_params = dict(
    maxCorners=100,       # Maximum number of points to track
    qualityLevel=0.3,    # Minimum quality (0-1)
    minDistance=7,       # Minimum distance between points
    blockSize=7          # Neighborhood size for corner detection
)

# Parameters for Lucas-Kanade optical flow
lk_params = dict(
    winSize=(15, 15),     # Search window size
    maxLevel=2,          # Pyramid levels (handle large motions)
    criteria=(cv2.TERM_CRITERIA_EPS | cv2.TERM_CRITERIA_COUNT, 10, 0.03)
)

# Find initial points to track
p0 = cv2.goodFeaturesToTrack(old_gray, mask=None, **feature_params)

# Track points across frames
while cap.isOpened():
    ret, frame = cap.read()
    if not ret:
        break
    
    frame_gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    
    # Calculate optical flow (find new positions of tracked points)
    p1, status, error = cv2.calcOpticalFlowPyrLK(
        old_gray, frame_gray, p0, None, **lk_params
    )
    
    # Select good points (status=1 means successfully tracked)
    good_new = p1[status == 1]
    good_old = p0[status == 1]
    
    # Draw the tracks
    for i, (new, old) in enumerate(zip(good_new, good_old)):
        a, b = new.ravel().astype(int)
        c, d = old.ravel().astype(int)
        # Draw line from old to new position
        frame = cv2.line(frame, (c, d), (a, b), (0, 255, 0), 2)
        # Draw circle at new position
        frame = cv2.circle(frame, (a, b), 5, (0, 0, 255), -1)
    
    # Update for next iteration
    old_gray = frame_gray.copy()
    p0 = good_new.reshape(-1, 1, 2)

cap.release()
```

```python
# ============================================
# DENSE Optical Flow (Farneback)
# Computes flow for EVERY pixel
# ============================================

cap = cv2.VideoCapture('video.mp4')
ret, frame1 = cap.read()
prvs = cv2.cvtColor(frame1, cv2.COLOR_BGR2GRAY)

while cap.isOpened():
    ret, frame2 = cap.read()
    if not ret:
        break
    
    next_gray = cv2.cvtColor(frame2, cv2.COLOR_BGR2GRAY)
    
    # Calculate dense optical flow
    flow = cv2.calcOpticalFlowFarneback(
        prvs, next_gray,
        None,           # Output flow (None = allocate new)
        pyr_scale=0.5,  # Image scale (<1 to build pyramids)
        levels=3,       # Number of pyramid levels
        winsize=15,     # Averaging window size
        iterations=3,   # Number of iterations at each level
        poly_n=5,       # Size of pixel neighborhood for polynomial
        poly_sigma=1.2, # Gaussian std for polynomial expansion
        flags=0
    )
    # flow shape: (H, W, 2) — flow[y,x,0]=dx, flow[y,x,1]=dy
    
    # Visualize: Convert flow to HSV color
    # Hue = direction of motion, Value = magnitude of motion
    magnitude, angle = cv2.cartToPolar(flow[..., 0], flow[..., 1])
    
    hsv = np.zeros_like(frame2)
    hsv[..., 0] = angle * 180 / np.pi / 2   # Hue: direction
    hsv[..., 1] = 255                         # Saturation: full
    hsv[..., 2] = cv2.normalize(magnitude, None, 0, 255, cv2.NORM_MINMAX)  # Value: speed
    
    flow_rgb = cv2.cvtColor(hsv, cv2.COLOR_HSV2BGR)
    
    prvs = next_gray

cap.release()
```

```python
# ============================================
# Deep Learning Optical Flow with RAFT
# State-of-the-art accuracy
# ============================================

import torch
from torchvision.models.optical_flow import raft_large, Raft_Large_Weights
from torchvision.transforms.functional import resize
import torchvision.transforms.functional as F

# Load pretrained RAFT model
weights = Raft_Large_Weights.DEFAULT
model = raft_large(weights=weights).eval().cuda()
transforms = weights.transforms()

def compute_flow_raft(frame1, frame2):
    """Compute optical flow between two frames using RAFT."""
    # Convert BGR (OpenCV) to RGB tensors
    img1 = torch.from_numpy(frame1[..., ::-1].copy()).permute(2, 0, 1).float()
    img2 = torch.from_numpy(frame2[..., ::-1].copy()).permute(2, 0, 1).float()
    
    # Apply model-specific transforms
    img1, img2 = transforms(img1, img2)
    
    # Add batch dimension and move to GPU
    img1 = img1[None].cuda()
    img2 = img2[None].cuda()
    
    # Compute flow (returns list of flows at different iterations)
    with torch.no_grad():
        flows = model(img1, img2)
    
    # Take final flow prediction
    flow = flows[-1][0].permute(1, 2, 0).cpu().numpy()
    return flow  # Shape: (H, W, 2)

# Usage
# flow = compute_flow_raft(frame1, frame2)
# print(f"Flow shape: {flow.shape}, range: [{flow.min():.1f}, {flow.max():.1f}]")
```

> **Pro Tip:** For real-time applications, use GPU-accelerated flow (NVIDIA Optical Flow SDK) or lightweight models like FlowNet2-S. RAFT is accurate but slow for production.

---

## 3. Object Tracking

### What It Is
Object tracking means following a specific object across video frames. You identify the object once (in the first frame), and the tracker keeps finding it in subsequent frames — like keeping your eye on one person in a crowd.

### Why It Matters
- Surveillance: Follow suspicious individuals
- Sports analytics: Track players and ball
- Autonomous driving: Track vehicles, pedestrians
- AR/VR: Track hand/face movements
- Traffic analysis: Count vehicles, measure speed

### How It Works

**Tracking vs. Detection:**
```
Detection (per-frame):          Tracking (across frames):
Frame 1: Find all objects      Frame 1: Find object → ID=1
Frame 2: Find all objects      Frame 2: Where did ID=1 go?
Frame 3: Find all objects      Frame 3: Where did ID=1 go?
                               (Uses motion model + appearance)
```

**Types of Tracking:**

| Type | Description | Example |
|------|-------------|---------|
| Single Object Tracking (SOT) | Track ONE object | Following a specific car |
| Multi-Object Tracking (MOT) | Track MANY objects | All people in a scene |
| Tracking-by-Detection | Detect → Associate | SORT, DeepSORT, ByteTrack |

**Single Object Tracking Algorithms:**

| Tracker | Speed | Accuracy | When to Use |
|---------|-------|----------|-------------|
| CSRT | Slow | High | When accuracy matters, object changes appearance |
| KCF | Fast | Medium | Real-time with limited compute |
| MOSSE | Very Fast | Low | Ultra-fast tracking, simple motion |
| MedianFlow | Fast | Medium | Predictable, smooth motion |
| SiamFC/SiamRPN | Fast | High | Deep learning, GPU available |

### Code Examples

```python
import cv2

# ============================================
# Single Object Tracking with OpenCV
# ============================================

# Available trackers in OpenCV
TRACKERS = {
    'csrt': cv2.TrackerCSRT_create,
    'kcf': cv2.TrackerKCF_create,
}

# Create tracker
tracker = cv2.TrackerCSRT_create()

# Open video
cap = cv2.VideoCapture('video.mp4')
ret, frame = cap.read()

# Select ROI (Region of Interest) to track
# In production, this comes from a detector or user input
bbox = cv2.selectROI("Select Object", frame, fromCenter=False)
cv2.destroyAllWindows()

# Initialize tracker with first frame and bounding box
tracker.init(frame, bbox)

while cap.isOpened():
    ret, frame = cap.read()
    if not ret:
        break
    
    # Update tracker
    success, bbox = tracker.update(frame)
    
    if success:
        # Draw bounding box
        x, y, w, h = [int(v) for v in bbox]
        cv2.rectangle(frame, (x, y), (x + w, y + h), (0, 255, 0), 2)
        cv2.putText(frame, "Tracking", (x, y - 10),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)
    else:
        cv2.putText(frame, "Lost!", (50, 50),
                    cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)

cap.release()
```

```python
# ============================================
# Multi-Object Tracking with DeepSORT
# Detection + Tracking pipeline
# ============================================

# pip install deep-sort-realtime

from deep_sort_realtime.deepsort_tracker import DeepSort
import cv2

# Initialize DeepSORT tracker
tracker = DeepSort(
    max_age=30,          # Frames to keep lost track alive
    n_init=3,            # Frames before confirming a track
    max_iou_distance=0.7,  # IoU threshold for matching
    max_cosine_distance=0.2,  # Appearance threshold
)

cap = cv2.VideoCapture('video.mp4')

while cap.isOpened():
    ret, frame = cap.read()
    if not ret:
        break
    
    # Step 1: Detect objects (using any detector)
    # Format: [[x1, y1, width, height, confidence, class_id], ...]
    detections = detect_objects(frame)  # Your detector function
    
    # Step 2: Convert detections to DeepSORT format
    # Each detection: ([left, top, w, h], confidence, class_name)
    det_list = []
    for det in detections:
        x1, y1, w, h, conf, cls = det
        det_list.append(([x1, y1, w, h], conf, str(cls)))
    
    # Step 3: Update tracker with new detections
    tracks = tracker.update_tracks(det_list, frame=frame)
    
    # Step 4: Draw confirmed tracks
    for track in tracks:
        if not track.is_confirmed():
            continue
        
        track_id = track.track_id
        bbox = track.to_ltrb()  # [left, top, right, bottom]
        
        x1, y1, x2, y2 = [int(v) for v in bbox]
        cv2.rectangle(frame, (x1, y1), (x2, y2), (0, 255, 0), 2)
        cv2.putText(frame, f"ID: {track_id}", (x1, y1 - 10),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.6, (0, 255, 0), 2)

cap.release()
```

```python
# ============================================
# ByteTrack — State-of-the-art MOT
# Key insight: Use ALL detections (high + low confidence)
# ============================================

# pip install bytetracker

import numpy as np

class SimpleByteTrack:
    """Simplified ByteTrack logic for understanding."""
    
    def __init__(self, track_thresh=0.5, match_thresh=0.8):
        self.track_thresh = track_thresh  # High confidence threshold
        self.match_thresh = match_thresh  # IoU matching threshold
        self.tracks = []
        self.track_id_count = 0
    
    def update(self, detections, scores):
        """
        ByteTrack's key innovation:
        1. Match high-confidence detections with existing tracks
        2. Match LOW-confidence detections with UNMATCHED tracks
        This recovers tracks that were briefly occluded!
        """
        # Split into high and low confidence
        high_mask = scores > self.track_thresh
        high_dets = detections[high_mask]
        low_dets = detections[~high_mask]
        
        # First association: high-confidence with all tracks
        # (using IoU or appearance matching)
        matched_high, unmatched_tracks, unmatched_dets = self._match(
            self.tracks, high_dets
        )
        
        # Second association: low-confidence with UNMATCHED tracks
        # This is ByteTrack's key contribution!
        matched_low, still_unmatched, _ = self._match(
            unmatched_tracks, low_dets
        )
        
        # Update matched tracks, create new ones, remove lost ones
        # ... (simplified)
        
        return self.tracks
    
    def _match(self, tracks, detections):
        """Match tracks to detections using IoU."""
        # Compute IoU matrix and use Hungarian algorithm
        # ... (simplified)
        pass
```

> **Important:** DeepSORT uses appearance features (Re-ID) + motion (Kalman Filter) for matching. ByteTrack primarily uses IoU but exploits low-confidence detections. In practice, ByteTrack is faster and often better.

---

## 4. Action Recognition

### What It Is
Action recognition means understanding WHAT a person (or object) is doing in a video. Instead of just detecting a person, you determine they are "running", "cooking", "playing guitar", etc.

### Why It Matters
- Security: Detect fights, falls, suspicious behavior
- Healthcare: Monitor patient activities, detect falls in elderly
- Sports: Automatic game highlights, player performance analysis
- Content: Video search, auto-tagging, content moderation
- Robotics: Understand human actions for collaboration

### How It Works

**The Challenge:**
Unlike image classification (spatial only), action recognition requires understanding:
1. **Spatial info:** What objects/people are in the scene
2. **Temporal info:** How they move over time

```
Image Classification:       Action Recognition:
Single frame → Label       Multiple frames → Action

┌──────┐                   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│ 🏃  │ → "Person"       │ 🏃  │→│  🏃 │→│   🏃│→│    🏃│ → "Running"
└──────┘                   └──────┘ └──────┘ └──────┘ └──────┘
                           Need temporal context!
```

**Evolution of Approaches:**

```
Hand-crafted     →    Two-Stream    →    3D CNNs    →    Transformers
(HOG+SVM)            (Spatial+Flow)     (C3D, I3D)      (TimeSformer, ViViT)
2010-2014             2014-2016         2016-2020        2020-present
```

**Two-Stream Architecture:**
```
                    ┌─────────────────────┐
                    │   Fusion / Late     │
                    │   Combination       │
                    └──────┬──────┬───────┘
                           │      │
              ┌────────────┘      └────────────┐
              │                                │
     ┌────────┴────────┐           ┌───────────┴─────────┐
     │  Spatial Stream  │           │  Temporal Stream     │
     │  (Appearance)    │           │  (Motion)            │
     │  Regular CNN     │           │  CNN on Optical Flow │
     └────────┬────────┘           └───────────┬─────────┘
              │                                │
        Single RGB Frame              Stacked Optical Flow
                                      (10 consecutive frames)
```

**3D Convolutions (C3D/I3D):**
```
2D Conv (images):               3D Conv (video):
┌───┐                           ┌───┐
│3×3│ filter                    │3×3×3│ filter (space + time)
└───┘                           └─────┘

Slides over H × W               Slides over H × W × T
                                 Captures spatiotemporal patterns!
```

### Code Examples

```python
# ============================================
# Action Recognition with Pre-trained Models
# Using torchvision's video models
# ============================================

import torch
import torchvision.transforms as T
from torchvision.models.video import r3d_18, R3D_18_Weights
from torchvision.models.video import mvit_v2_s, MViT_V2_S_Weights
import numpy as np

# --- Method 1: ResNet3D (3D CNN) ---

# Load pretrained model
weights = R3D_18_Weights.DEFAULT
model = r3d_18(weights=weights).eval()
preprocess = weights.transforms()

# Prepare video clip
# Input shape: (C, T, H, W) = (3, 16, 112, 112)
def prepare_clip(frames, num_frames=16, size=112):
    """Prepare a video clip for the model."""
    # Sample num_frames evenly
    indices = np.linspace(0, len(frames) - 1, num_frames, dtype=int)
    clip = [frames[i] for i in indices]
    
    # Convert to tensor: (T, H, W, C) → (C, T, H, W)
    clip = torch.tensor(np.array(clip), dtype=torch.float32)
    clip = clip.permute(3, 0, 1, 2)  # (C, T, H, W)
    
    # Resize and normalize
    clip = clip / 255.0
    clip = torch.nn.functional.interpolate(
        clip.unsqueeze(0),  # Add batch: (1, C, T, H, W)
        size=(num_frames, size, size),
        mode='trilinear',
        align_corners=False
    ).squeeze(0)  # Remove batch: (C, T, H, W)
    
    return clip

# Classify action
def predict_action(frames, model, weights):
    """Predict action from video frames."""
    clip = prepare_clip(frames)
    clip = preprocess(clip)  # Apply model-specific transforms
    
    with torch.no_grad():
        output = model(clip.unsqueeze(0))  # Add batch dim
        probs = torch.softmax(output, dim=1)
    
    # Get top-5 predictions
    top5_prob, top5_idx = torch.topk(probs, 5)
    categories = weights.meta["categories"]
    
    results = []
    for prob, idx in zip(top5_prob[0], top5_idx[0]):
        results.append((categories[idx], prob.item()))
    
    return results

# Example usage
# results = predict_action(video_frames, model, weights)
# for action, confidence in results:
#     print(f"{action}: {confidence:.3f}")
```

```python
# --- Method 2: MViT v2 (Video Transformer — SOTA) ---

weights = MViT_V2_S_Weights.DEFAULT
model = mvit_v2_s(weights=weights).eval()
preprocess = weights.transforms()

# MViT expects (C, T, H, W) = (3, 16, 224, 224)
# It uses multi-scale attention for efficient video understanding

def predict_with_mvit(video_path):
    """Action recognition with MViT-v2."""
    frames = sample_frames(video_path, num_frames=16)
    
    # Prepare input
    clip = torch.from_numpy(frames).float().permute(3, 0, 1, 2) / 255.0
    clip = torch.nn.functional.interpolate(
        clip.unsqueeze(0), size=(16, 224, 224),
        mode='trilinear', align_corners=False
    )
    clip = preprocess(clip.squeeze(0))
    
    with torch.no_grad():
        output = model(clip.unsqueeze(0))
        probs = torch.softmax(output, dim=1)
    
    categories = weights.meta["categories"]
    top5 = torch.topk(probs, 5)
    
    for prob, idx in zip(top5.values[0], top5.indices[0]):
        print(f"  {categories[idx]:30s} {prob.item():.3f}")

# predict_with_mvit('cooking_video.mp4')
```

```python
# ============================================
# Skeleton-based Action Recognition
# Using body keypoints instead of raw pixels
# ============================================

# Advantages of skeleton-based:
# - Robust to appearance changes (clothing, lighting)
# - Much smaller input (17 keypoints vs full image)
# - Interpretable

import numpy as np

class SkeletonActionRecognizer:
    """
    Simple skeleton-based action recognition using joint angles and velocities.
    For production, use ST-GCN or PoseC3D.
    """
    
    def __init__(self):
        # Define body joint connections
        self.JOINTS = {
            'nose': 0, 'left_shoulder': 5, 'right_shoulder': 6,
            'left_elbow': 7, 'right_elbow': 8,
            'left_wrist': 9, 'right_wrist': 10,
            'left_hip': 11, 'right_hip': 12,
            'left_knee': 13, 'right_knee': 14,
            'left_ankle': 15, 'right_ankle': 16
        }
    
    def compute_features(self, keypoints_sequence):
        """
        Compute motion features from a sequence of skeletons.
        keypoints_sequence: (T, 17, 2) — T frames, 17 joints, (x,y)
        """
        features = {}
        
        # 1. Joint velocities (temporal difference)
        velocities = np.diff(keypoints_sequence, axis=0)  # (T-1, 17, 2)
        features['mean_velocity'] = np.mean(np.linalg.norm(velocities, axis=-1))
        
        # 2. Wrist velocity (for hand actions)
        wrist_vel = velocities[:, [9, 10], :]  # Left and right wrist
        features['wrist_speed'] = np.mean(np.linalg.norm(wrist_vel, axis=-1))
        
        # 3. Body center of mass movement
        hips = keypoints_sequence[:, [11, 12], :]
        com = np.mean(hips, axis=1)  # (T, 2)
        com_vel = np.diff(com, axis=0)
        features['body_movement'] = np.mean(np.linalg.norm(com_vel, axis=-1))
        
        # 4. Vertical extent (standing vs sitting vs lying)
        y_range = (keypoints_sequence[:, :, 1].max(axis=1) - 
                   keypoints_sequence[:, :, 1].min(axis=1))
        features['vertical_extent'] = np.mean(y_range)
        
        return features
    
    def classify_simple(self, features):
        """Rule-based classification (for demonstration)."""
        if features['body_movement'] > 20 and features['mean_velocity'] > 15:
            return "Running"
        elif features['wrist_speed'] > 10 and features['body_movement'] < 5:
            return "Waving"
        elif features['vertical_extent'] < 100:
            return "Sitting"
        else:
            return "Standing"
```

---

## 5. Video Classification

### What It Is
Video classification assigns a label to an entire video clip. While action recognition focuses on human actions, video classification is broader — it can classify any type of video content (sports highlights, news, music videos, etc.).

### Why It Matters
- Content moderation at scale (YouTube processes 500+ hours/min)
- Video recommendation systems
- Automated video tagging for search
- Medical video analysis (endoscopy, ultrasound)

### How It Works

**Key Approaches:**

```
Approach 1: Frame-level → Aggregate
┌───────────────────────────────────────────────────────┐
│ Frame 1 → CNN → feat₁ ─┐                             │
│ Frame 2 → CNN → feat₂ ─┼─→ Pool/LSTM → Classifier   │
│ Frame 3 → CNN → feat₃ ─┘                             │
└───────────────────────────────────────────────────────┘

Approach 2: 3D CNN (joint space-time)
┌───────────────────────────────────────────────────────┐
│ Video Clip (T×H×W) → 3D CNN → Features → Classifier  │
└───────────────────────────────────────────────────────┘

Approach 3: Transformer (attention over space+time)
┌───────────────────────────────────────────────────────┐
│ Video → Patches → Embed → Transformer → Classifier   │
└───────────────────────────────────────────────────────┘
```

### Code Examples

```python
# ============================================
# Video Classification: Frame-level + Temporal Pooling
# Simple but effective baseline
# ============================================

import torch
import torch.nn as nn
from torchvision.models import resnet50, ResNet50_Weights

class FrameLevelClassifier(nn.Module):
    """
    Extract features from individual frames with a CNN,
    then aggregate temporally.
    """
    
    def __init__(self, num_classes=400, num_frames=16):
        super().__init__()
        
        # Pretrained CNN backbone (freeze early layers)
        backbone = resnet50(weights=ResNet50_Weights.DEFAULT)
        self.features = nn.Sequential(*list(backbone.children())[:-1])
        self.feat_dim = 2048
        
        # Temporal aggregation
        self.temporal_pool = nn.AdaptiveAvgPool1d(1)
        
        # Classifier head
        self.classifier = nn.Sequential(
            nn.Dropout(0.5),
            nn.Linear(self.feat_dim, num_classes)
        )
    
    def forward(self, x):
        """
        x: (batch, T, C, H, W) — T frames per video
        """
        B, T, C, H, W = x.shape
        
        # Reshape to process all frames at once
        x = x.view(B * T, C, H, W)
        
        # Extract features for each frame
        features = self.features(x)  # (B*T, 2048, 1, 1)
        features = features.view(B, T, self.feat_dim)  # (B, T, 2048)
        
        # Temporal pooling: (B, T, 2048) → (B, 2048)
        features = features.permute(0, 2, 1)  # (B, 2048, T)
        features = self.temporal_pool(features).squeeze(-1)  # (B, 2048)
        
        # Classify
        output = self.classifier(features)
        return output

# Usage
model = FrameLevelClassifier(num_classes=101)  # UCF-101 dataset
# clip = torch.randn(2, 16, 3, 224, 224)  # Batch of 2, 16 frames each
# output = model(clip)  # (2, 101)
```

```python
# ============================================
# Video Classification with LSTM for temporal modeling
# Better than simple pooling for order-dependent actions
# ============================================

class VideoLSTMClassifier(nn.Module):
    """
    CNN (spatial) + LSTM (temporal) for video classification.
    The LSTM captures the ORDER of events, not just their presence.
    """
    
    def __init__(self, num_classes=101, hidden_size=512, num_layers=2):
        super().__init__()
        
        # Spatial feature extractor
        backbone = resnet50(weights=ResNet50_Weights.DEFAULT)
        self.cnn = nn.Sequential(*list(backbone.children())[:-1])
        self.feat_dim = 2048
        
        # Temporal modeling with LSTM
        self.lstm = nn.LSTM(
            input_size=self.feat_dim,
            hidden_size=hidden_size,
            num_layers=num_layers,
            batch_first=True,
            dropout=0.3,
            bidirectional=True  # Look at past AND future
        )
        
        # Classifier (bidirectional → 2x hidden_size)
        self.classifier = nn.Sequential(
            nn.Dropout(0.5),
            nn.Linear(hidden_size * 2, num_classes)
        )
    
    def forward(self, x):
        """x: (batch, T, C, H, W)"""
        B, T, C, H, W = x.shape
        
        # Extract frame features
        x = x.view(B * T, C, H, W)
        with torch.no_grad():  # Freeze CNN for efficiency
            features = self.cnn(x).view(B, T, self.feat_dim)
        
        # LSTM over time
        lstm_out, (hidden, cell) = self.lstm(features)
        # lstm_out: (B, T, hidden*2)
        
        # Use last timestep output
        output = self.classifier(lstm_out[:, -1, :])
        return output
```

```python
# ============================================
# SlowFast Network Concept
# Two pathways: Slow (appearance) + Fast (motion)
# ============================================

class SlowFastConcept(nn.Module):
    """
    Conceptual SlowFast implementation.
    - Slow pathway: Low frame rate, high channels (appearance)
    - Fast pathway: High frame rate, low channels (motion)
    
    Real implementation: use pytorchvideo
    """
    
    def __init__(self, num_classes=400):
        super().__init__()
        
        # Slow pathway: processes 1/8 of frames (e.g., 8 out of 64)
        # High capacity for spatial semantics
        self.slow_path = nn.Sequential(
            nn.Conv3d(3, 64, kernel_size=(1, 7, 7), stride=(1, 2, 2), padding=(0, 3, 3)),
            nn.BatchNorm3d(64),
            nn.ReLU(),
            # ... more layers
        )
        
        # Fast pathway: processes ALL frames (e.g., 64)
        # Low capacity (1/8 channels) for temporal dynamics
        self.fast_path = nn.Sequential(
            nn.Conv3d(3, 8, kernel_size=(5, 7, 7), stride=(1, 2, 2), padding=(2, 3, 3)),
            nn.BatchNorm3d(8),
            nn.ReLU(),
            # ... more layers
        )
        
        # Lateral connections: Fast → Slow (share temporal info)
        self.lateral = nn.Conv3d(8, 16, kernel_size=(5, 1, 1), stride=(8, 1, 1), padding=(2, 0, 0))
    
    def forward(self, x_slow, x_fast):
        """
        x_slow: (B, 3, 8, H, W)   — 8 frames
        x_fast: (B, 3, 64, H, W)  — 64 frames
        """
        slow_feat = self.slow_path(x_slow)
        fast_feat = self.fast_path(x_fast)
        
        # Lateral connection: transfer temporal info from fast to slow
        lateral = self.lateral(fast_feat)
        slow_feat = torch.cat([slow_feat, lateral], dim=1)
        
        # ... rest of network
        return slow_feat, fast_feat
```

> **Pro Tip:** For production video classification, use `pytorchvideo` library which has pre-trained SlowFast, X3D, MViT models ready to use. Training from scratch is rarely needed.

---

## 6. Temporal Modeling Architectures

### What It Is
Temporal modeling is about understanding HOW things change over time in a video. It's the "time" component that makes video different from a bag of images.

### Why It Matters
The same set of frames in different ORDER can mean completely different things:
- "Picking up" vs "Putting down" — same frames, reversed order
- "Walking" requires consistent motion direction over time
- A "goal" in soccer is defined by the temporal sequence of events

### Comparison of Temporal Models

| Architecture | How It Models Time | Strengths | Weaknesses |
|---|---|---|---|
| 3D CNN (C3D, I3D) | Local temporal convolutions | Simple, captures short-term motion | Limited long-range, expensive |
| Two-Stream | Explicit optical flow input | Proven effective, intuitive | Need to compute flow (slow) |
| LSTM/GRU | Sequential hidden states | Variable length, long-range | Slow training, vanishing gradients |
| Temporal Shift (TSM) | Shift channels along time | Zero-cost temporal modeling! | Limited temporal receptive field |
| SlowFast | Dual frame rates | Best of both worlds | Complex architecture |
| Video Transformer | Self-attention over time | Global context, flexible | Quadratic cost, needs lots of data |

### Code Example — Temporal Shift Module (TSM)

```python
# ============================================
# Temporal Shift Module (TSM)
# FREE temporal modeling — just shift channels!
# ============================================

import torch
import torch.nn as nn

class TemporalShift(nn.Module):
    """
    Shift part of the channels along the temporal dimension.
    This gives 2D CNNs temporal reasoning ability for FREE
    (no extra parameters or computation).
    
    How it works:
    - 1/8 channels shift forward (see past)
    - 1/8 channels shift backward (see future)
    - 6/8 channels stay (current frame)
    """
    
    def __init__(self, n_segment=8, shift_div=8):
        super().__init__()
        self.n_segment = n_segment  # Number of frames
        self.shift_div = shift_div  # What fraction to shift
    
    def forward(self, x):
        """
        x: (B*T, C, H, W) — frames from all videos stacked
        """
        BT, C, H, W = x.shape
        B = BT // self.n_segment
        T = self.n_segment
        
        # Reshape to separate time dimension
        x = x.view(B, T, C, H, W)
        
        # Determine shift amounts
        fold = C // self.shift_div  # Number of channels to shift
        
        out = x.clone()
        
        # Shift forward: current frame gets PAST info
        out[:, 1:, :fold, :, :] = x[:, :-1, :fold, :, :]
        
        # Shift backward: current frame gets FUTURE info  
        out[:, :-1, fold:2*fold, :, :] = x[:, 1:, fold:2*fold, :, :]
        
        # Remaining channels unchanged (current frame info)
        
        return out.view(BT, C, H, W)


class TSMResNet(nn.Module):
    """ResNet with Temporal Shift — 2D CNN with video understanding!"""
    
    def __init__(self, num_classes=101, num_segments=8):
        super().__init__()
        self.num_segments = num_segments
        
        # Standard 2D ResNet backbone
        from torchvision.models import resnet50
        base = resnet50(pretrained=True)
        
        # Insert TSM before each residual block
        self.tsm = TemporalShift(n_segment=num_segments)
        self.backbone = nn.Sequential(*list(base.children())[:-1])
        self.fc = nn.Linear(2048, num_classes)
    
    def forward(self, x):
        """x: (B, T, C, H, W)"""
        B, T, C, H, W = x.shape
        x = x.view(B * T, C, H, W)
        
        # Apply temporal shift
        x = self.tsm(x)
        
        # Standard 2D CNN processing
        x = self.backbone(x)  # (B*T, 2048, 1, 1)
        x = x.view(B, T, -1).mean(dim=1)  # Average over time
        
        return self.fc(x)
```

---

## 7. Video Transformers

### What It Is
Video Transformers apply the self-attention mechanism (originally from NLP) to video data. Instead of processing local neighborhoods like CNNs, they can attend to any part of the video — any pixel in any frame can "look at" any other pixel in any frame.

### Why It Matters
- Current SOTA on most video benchmarks (Kinetics-400/600/700)
- Can capture long-range dependencies (beginning of video → end)
- Unified architecture for images AND video (less engineering)
- Foundation for video-language models (ask questions about videos)

### How It Works

```
Input Video: (T frames × H × W × 3)
        │
        ▼
┌─────────────────────────┐
│   Patch Embedding       │  Divide each frame into patches
│   (T×H×W) → (N tokens) │  Each patch → token (like a word)
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   Positional Encoding   │  Add space + time position info
│   (spatial + temporal)  │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   Transformer Layers    │  Self-attention across all tokens
│   (Multi-head Attention │  Each patch can attend to any
│    + Feed Forward)      │  other patch in any frame!
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   Classification Head   │  [CLS] token → class prediction
└─────────────────────────┘
```

**The Attention Challenge:**
With T=16 frames of 14×14 patches = 3136 tokens.
Self-attention is $O(n^2)$, so 3136² ≈ 10M operations per layer!

**Solutions — Factorized Attention:**

```
Full Attention (impractical):
Every token attends to ALL other tokens (3136 × 3136)

TimeSformer — Divided Attention:
Step 1: Spatial attention (within each frame: 196 × 196)
Step 2: Temporal attention (same position across frames: 16 × 16)

ViViT — Factorized Encoder:
Encoder 1: Spatial transformer per frame
Encoder 2: Temporal transformer across frame representations
```

### Code Example

```python
# ============================================
# Simplified Video Transformer (TimeSformer-style)
# ============================================

import torch
import torch.nn as nn
import torch.nn.functional as F

class DividedSpaceTimeAttention(nn.Module):
    """
    TimeSformer's Divided Space-Time Attention.
    Instead of attending to all tokens at once (expensive),
    do spatial attention then temporal attention separately.
    """
    
    def __init__(self, dim=768, num_heads=12, num_frames=8, num_patches=196):
        super().__init__()
        self.num_frames = num_frames
        self.num_patches = num_patches
        
        # Temporal attention (across frames, same spatial position)
        self.temporal_attn = nn.MultiheadAttention(dim, num_heads, batch_first=True)
        self.temporal_norm = nn.LayerNorm(dim)
        
        # Spatial attention (within frame, all positions)
        self.spatial_attn = nn.MultiheadAttention(dim, num_heads, batch_first=True)
        self.spatial_norm = nn.LayerNorm(dim)
        
        # Feed-forward
        self.ffn = nn.Sequential(
            nn.Linear(dim, dim * 4),
            nn.GELU(),
            nn.Linear(dim * 4, dim),
            nn.Dropout(0.1)
        )
        self.ffn_norm = nn.LayerNorm(dim)
    
    def forward(self, x):
        """
        x: (B, T*P, D) where T=frames, P=patches per frame, D=dimension
        """
        B, N, D = x.shape
        T = self.num_frames
        P = self.num_patches
        
        # Reshape for temporal attention: (B*P, T, D)
        x_t = x.view(B, T, P, D).permute(0, 2, 1, 3).reshape(B * P, T, D)
        
        # Temporal self-attention (each spatial position attends across time)
        residual = x_t
        x_t = self.temporal_norm(x_t)
        x_t, _ = self.temporal_attn(x_t, x_t, x_t)
        x_t = x_t + residual
        
        # Reshape back and prepare for spatial attention: (B*T, P, D)
        x_s = x_t.reshape(B, P, T, D).permute(0, 2, 1, 3).reshape(B * T, P, D)
        
        # Spatial self-attention (each frame's patches attend to each other)
        residual = x_s
        x_s = self.spatial_norm(x_s)
        x_s, _ = self.spatial_attn(x_s, x_s, x_s)
        x_s = x_s + residual
        
        # Reshape back to (B, T*P, D)
        x = x_s.reshape(B, T, P, D).reshape(B, T * P, D)
        
        # Feed-forward
        x = x + self.ffn(self.ffn_norm(x))
        
        return x


class SimpleVideoTransformer(nn.Module):
    """Simplified Video Transformer for understanding."""
    
    def __init__(self, num_classes=400, num_frames=8, 
                 img_size=224, patch_size=16, dim=768, depth=12):
        super().__init__()
        
        self.num_frames = num_frames
        num_patches = (img_size // patch_size) ** 2  # 196 for 224/16
        
        # Patch embedding: image patches → tokens
        self.patch_embed = nn.Conv2d(
            3, dim, kernel_size=patch_size, stride=patch_size
        )
        
        # Position embeddings (spatial + temporal)
        self.spatial_pos = nn.Parameter(torch.randn(1, num_patches, dim) * 0.02)
        self.temporal_pos = nn.Parameter(torch.randn(1, num_frames, dim) * 0.02)
        
        # CLS token for classification
        self.cls_token = nn.Parameter(torch.randn(1, 1, dim) * 0.02)
        
        # Transformer layers
        self.layers = nn.ModuleList([
            DividedSpaceTimeAttention(dim, num_heads=12, 
                                      num_frames=num_frames, 
                                      num_patches=num_patches)
            for _ in range(depth)
        ])
        
        # Classification head
        self.norm = nn.LayerNorm(dim)
        self.head = nn.Linear(dim, num_classes)
    
    def forward(self, x):
        """x: (B, T, C, H, W)"""
        B, T, C, H, W = x.shape
        
        # Embed patches from each frame
        x = x.view(B * T, C, H, W)
        x = self.patch_embed(x)  # (B*T, dim, h, w)
        x = x.flatten(2).transpose(1, 2)  # (B*T, num_patches, dim)
        
        P = x.shape[1]
        x = x.view(B, T, P, -1)  # (B, T, P, dim)
        
        # Add positional embeddings
        x = x + self.spatial_pos.unsqueeze(1)  # Spatial pos to each frame
        x = x + self.temporal_pos.unsqueeze(2)  # Temporal pos to each position
        
        x = x.view(B, T * P, -1)  # (B, T*P, dim)
        
        # Transformer layers
        for layer in self.layers:
            x = layer(x)
        
        # Global average pooling → classification
        x = self.norm(x.mean(dim=1))  # (B, dim)
        return self.head(x)

# Usage
# model = SimpleVideoTransformer(num_classes=400, num_frames=8)
# video = torch.randn(2, 8, 3, 224, 224)
# output = model(video)  # (2, 400)
```

```python
# ============================================
# Using Pre-trained Video Transformers (Production)
# ============================================

# pip install pytorchvideo

from pytorchvideo.models import create_multiscale_vision_transformers
import torch

# Or use torchvision's MViT
from torchvision.models.video import mvit_v2_s, MViT_V2_S_Weights

def video_classification_pipeline(video_path):
    """Complete pipeline for video classification."""
    
    # Load model
    weights = MViT_V2_S_Weights.DEFAULT
    model = mvit_v2_s(weights=weights).eval()
    
    # Load and preprocess video
    frames = sample_frames(video_path, num_frames=16)
    
    # Convert to tensor and normalize
    clip = torch.from_numpy(frames).float().permute(3, 0, 1, 2) / 255.0
    
    # Resize to expected input
    clip = F.interpolate(
        clip.unsqueeze(0), size=(16, 224, 224),
        mode='trilinear', align_corners=False
    )
    
    # Apply model transforms
    transforms = weights.transforms()
    clip = transforms(clip.squeeze(0)).unsqueeze(0)
    
    # Inference
    with torch.no_grad():
        output = model(clip)
        probs = F.softmax(output, dim=1)
    
    # Get results
    categories = weights.meta["categories"]
    top5 = torch.topk(probs[0], 5)
    
    results = []
    for score, idx in zip(top5.values, top5.indices):
        results.append({
            'action': categories[idx],
            'confidence': score.item()
        })
    
    return results
```

---

## 8. Real-World Applications & Deployment

### Video Processing Pipeline (Production)

```python
# ============================================
# Production Video Analysis Pipeline
# ============================================

import cv2
import numpy as np
from collections import deque
import time

class VideoAnalysisPipeline:
    """
    Complete video analysis pipeline combining:
    - Object detection
    - Tracking
    - Action recognition
    """
    
    def __init__(self, detector, tracker, action_model, 
                 skip_frames=2, clip_length=16):
        self.detector = detector
        self.tracker = tracker
        self.action_model = action_model
        self.skip_frames = skip_frames
        self.clip_length = clip_length
        
        # Buffer for temporal analysis
        self.frame_buffer = deque(maxlen=clip_length)
        self.frame_count = 0
    
    def process_video(self, video_path, output_path=None):
        """Process entire video with detection + tracking + action."""
        cap = cv2.VideoCapture(video_path)
        fps = cap.get(cv2.CAP_PROP_FPS)
        
        # Setup writer if saving output
        if output_path:
            w = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
            h = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
            fourcc = cv2.VideoWriter_fourcc(*'mp4v')
            writer = cv2.VideoWriter(output_path, fourcc, fps, (w, h))
        
        results = []
        
        while cap.isOpened():
            ret, frame = cap.read()
            if not ret:
                break
            
            self.frame_count += 1
            self.frame_buffer.append(frame.copy())
            
            # Detection + Tracking (skip frames for speed)
            if self.frame_count % self.skip_frames == 0:
                detections = self.detector(frame)
                tracks = self.tracker.update(detections, frame)
            
            # Action recognition (when buffer is full)
            if len(self.frame_buffer) == self.clip_length:
                if self.frame_count % (self.clip_length // 2) == 0:
                    clip = np.array(list(self.frame_buffer))
                    action = self.action_model(clip)
                    results.append({
                        'frame': self.frame_count,
                        'time': self.frame_count / fps,
                        'action': action
                    })
            
            # Annotate frame
            annotated = self._draw_annotations(frame, tracks, results)
            
            if output_path:
                writer.write(annotated)
        
        cap.release()
        if output_path:
            writer.release()
        
        return results
    
    def _draw_annotations(self, frame, tracks, results):
        """Draw bounding boxes, IDs, and actions on frame."""
        for track in tracks:
            x1, y1, x2, y2 = track['bbox']
            tid = track['id']
            cv2.rectangle(frame, (x1, y1), (x2, y2), (0, 255, 0), 2)
            cv2.putText(frame, f"ID:{tid}", (x1, y1-10),
                       cv2.FONT_HERSHEY_SIMPLEX, 0.6, (0, 255, 0), 2)
        
        # Show latest action
        if results:
            action = results[-1]['action']
            cv2.putText(frame, f"Action: {action}", (10, 30),
                       cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)
        
        return frame
```

```python
# ============================================
# Efficient Video Processing Tips
# ============================================

# 1. Multi-threaded video reading (don't let I/O bottleneck GPU)
from threading import Thread
from queue import Queue

class VideoReader:
    """Threaded video reader for faster processing."""
    
    def __init__(self, video_path, queue_size=128):
        self.cap = cv2.VideoCapture(video_path)
        self.queue = Queue(maxsize=queue_size)
        self.stopped = False
    
    def start(self):
        Thread(target=self._read, daemon=True).start()
        return self
    
    def _read(self):
        while not self.stopped:
            if not self.queue.full():
                ret, frame = self.cap.read()
                if not ret:
                    self.stopped = True
                    break
                self.queue.put(frame)
    
    def read(self):
        return self.queue.get()
    
    def running(self):
        return not self.stopped or not self.queue.empty()
    
    def stop(self):
        self.stopped = True
        self.cap.release()

# 2. Batch processing for GPU efficiency
def batch_process_frames(frames, model, batch_size=8):
    """Process frames in batches for GPU efficiency."""
    results = []
    for i in range(0, len(frames), batch_size):
        batch = frames[i:i+batch_size]
        batch_tensor = torch.stack([preprocess(f) for f in batch])
        
        with torch.no_grad():
            outputs = model(batch_tensor.cuda())
        
        results.extend(outputs.cpu().numpy())
    return results

# 3. Resolution management
def adaptive_resize(frame, max_size=640):
    """Resize frame maintaining aspect ratio for speed."""
    h, w = frame.shape[:2]
    scale = max_size / max(h, w)
    if scale < 1:
        frame = cv2.resize(frame, None, fx=scale, fy=scale)
    return frame
```

---

## 9. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Processing every frame for detection | Wastes compute, objects don't move much between adjacent frames | Skip 2-5 frames, use tracking in between |
| Not handling variable-length videos | Models expect fixed temporal input | Use uniform sampling or sliding windows |
| Ignoring temporal augmentation | Overfits to specific speeds/durations | Add temporal jittering, speed augmentation |
| Using raw optical flow images | Flow values can be very large/small | Normalize flow, clip to reasonable range |
| Training video models from scratch | Need massive data (Kinetics-scale) | Always start with pretrained weights |
| Loading entire video into RAM | OOM for long videos | Use frame sampling or streaming |
| Ignoring frame rate differences | 30fps ≠ 60fps affects temporal models | Normalize to consistent FPS before processing |
| Not using GPU for video decoding | CPU decoding is the bottleneck | Use `decord`, NVIDIA DALI, or `torchvision.io` |
| Applying per-frame augmentation independently | Breaks temporal consistency | Apply SAME augmentation to all frames in clip |
| Tracking without re-detection | Tracks drift over time | Re-detect periodically to correct drift |

---

## 10. Interview Questions

### Conceptual

**Q1: What's the difference between optical flow and object tracking?**
> Optical flow estimates pixel-level motion between frames (dense or sparse). Tracking follows specific objects across frames using identity. Flow is low-level (pixel motion), tracking is high-level (object identity persistence).

**Q2: Why do 3D CNNs work for video but are expensive?**
> 3D CNNs apply 3D kernels (space + time) to capture spatiotemporal features. A 3×3×3 kernel has 27 parameters vs 9 for 2D. With T frames, computation scales linearly. Alternatives: (2+1)D factorization, TSM (zero-cost).

**Q3: Explain the Two-Stream architecture for action recognition.**
> Two parallel CNNs: (1) Spatial stream processes a single RGB frame for appearance/object recognition. (2) Temporal stream processes stacked optical flow maps for motion patterns. Their predictions are fused (average or learned fusion).

**Q4: How does DeepSORT improve over SORT?**
> SORT uses only Kalman filter + IoU matching. DeepSORT adds a deep appearance model (Re-ID network) that computes appearance embeddings. This handles occlusions better — when an object reappears, it can be re-identified by appearance even if IoU is 0.

**Q5: What is the Temporal Shift Module and why is it clever?**
> TSM shifts a portion of channels along the temporal axis. This allows information exchange between frames with ZERO extra parameters or computation. A 2D CNN with TSM achieves 3D CNN accuracy at 2D CNN cost.

**Q6: Why do Video Transformers use factorized attention?**
> Full space-time attention on a video with T=16, P=196 patches means 3136 tokens. Self-attention is O(n²), so 3136² ≈ 10M operations per layer. Factorizing into spatial (196²) + temporal (16²) = 38K + 256 ≈ 38K — orders of magnitude cheaper.

### Coding

**Q7: Write code to compute and visualize optical flow between two frames.**
> See Section 2 code examples (Farneback dense flow with HSV visualization).

**Q8: Implement a simple frame-sampling strategy for video classification.**
> See `sample_frames()` function in Section 1 — uniform sampling with `np.linspace`.

### System Design

**Q9: Design a real-time surveillance system that detects abnormal activities.**
> Components: (1) Camera stream ingestion with threading. (2) Person detection (YOLOv8) every N frames. (3) Multi-object tracking (ByteTrack). (4) Per-person action recognition with sliding window of crops. (5) Anomaly scoring (fighting, falling = high score). (6) Alert system with cooldown to avoid spam.

**Q10: How would you handle a 2-hour video for action recognition?**
> Don't process all frames! Use hierarchical approach: (1) Scene change detection to find segments. (2) Sample clips from each segment. (3) Classify each clip. (4) Aggregate predictions per segment. For temporal localization, use sliding window with stride.

---

## 11. Quick Reference

### Key Formulas

| Concept | Formula |
|---------|---------|
| Optical Flow Constraint | $I_x u + I_y v + I_t = 0$ |
| Video Tensor Shape | $(B, C, T, H, W)$ or $(B, T, C, H, W)$ |
| 3D Conv Output Size | $\lfloor\frac{T - k_t + 2p_t}{s_t}\rfloor + 1$ |
| Attention Complexity | $O((T \cdot P)^2 \cdot d)$ full, $O(T \cdot P^2 \cdot d + P \cdot T^2 \cdot d)$ factorized |

### Model Comparison

| Model | Year | Top-1 (K400) | Params | FLOPs | Key Idea |
|-------|------|-------------|--------|-------|----------|
| C3D | 2015 | 65.6% | 78M | 39G | First 3D CNN for video |
| I3D | 2017 | 72.1% | 25M | 108G | Inflate 2D → 3D |
| SlowFast | 2019 | 79.8% | 34M | 36G | Dual frame-rate paths |
| TSM | 2019 | 74.7% | 24M | 33G | Zero-cost temporal shift |
| TimeSformer | 2021 | 80.7% | 121M | 196G | Divided space-time attention |
| MViT-v2 | 2022 | 81.2% | 35M | 64G | Pooling attention |
| VideoMAE v2 | 2023 | 81.5% | 633M | - | Self-supervised pre-training |

### Libraries & Tools

| Library | Use Case |
|---------|----------|
| `torchvision.models.video` | Pretrained video models (R3D, MViT) |
| `pytorchvideo` | Facebook's video library (SlowFast, X3D) |
| `decord` | Fast GPU video decoding |
| `mmaction2` | Comprehensive action recognition toolkit |
| `deep-sort-realtime` | Multi-object tracking |
| `supervision` | Roboflow's tracking + annotation library |
| `norfair` | Lightweight multi-object tracking |

### Dataset Reference

| Dataset | Classes | Clips | Task |
|---------|---------|-------|------|
| Kinetics-400 | 400 | 300K | Action recognition |
| UCF-101 | 101 | 13K | Action recognition |
| HMDB-51 | 51 | 7K | Action recognition |
| MOT17/20 | - | - | Multi-object tracking |
| ActivityNet | 200 | 20K | Temporal localization |
| AVA | 80 | 430 videos | Spatial-temporal detection |

---

> **Key Takeaway:** Video understanding = Spatial (what's in the frame) + Temporal (how it changes). Start with pretrained models, use efficient architectures (TSM, SlowFast), and always profile your pipeline — I/O is often the bottleneck, not the model.
