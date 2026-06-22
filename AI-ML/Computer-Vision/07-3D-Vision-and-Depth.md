# Chapter 07: 3D Vision and Depth — Depth Estimation, Point Clouds, 3D Reconstruction & NeRF

---

## Table of Contents
1. [Introduction to 3D Vision](#1-introduction-to-3d-vision)
2. [Depth Estimation](#2-depth-estimation)
3. [Stereo Vision](#3-stereo-vision)
4. [Point Clouds](#4-point-clouds)
5. [3D Reconstruction](#5-3d-reconstruction)
6. [Neural Radiance Fields (NeRF)](#6-neural-radiance-fields-nerf)
7. [3D Gaussian Splatting](#7-3d-gaussian-splatting)
8. [Applications & Deployment](#8-applications--deployment)
9. [Common Mistakes](#9-common-mistakes)
10. [Interview Questions](#10-interview-questions)
11. [Quick Reference](#11-quick-reference)

---

## 1. Introduction to 3D Vision

### What It Is
3D vision is about understanding the three-dimensional structure of the world from 2D images. Our eyes do this naturally — from two flat images (left and right eye), our brain constructs a 3D understanding of the world. Computers can do the same thing, but it's much harder.

### Why It Matters
- **Autonomous Driving:** Must know exact 3D positions of cars, pedestrians, lanes
- **Robotics:** Manipulation requires 3D understanding (how far, what shape)
- **AR/VR:** Placing virtual objects in real 3D space
- **Architecture/Construction:** 3D scanning and modeling buildings
- **Medical Imaging:** 3D reconstruction from CT/MRI slices
- **Gaming/Film:** Creating 3D assets from photos

### The Fundamental Challenge

```
Real World (3D)              Camera (2D projection)
                            
    z                        ┌──────────────┐
    │  ●A (far)             │              │
    │     ●B (close)        │    ●A ●B     │  A and B appear at
    │                        │              │  SAME size if B is
    └────── x               │              │  small and close!
                            └──────────────┘
                            
Going from 3D → 2D is EASY (projection)
Going from 2D → 3D is HARD (information is lost!)
```

**Key 3D Representations:**

| Representation | Description | Pros | Cons |
|---------------|-------------|------|------|
| Depth Map | Per-pixel distance from camera | Dense, image-aligned | Single viewpoint |
| Point Cloud | Set of 3D points (x,y,z) | Simple, flexible | Sparse, no surface |
| Mesh | Vertices + faces (triangles) | Explicit surface, renderable | Complex topology |
| Voxel Grid | 3D grid of occupancy values | Regular structure | Memory-heavy (O(N³)) |
| Implicit (NeRF) | Neural network encoding 3D | Continuous, high quality | Slow rendering |
| Gaussian Splatting | 3D Gaussians | Fast rendering, high quality | Large storage |

---

## 2. Depth Estimation

### What It Is
Depth estimation predicts how far each pixel in an image is from the camera. The output is a **depth map** — a grayscale image where brighter means farther (or closer, depending on convention).

### Why It Matters
- Foundation for all 3D understanding from images
- Enables 3D photo effects, portrait mode (blur background)
- Critical for autonomous driving (how far is that car?)
- Enables AR object placement in correct 3D position

### How It Works

**Two Types:**

| Type | Input | Method | Accuracy |
|------|-------|--------|----------|
| Monocular | Single image | Deep learning (learned priors) | Relative (up-to-scale) |
| Stereo | Two images (left+right) | Geometry (triangulation) | Absolute (metric) |
| Multi-view | Multiple images | Structure from Motion | Absolute |
| Active (LiDAR/ToF) | Sensor data | Direct measurement | Highest |

**Monocular Depth — How Does a Network "Know" Depth from ONE Image?**

It learns **depth cues** that humans also use:
- **Relative size:** Cars far away appear smaller
- **Texture gradient:** Floor tiles get smaller with distance
- **Occlusion:** Objects in front block objects behind
- **Atmospheric haze:** Far objects are hazier
- **Height in image:** Ground plane objects higher = farther
- **Known object sizes:** A person is ~1.7m tall

```
Single Image                    Depth Map Output
┌──────────────────────┐       ┌──────────────────────┐
│  ☁️  ☁️    (sky=far)  │       │ ████████ (far=bright)│
│ 🏔️         (mtn=far) │       │ ███████             │
│                      │  →    │ █████               │
│  🚗      (car=mid)   │       │ ██   (mid)         │
│ ▓▓▓▓▓▓▓ (road=close)│       │ ░░░░░░░ (close=dark)│
└──────────────────────┘       └──────────────────────┘
```

### Code Examples

```python
# ============================================
# Monocular Depth Estimation with MiDaS
# (State-of-the-art relative depth)
# ============================================

import torch
import cv2
import numpy as np

# Load MiDaS model (Intel ISL)
model_type = "DPT_Large"  # Options: DPT_Large, DPT_Hybrid, MiDaS_small
midas = torch.hub.load("intel-isl/MiDaS", model_type)
midas.eval()

# Load transforms
midas_transforms = torch.hub.load("intel-isl/MiDaS", "transforms")
if model_type in ["DPT_Large", "DPT_Hybrid"]:
    transform = midas_transforms.dpt_transform
else:
    transform = midas_transforms.small_transform

def estimate_depth_midas(image_path):
    """
    Estimate relative depth from a single image.
    Note: MiDaS gives RELATIVE depth (inverse depth, actually).
    Higher values = CLOSER to camera.
    """
    # Read and transform image
    img = cv2.imread(image_path)
    img_rgb = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
    
    # Apply transforms
    input_batch = transform(img_rgb).unsqueeze(0)
    
    # Predict
    with torch.no_grad():
        prediction = midas(input_batch)
        # Resize to original image size
        prediction = torch.nn.functional.interpolate(
            prediction.unsqueeze(1),
            size=img.shape[:2],
            mode="bicubic",
            align_corners=False,
        ).squeeze()
    
    depth_map = prediction.cpu().numpy()
    
    # Normalize for visualization
    depth_normalized = cv2.normalize(depth_map, None, 0, 255, cv2.NORM_MINMAX)
    depth_colored = cv2.applyColorMap(depth_normalized.astype(np.uint8), cv2.COLORMAP_INFERNO)
    
    return depth_map, depth_colored

# Usage
# depth, depth_vis = estimate_depth_midas("street_scene.jpg")
# cv2.imwrite("depth_output.png", depth_vis)
```

```python
# ============================================
# Depth Anything v2 — Latest SOTA (2024)
# Metric depth from single images!
# ============================================

# pip install transformers

from transformers import pipeline
from PIL import Image
import numpy as np

# Load Depth Anything v2
depth_estimator = pipeline(
    task="depth-estimation",
    model="depth-anything/Depth-Anything-V2-Large-hf"
)

def estimate_depth_anything(image_path):
    """
    Estimate depth using Depth Anything v2.
    Returns relative depth (can be converted to metric with scaling).
    """
    image = Image.open(image_path)
    
    # Run inference
    result = depth_estimator(image)
    
    # result['depth'] is a PIL Image with depth values
    depth_map = np.array(result['depth'])
    
    return depth_map

# For METRIC depth (actual meters):
# Use Depth Anything v2 with metric fine-tuning
# or ZoeDepth which directly predicts metric depth

# depth = estimate_depth_anything("room.jpg")
# print(f"Depth range: {depth.min():.2f} to {depth.max():.2f}")
```

```python
# ============================================
# ZoeDepth — Metric Depth Estimation
# Actually predicts distance in METERS
# ============================================

import torch

# Load ZoeDepth
zoe = torch.hub.load("isl-org/ZoeDepth", "ZoeD_NK", pretrained=True)
zoe.eval()

def estimate_metric_depth(image_path):
    """
    Estimate metric (absolute) depth in meters.
    Works best for indoor scenes (NYU) and outdoor (KITTI).
    """
    from PIL import Image
    
    image = Image.open(image_path).convert("RGB")
    
    # Predict metric depth
    with torch.no_grad():
        depth = zoe.infer_pil(image)  # Returns numpy array in meters
    
    print(f"Closest point: {depth.min():.2f}m")
    print(f"Farthest point: {depth.max():.2f}m")
    
    return depth

# Example output for a room:
# Closest point: 0.45m (table in front)
# Farthest point: 6.23m (far wall)
```

```python
# ============================================
# Depth from Video (Consistent Depth)
# Challenge: Per-frame depth is temporally inconsistent
# ============================================

import torch
import numpy as np

class ConsistentVideoDepth:
    """
    Compute temporally consistent depth for video.
    Key insight: Use optical flow to enforce consistency.
    """
    
    def __init__(self, depth_model):
        self.depth_model = depth_model
        self.prev_depth = None
        self.alpha = 0.7  # Temporal smoothing factor
    
    def estimate_frame(self, frame, flow_from_prev=None):
        """
        Estimate depth with temporal consistency.
        
        Args:
            frame: Current frame
            flow_from_prev: Optical flow from previous to current frame
        """
        # Raw depth prediction
        raw_depth = self.depth_model(frame)
        
        if self.prev_depth is None or flow_from_prev is None:
            self.prev_depth = raw_depth
            return raw_depth
        
        # Warp previous depth to current frame using flow
        h, w = raw_depth.shape
        flow_map = flow_from_prev  # (H, W, 2)
        
        # Create mesh grid
        y, x = np.mgrid[0:h, 0:w].astype(np.float32)
        warped_x = x + flow_map[..., 0]
        warped_y = y + flow_map[..., 1]
        
        # Warp previous depth
        warped_depth = cv2.remap(
            self.prev_depth.astype(np.float32),
            warped_x, warped_y,
            cv2.INTER_LINEAR
        )
        
        # Blend: use warped previous depth where consistent
        # Use raw prediction where flow is unreliable
        valid_mask = (warped_depth > 0) & np.isfinite(warped_depth)
        
        consistent_depth = np.where(
            valid_mask,
            self.alpha * warped_depth + (1 - self.alpha) * raw_depth,
            raw_depth
        )
        
        self.prev_depth = consistent_depth
        return consistent_depth
```

---

## 3. Stereo Vision

### What It Is
Stereo vision uses two cameras (like our two eyes) to compute depth through **triangulation**. By knowing how far apart the cameras are and finding corresponding points in both images, you can calculate exact 3D positions.

### How It Works

```
Left Camera          Right Camera
    ●───── B (baseline) ─────●
    │\                      /│
    │ \                    / │
    │  \                  /  │
    │   \   3D Point P   /   │
    │    \      ●       /    │
    │     \    /|\     /     │
    f      \  / | \   /      f (focal length)
    │       \/  |  \ /       │
    │       /\  |  /\        │
    ├──────┤    d    ├───────┤
    xL          |         xR
                |
         Disparity = xL - xR
         
    Depth Z = (f × B) / disparity
    
    - Larger disparity → CLOSER object
    - Smaller disparity → FARTHER object
```

**Key Formula:**
$$Z = \frac{f \cdot B}{d}$$

Where:
- $Z$ = depth (distance from camera)
- $f$ = focal length (pixels)
- $B$ = baseline (distance between cameras)
- $d$ = disparity (xL - xR, in pixels)

### Code Example

```python
# ============================================
# Stereo Depth Estimation with OpenCV
# ============================================

import cv2
import numpy as np

def stereo_depth_sgbm(left_image, right_image, baseline=0.1, focal_length=700):
    """
    Compute depth map from stereo image pair using SGBM.
    
    Args:
        left_image: Left camera image
        right_image: Right camera image
        baseline: Distance between cameras in meters
        focal_length: Camera focal length in pixels
    
    Returns:
        depth_map: Depth in meters for each pixel
    """
    # Convert to grayscale
    left_gray = cv2.cvtColor(left_image, cv2.COLOR_BGR2GRAY)
    right_gray = cv2.cvtColor(right_image, cv2.COLOR_BGR2GRAY)
    
    # SGBM parameters (Semi-Global Block Matching)
    # These need tuning for your specific camera setup!
    min_disparity = 0
    num_disparities = 128  # Must be divisible by 16
    block_size = 5         # Odd number, 3-11
    
    stereo = cv2.StereoSGBM_create(
        minDisparity=min_disparity,
        numDisparities=num_disparities,
        blockSize=block_size,
        P1=8 * 3 * block_size ** 2,    # Smoothness penalty (small changes)
        P2=32 * 3 * block_size ** 2,   # Smoothness penalty (large changes)
        disp12MaxDiff=1,               # Left-right consistency check
        uniquenessRatio=10,            # Margin of winning match
        speckleWindowSize=100,         # Filter out noise
        speckleRange=32,
        preFilterCap=63,
        mode=cv2.STEREO_SGBM_MODE_SGBM_3WAY
    )
    
    # Compute disparity map
    disparity = stereo.compute(left_gray, right_gray).astype(np.float32) / 16.0
    
    # Convert disparity to depth (Z = f*B/d)
    # Avoid division by zero
    depth_map = np.zeros_like(disparity)
    valid = disparity > 0
    depth_map[valid] = (focal_length * baseline) / disparity[valid]
    
    # Cap maximum depth (remove noise at infinity)
    depth_map[depth_map > 100] = 0  # Max 100 meters
    
    return depth_map, disparity

# Visualization
def visualize_depth(depth_map, max_depth=50):
    """Create a colored depth visualization."""
    # Normalize to 0-255
    depth_vis = np.clip(depth_map / max_depth * 255, 0, 255).astype(np.uint8)
    depth_colored = cv2.applyColorMap(depth_vis, cv2.COLORMAP_JET)
    # Black out invalid pixels
    depth_colored[depth_map == 0] = 0
    return depth_colored

# Usage:
# left = cv2.imread('left.png')
# right = cv2.imread('right.png')
# depth, disparity = stereo_depth_sgbm(left, right, baseline=0.12, focal_length=718)
# cv2.imwrite('depth.png', visualize_depth(depth))
```

```python
# ============================================
# Deep Learning Stereo (RAFT-Stereo)
# Much better than traditional methods
# ============================================

# pip install torch torchvision

import torch
from torchvision.models.optical_flow import raft_large

# For production stereo, use:
# - RAFT-Stereo (Facebook)
# - CREStereo (Megvii)  
# - UniMatch (latest SOTA)

# Conceptual implementation of learning-based stereo:
class LearnedStereoMatcher(torch.nn.Module):
    """
    Conceptual deep stereo matching pipeline.
    
    Steps:
    1. Extract features from left and right images
    2. Build cost volume (all possible disparities)
    3. Regularize cost volume with 3D CNN
    4. Regression to get sub-pixel disparity
    """
    
    def __init__(self, max_disp=192):
        super().__init__()
        self.max_disp = max_disp
        
        # Feature extractor (shared weights for left/right)
        self.feature_net = torch.nn.Sequential(
            torch.nn.Conv2d(3, 32, 3, padding=1),
            torch.nn.ReLU(),
            torch.nn.Conv2d(32, 32, 3, padding=1),
            torch.nn.ReLU(),
        )
        
        # Cost volume regularization (3D CNN)
        self.cost_reg = torch.nn.Sequential(
            torch.nn.Conv3d(64, 32, 3, padding=1),
            torch.nn.ReLU(),
            torch.nn.Conv3d(32, 1, 3, padding=1),
        )
    
    def build_cost_volume(self, left_feat, right_feat):
        """
        Build 4D cost volume by shifting right features.
        Cost[d, h, w] = similarity(left[h,w], right[h, w-d])
        """
        B, C, H, W = left_feat.shape
        cost_volume = torch.zeros(B, 2*C, self.max_disp, H, W, device=left_feat.device)
        
        for d in range(self.max_disp):
            if d == 0:
                cost_volume[:, :C, d, :, :] = left_feat
                cost_volume[:, C:, d, :, :] = right_feat
            else:
                cost_volume[:, :C, d, :, d:] = left_feat[:, :, :, d:]
                cost_volume[:, C:, d, :, d:] = right_feat[:, :, :, :-d]
        
        return cost_volume
    
    def forward(self, left, right):
        # Extract features
        left_feat = self.feature_net(left)
        right_feat = self.feature_net(right)
        
        # Build and regularize cost volume
        cost = self.build_cost_volume(left_feat, right_feat)
        cost = self.cost_reg(cost).squeeze(1)  # (B, D, H, W)
        
        # Soft argmin for sub-pixel disparity
        prob = torch.softmax(-cost, dim=1)  # (B, D, H, W)
        disp_values = torch.arange(self.max_disp, device=cost.device).float()
        disp = torch.sum(prob * disp_values.view(1, -1, 1, 1), dim=1)
        
        return disp
```

---

## 4. Point Clouds

### What It Is
A point cloud is a collection of 3D points $(x, y, z)$ that represent the surface of objects in a scene. Think of it like a 3D version of a pointillist painting — millions of dots that together form the shape of objects.

### Why It Matters
- Direct 3D representation (no projection needed)
- Output of LiDAR sensors (autonomous driving)
- Output of 3D scanners (architecture, manufacturing)
- Input for robotics (grasp planning, navigation)
- Created from depth cameras (Kinect, RealSense)

### How It Works

```
Depth Map → Point Cloud Conversion:

For each pixel (u, v) with depth Z:

X = (u - cx) * Z / fx
Y = (v - cy) * Z / fy
Z = Z (depth value)

Where:
- (cx, cy) = camera principal point (center)
- (fx, fy) = focal lengths in pixels

       Image plane (u,v)          3D Point Cloud
       ┌──────────────┐          
       │  ● (u,v)     │              ● (X,Y,Z)
       │              │            ●     ●
       │              │          ●    ●    ●
       └──────────────┘        ●  ●  ●  ●  ●
                               (actual 3D positions)
```

### Code Examples

```python
# ============================================
# Point Cloud from Depth Map
# ============================================

import numpy as np
import cv2

def depth_to_pointcloud(depth_map, camera_intrinsics, color_image=None):
    """
    Convert a depth map to a 3D point cloud.
    
    Args:
        depth_map: (H, W) depth values in meters
        camera_intrinsics: dict with fx, fy, cx, cy
        color_image: Optional (H, W, 3) BGR image for colored points
    
    Returns:
        points: (N, 3) array of 3D points
        colors: (N, 3) array of RGB colors (if color_image provided)
    """
    fx = camera_intrinsics['fx']
    fy = camera_intrinsics['fy']
    cx = camera_intrinsics['cx']
    cy = camera_intrinsics['cy']
    
    H, W = depth_map.shape
    
    # Create pixel coordinate grids
    u = np.arange(W)
    v = np.arange(H)
    u, v = np.meshgrid(u, v)
    
    # Back-project to 3D
    Z = depth_map
    X = (u - cx) * Z / fx
    Y = (v - cy) * Z / fy
    
    # Filter out invalid depths
    valid = (Z > 0) & (Z < 100)  # Valid range: 0-100 meters
    
    points = np.stack([X[valid], Y[valid], Z[valid]], axis=-1)
    
    colors = None
    if color_image is not None:
        # Convert BGR to RGB and normalize to 0-1
        rgb = cv2.cvtColor(color_image, cv2.COLOR_BGR2RGB)
        colors = rgb[valid] / 255.0
    
    return points, colors

# Camera intrinsics (example for Intel RealSense D435)
intrinsics = {
    'fx': 615.0,  # Focal length x (pixels)
    'fy': 615.0,  # Focal length y (pixels)
    'cx': 320.0,  # Principal point x
    'cy': 240.0,  # Principal point y
}

# Usage:
# depth = np.load('depth.npy')  # or from depth camera
# color = cv2.imread('color.png')
# points, colors = depth_to_pointcloud(depth, intrinsics, color)
# print(f"Point cloud: {points.shape[0]} points")
```

```python
# ============================================
# Point Cloud Processing with Open3D
# ============================================

# pip install open3d

import open3d as o3d
import numpy as np

# Create point cloud from numpy arrays
def create_pointcloud(points, colors=None):
    """Create Open3D point cloud from numpy arrays."""
    pcd = o3d.geometry.PointCloud()
    pcd.points = o3d.utility.Vector3dVector(points)
    if colors is not None:
        pcd.colors = o3d.utility.Vector3dVector(colors)
    return pcd

# Load point cloud from file
# pcd = o3d.io.read_point_cloud("scene.ply")

# Common operations
def process_pointcloud(pcd):
    """Common point cloud processing operations."""
    
    # 1. Downsample (reduce number of points for speed)
    pcd_down = pcd.voxel_down_sample(voxel_size=0.02)  # 2cm voxels
    print(f"Downsampled: {len(pcd.points)} → {len(pcd_down.points)} points")
    
    # 2. Remove outliers (noise)
    # Statistical outlier removal
    pcd_clean, indices = pcd_down.remove_statistical_outlier(
        nb_neighbors=20,    # Number of neighbors to consider
        std_ratio=2.0       # Standard deviation threshold
    )
    print(f"After outlier removal: {len(pcd_clean.points)} points")
    
    # 3. Estimate normals (needed for surface reconstruction)
    pcd_clean.estimate_normals(
        search_param=o3d.geometry.KDTreeSearchParamHybrid(
            radius=0.1, max_nn=30
        )
    )
    
    # 4. Plane segmentation (find floor/walls)
    plane_model, inliers = pcd_clean.segment_plane(
        distance_threshold=0.02,  # Max distance from plane
        ransac_n=3,               # Points for plane estimation
        num_iterations=1000
    )
    a, b, c, d = plane_model
    print(f"Plane equation: {a:.2f}x + {b:.2f}y + {c:.2f}z + {d:.2f} = 0")
    
    # Separate plane (floor) from objects
    floor = pcd_clean.select_by_index(inliers)
    objects = pcd_clean.select_by_index(inliers, invert=True)
    
    # 5. Clustering (find individual objects)
    labels = np.array(objects.cluster_dbscan(
        eps=0.05,      # Maximum distance between points in cluster
        min_points=10  # Minimum points to form cluster
    ))
    num_clusters = labels.max() + 1
    print(f"Found {num_clusters} object clusters")
    
    return pcd_clean, floor, objects, labels

# Visualization
# o3d.visualization.draw_geometries([pcd_clean])
```

```python
# ============================================
# PointNet — Deep Learning on Point Clouds
# ============================================

import torch
import torch.nn as nn

class PointNet(nn.Module):
    """
    PointNet: Deep Learning on Point Sets.
    
    Key insight: Process each point independently with shared MLP,
    then aggregate with a symmetric function (max pooling).
    
    This makes it INVARIANT to point ordering!
    (Unlike images, point clouds have no canonical order)
    
    Architecture:
    Points (N,3) → MLP → MLP → MLP → MaxPool → Global Feature → Classifier
    """
    
    def __init__(self, num_classes=40, num_points=1024):
        super().__init__()
        
        # Per-point feature extraction (shared MLP)
        self.mlp1 = nn.Sequential(
            nn.Linear(3, 64),
            nn.BatchNorm1d(64),
            nn.ReLU(),
            nn.Linear(64, 128),
            nn.BatchNorm1d(128),
            nn.ReLU(),
            nn.Linear(128, 1024),
            nn.BatchNorm1d(1024),
            nn.ReLU(),
        )
        
        # Classification head
        self.classifier = nn.Sequential(
            nn.Linear(1024, 512),
            nn.BatchNorm1d(512),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(512, 256),
            nn.BatchNorm1d(256),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(256, num_classes),
        )
    
    def forward(self, x):
        """
        x: (B, N, 3) — batch of point clouds, N points each
        """
        B, N, _ = x.shape
        
        # Process each point (shared weights across all points)
        x = x.view(B * N, 3)
        x = self.mlp1(x)        # (B*N, 1024)
        x = x.view(B, N, 1024)  # (B, N, 1024)
        
        # Symmetric aggregation: MAX POOLING over points
        # This makes output INVARIANT to point order!
        x = x.max(dim=1)[0]     # (B, 1024) — global feature
        
        # Classify
        x = self.classifier(x)  # (B, num_classes)
        
        return x

# Usage:
# model = PointNet(num_classes=40)  # ModelNet40 dataset
# points = torch.randn(4, 1024, 3)  # 4 point clouds, 1024 points each
# output = model(points)  # (4, 40)
```

```python
# ============================================
# PointNet++ (Hierarchical PointNet)
# ============================================

class PointNetPPLayer(nn.Module):
    """
    PointNet++ processes points HIERARCHICALLY:
    1. Sample subset of points (farthest point sampling)
    2. Group nearby points around each sampled point
    3. Apply PointNet to each local group
    
    This captures LOCAL structure (PointNet doesn't!)
    
    Analogy: Like CNN pooling layers, but for irregular 3D points
    """
    
    def __init__(self, npoint, radius, nsample, in_channels, mlp_channels):
        super().__init__()
        self.npoint = npoint      # Number of points to sample
        self.radius = radius      # Ball query radius
        self.nsample = nsample    # Max points per local group
        
        # MLP for local feature extraction
        layers = []
        last_channel = in_channels + 3  # +3 for relative coordinates
        for out_channel in mlp_channels:
            layers.extend([
                nn.Conv1d(last_channel, out_channel, 1),
                nn.BatchNorm1d(out_channel),
                nn.ReLU(),
            ])
            last_channel = out_channel
        self.mlp = nn.Sequential(*layers)
    
    def forward(self, xyz, features):
        """
        xyz: (B, N, 3) — point coordinates
        features: (B, N, C) — point features
        
        Returns:
        new_xyz: (B, npoint, 3) — sampled point coordinates
        new_features: (B, npoint, C') — aggregated features
        """
        B, N, _ = xyz.shape
        
        # 1. Farthest Point Sampling: select npoint representative points
        # (ensures good coverage of the point cloud)
        fps_idx = self.farthest_point_sample(xyz, self.npoint)
        new_xyz = self.gather(xyz, fps_idx)  # (B, npoint, 3)
        
        # 2. Ball Query: find neighbors within radius
        # (groups nearby points around each sampled point)
        group_idx = self.ball_query(xyz, new_xyz, self.radius, self.nsample)
        
        # 3. Get local point groups
        grouped_xyz = self.gather(xyz, group_idx) - new_xyz.unsqueeze(2)
        if features is not None:
            grouped_features = self.gather(features, group_idx)
            grouped_features = torch.cat([grouped_xyz, grouped_features], dim=-1)
        else:
            grouped_features = grouped_xyz
        
        # 4. Apply MLP + Max Pool (PointNet on each local group)
        # (B, npoint, nsample, C) → (B, C, npoint*nsample) → MLP → reshape → max
        grouped_features = grouped_features.view(B * self.npoint, self.nsample, -1)
        grouped_features = grouped_features.permute(0, 2, 1)  # (B*npoint, C, nsample)
        grouped_features = self.mlp(grouped_features)  # Apply shared MLP
        new_features = grouped_features.max(dim=-1)[0]  # Max pool
        new_features = new_features.view(B, self.npoint, -1)
        
        return new_xyz, new_features
    
    def farthest_point_sample(self, xyz, npoint):
        """Sample points that are maximally spread out."""
        # Simplified — real impl uses iterative farthest point
        B, N, _ = xyz.shape
        idx = torch.randint(0, N, (B, npoint), device=xyz.device)
        return idx
    
    def ball_query(self, xyz, new_xyz, radius, nsample):
        """Find all points within radius of each query point."""
        # Simplified
        B, N, _ = xyz.shape
        _, S, _ = new_xyz.shape
        idx = torch.randint(0, N, (B, S, nsample), device=xyz.device)
        return idx
    
    def gather(self, points, idx):
        """Gather points by index."""
        B = points.shape[0]
        idx_expanded = idx.unsqueeze(-1).expand(-1, -1, points.shape[-1]) if idx.dim() == 2 else \
                       idx.unsqueeze(-1).expand(-1, -1, -1, points.shape[-1])
        return torch.gather(points.unsqueeze(1).expand(-1, idx.shape[1], -1, -1) if idx.dim() == 3 else points,
                           1, idx_expanded)
```

---

## 5. 3D Reconstruction

### What It Is
3D reconstruction creates a complete 3D model of a scene or object from multiple 2D images (or other sensor data). It's like building a virtual 3D copy of the real world.

### Why It Matters
- Digital twins of buildings, cities, objects
- Heritage preservation (digitize monuments)
- E-commerce (3D product models from photos)
- Gaming/VFX (scan real environments)
- Construction monitoring

### How It Works — Structure from Motion (SfM)

```
Multiple Views of Same Scene:

Image 1        Image 2        Image 3        Image 4
┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐
│  🏠  │      │  🏠  │      │  🏠  │      │  🏠  │
│ /  \ │      │ |  | │      │ \  / │      │  || │
│/    \│      │ |  | │      │  \/ │      │  ||  │
└──────┘      └──────┘      └──────┘      └──────┘
(front-left)  (front)       (front-right)  (side)

    ↓ Feature matching + Camera pose estimation ↓

    ┌─────────────────────────────────────────────┐
    │           3D Point Cloud / Mesh             │
    │                                             │
    │              /\                              │
    │             /  \                             │
    │            /    \                            │
    │           /______\    ← 3D House            │
    │           |      |                          │
    │           |  🚪  |                          │
    │           |______|                          │
    └─────────────────────────────────────────────┘
```

**Pipeline:**
1. **Feature Detection:** Find keypoints in all images (SIFT, SuperPoint)
2. **Feature Matching:** Match keypoints between image pairs
3. **Camera Pose Estimation:** Where was the camera for each photo?
4. **Triangulation:** Compute 3D point from matched 2D points
5. **Bundle Adjustment:** Jointly optimize all cameras + points
6. **Dense Reconstruction:** Fill in gaps (MVS — Multi-View Stereo)
7. **Surface Reconstruction:** Convert point cloud → mesh

### Code Example

```python
# ============================================
# 3D Reconstruction Pipeline with COLMAP
# (Industry standard for SfM)
# ============================================

import subprocess
import os

def run_colmap_reconstruction(image_dir, output_dir):
    """
    Run COLMAP SfM + MVS pipeline.
    
    This is the standard pipeline used in research and industry.
    Also used as preprocessing for NeRF!
    """
    os.makedirs(output_dir, exist_ok=True)
    database_path = os.path.join(output_dir, "database.db")
    sparse_dir = os.path.join(output_dir, "sparse")
    dense_dir = os.path.join(output_dir, "dense")
    
    # Step 1: Feature extraction
    subprocess.run([
        "colmap", "feature_extractor",
        "--database_path", database_path,
        "--image_path", image_dir,
        "--ImageReader.single_camera", "1",
        "--SiftExtraction.use_gpu", "1",
    ])
    
    # Step 2: Feature matching
    subprocess.run([
        "colmap", "exhaustive_matcher",
        "--database_path", database_path,
        "--SiftMatching.use_gpu", "1",
    ])
    
    # Step 3: Sparse reconstruction (SfM)
    os.makedirs(sparse_dir, exist_ok=True)
    subprocess.run([
        "colmap", "mapper",
        "--database_path", database_path,
        "--image_path", image_dir,
        "--output_path", sparse_dir,
    ])
    
    # Step 4: Dense reconstruction (MVS)
    os.makedirs(dense_dir, exist_ok=True)
    subprocess.run([
        "colmap", "image_undistorter",
        "--image_path", image_dir,
        "--input_path", os.path.join(sparse_dir, "0"),
        "--output_path", dense_dir,
    ])
    
    subprocess.run([
        "colmap", "patch_match_stereo",
        "--workspace_path", dense_dir,
    ])
    
    subprocess.run([
        "colmap", "stereo_fusion",
        "--workspace_path", dense_dir,
        "--output_path", os.path.join(dense_dir, "fused.ply"),
    ])
    
    print(f"Reconstruction complete! Output: {dense_dir}/fused.ply")

# Usage:
# run_colmap_reconstruction("./my_photos/", "./reconstruction/")
```

```python
# ============================================
# Mesh from Point Cloud (Poisson Reconstruction)
# ============================================

import open3d as o3d
import numpy as np

def pointcloud_to_mesh(pcd_path, output_path="mesh.ply"):
    """
    Convert point cloud to mesh using Poisson surface reconstruction.
    """
    # Load point cloud
    pcd = o3d.io.read_point_cloud(pcd_path)
    print(f"Point cloud: {len(pcd.points)} points")
    
    # Estimate normals (required for Poisson)
    pcd.estimate_normals(
        search_param=o3d.geometry.KDTreeSearchParamHybrid(radius=0.1, max_nn=30)
    )
    # Orient normals consistently (all pointing outward)
    pcd.orient_normals_consistent_tangent_plane(k=15)
    
    # Poisson surface reconstruction
    mesh, densities = o3d.geometry.TriangleMesh.create_from_point_cloud_poisson(
        pcd, depth=9  # Higher depth = more detail but slower
    )
    
    # Remove low-density vertices (clean up edges)
    densities = np.asarray(densities)
    density_threshold = np.quantile(densities, 0.01)
    vertices_to_remove = densities < density_threshold
    mesh.remove_vertices_by_mask(vertices_to_remove)
    
    # Save
    o3d.io.write_triangle_mesh(output_path, mesh)
    print(f"Mesh saved: {len(mesh.vertices)} vertices, {len(mesh.triangles)} faces")
    
    return mesh
```

---

## 6. Neural Radiance Fields (NeRF)

### What It Is
NeRF (Neural Radiance Field) represents a 3D scene as a neural network. Given a 3D position and viewing direction, it outputs the color and density at that point. By "querying" the network along camera rays, you can render photorealistic images from ANY viewpoint.

### Why It Matters
- Photorealistic novel view synthesis (see a scene from any angle)
- Revolutionary quality leap in 3D reconstruction
- Enables VR/AR scene capture from just phone photos
- Foundation for 3D generation (DreamFusion, Magic3D)
- Industry adoption: Google, NVIDIA, Apple using NeRF-based tech

### How It Works

```
Given: Multiple photos of a scene + camera poses

Step 1: For a new viewpoint, cast rays through each pixel

             Camera (new viewpoint)
                ●
               /|\
              / | \
             /  |  \
            ●   ●   ●  ← Rays through each pixel
           /    |    \
          /     |     \
    ─────/──────|──────\────── Scene

Step 2: Sample points along each ray

    Ray: ●──●──●──●──●──●──●──●→
              ↑   ↑   ↑   ↑
         Query the neural network at each point!

Step 3: Neural Network predicts color + density

    Input: (x, y, z, θ, φ) → MLP → Output: (r, g, b, σ)
    position  direction            color   density

Step 4: Volume Rendering — accumulate color along ray

    C(ray) = Σ Tᵢ · αᵢ · cᵢ

    Where:
    - Tᵢ = transmittance (how much light passes through)
    - αᵢ = 1 - exp(-σᵢ · δᵢ) (opacity at sample i)
    - cᵢ = color at sample i
    - δᵢ = distance between samples
```

**The Math (Volume Rendering):**

$$C(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \cdot \sigma(\mathbf{r}(t)) \cdot \mathbf{c}(\mathbf{r}(t), \mathbf{d}) \, dt$$

Where:
$$T(t) = \exp\left(-\int_{t_n}^{t} \sigma(\mathbf{r}(s)) \, ds\right)$$

Discrete approximation:
$$\hat{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i (1 - e^{-\sigma_i \delta_i}) \mathbf{c}_i$$
$$T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$$

### Code Example

```python
# ============================================
# Simplified NeRF Implementation
# ============================================

import torch
import torch.nn as nn
import numpy as np

class NeRF(nn.Module):
    """
    Neural Radiance Field network.
    
    Input: 3D position (x,y,z) + 2D viewing direction (θ,φ)
    Output: RGB color (r,g,b) + volume density (σ)
    
    Key design choices:
    1. Positional encoding (to capture high-frequency details)
    2. Skip connection at layer 5 (helps optimization)
    3. Direction only affects COLOR (not density)
       (an object's shape doesn't change with viewing angle,
        but its appearance does — think of specular highlights)
    """
    
    def __init__(self, pos_dim=63, dir_dim=27, hidden_dim=256):
        super().__init__()
        
        # Position encoding dimensions:
        # xyz (3) → positional encoding → 3 + 3*2*10 = 63
        # dir (3) → positional encoding → 3 + 3*2*4  = 27
        
        # Network for position → density + feature
        self.pos_net = nn.Sequential(
            nn.Linear(pos_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
        )
        
        # Skip connection (concatenate input position again)
        self.pos_net2 = nn.Sequential(
            nn.Linear(hidden_dim + pos_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim), nn.ReLU(),
        )
        
        # Density output (independent of view direction)
        self.density_head = nn.Linear(hidden_dim, 1)
        
        # Feature for color
        self.feature_head = nn.Linear(hidden_dim, hidden_dim)
        
        # Color depends on viewing direction (for specular effects)
        self.color_net = nn.Sequential(
            nn.Linear(hidden_dim + dir_dim, hidden_dim // 2), nn.ReLU(),
            nn.Linear(hidden_dim // 2, 3), nn.Sigmoid(),  # RGB in [0,1]
        )
    
    def forward(self, positions, directions):
        """
        positions: (N, pos_dim) — positionally encoded 3D positions
        directions: (N, dir_dim) — positionally encoded view directions
        
        Returns:
            colors: (N, 3) — RGB colors
            density: (N, 1) — volume density (σ)
        """
        # Process position
        h = self.pos_net(positions)
        
        # Skip connection
        h = self.pos_net2(torch.cat([h, positions], dim=-1))
        
        # Density (view-independent)
        density = torch.relu(self.density_head(h))  # σ ≥ 0
        
        # Color (view-dependent)
        feature = self.feature_head(h)
        color = self.color_net(torch.cat([feature, directions], dim=-1))
        
        return color, density


def positional_encoding(x, num_frequencies=10):
    """
    Positional encoding: map low-dim input to high-dim.
    
    Why? Neural networks are biased toward learning LOW-frequency
    functions. But scenes have high-frequency details (sharp edges,
    fine textures). Positional encoding fixes this!
    
    γ(x) = [x, sin(2⁰πx), cos(2⁰πx), sin(2¹πx), cos(2¹πx), ..., 
             sin(2^(L-1)πx), cos(2^(L-1)πx)]
    """
    encoded = [x]
    for i in range(num_frequencies):
        freq = 2.0 ** i * np.pi
        encoded.append(torch.sin(freq * x))
        encoded.append(torch.cos(freq * x))
    return torch.cat(encoded, dim=-1)


def render_rays(model, ray_origins, ray_directions, near=2.0, far=6.0, 
                num_samples=64):
    """
    Render color for a batch of rays using volume rendering.
    
    Args:
        model: NeRF model
        ray_origins: (N_rays, 3) — where rays start
        ray_directions: (N_rays, 3) — direction of each ray
        near/far: sampling bounds along ray
        num_samples: number of points to sample per ray
    """
    N_rays = ray_origins.shape[0]
    device = ray_origins.device
    
    # 1. Sample points along each ray
    t_vals = torch.linspace(near, far, num_samples, device=device)
    # Add noise for continuous sampling (anti-aliasing)
    noise = torch.rand(N_rays, num_samples, device=device) * (far - near) / num_samples
    t_vals = t_vals.unsqueeze(0) + noise
    
    # 3D positions along rays: origin + t * direction
    points = ray_origins[:, None, :] + t_vals[:, :, None] * ray_directions[:, None, :]
    # points shape: (N_rays, num_samples, 3)
    
    # 2. Encode positions and directions
    points_flat = points.reshape(-1, 3)
    dirs_flat = ray_directions[:, None, :].expand_as(points).reshape(-1, 3)
    
    pos_encoded = positional_encoding(points_flat, num_frequencies=10)
    dir_encoded = positional_encoding(dirs_flat, num_frequencies=4)
    
    # 3. Query NeRF network
    colors, densities = model(pos_encoded, dir_encoded)
    colors = colors.reshape(N_rays, num_samples, 3)
    densities = densities.reshape(N_rays, num_samples)
    
    # 4. Volume rendering
    # Compute distances between samples
    deltas = t_vals[:, 1:] - t_vals[:, :-1]
    deltas = torch.cat([deltas, torch.tensor([1e10], device=device).expand(N_rays, 1)], dim=-1)
    
    # Alpha compositing
    alpha = 1.0 - torch.exp(-densities * deltas)  # Opacity per sample
    
    # Transmittance (how much light passes through previous samples)
    transmittance = torch.cumprod(1.0 - alpha + 1e-10, dim=-1)
    transmittance = torch.cat([
        torch.ones(N_rays, 1, device=device), 
        transmittance[:, :-1]
    ], dim=-1)
    
    # Final color: weighted sum
    weights = transmittance * alpha  # (N_rays, num_samples)
    rgb = torch.sum(weights[:, :, None] * colors, dim=1)  # (N_rays, 3)
    
    # Depth (expected distance)
    depth = torch.sum(weights * t_vals, dim=1)  # (N_rays,)
    
    return rgb, depth, weights
```

```python
# ============================================
# Using Nerfstudio (Production NeRF Framework)
# ============================================

# pip install nerfstudio

# Process your data:
# ns-process-data images --data ./my_photos/ --output-dir ./processed/

# Train NeRF:
# ns-train nerfacto --data ./processed/

# Render video from trained model:
# ns-render camera-path --load-config outputs/my_scene/nerfacto/config.yml \
#     --camera-path-filename camera_path.json --output-path renders/

# Popular NeRF variants available in nerfstudio:
# - nerfacto: Good default, balanced speed/quality
# - instant-ngp: Very fast training (minutes instead of hours)
# - mipnerf: Anti-aliased, handles multi-scale
# - tensorf: Tensor decomposition (fast + compact)
```

---

## 7. 3D Gaussian Splatting

### What It Is
3D Gaussian Splatting (3DGS, 2023) is a revolutionary alternative to NeRF. Instead of encoding a scene in a neural network, it represents the scene as millions of tiny 3D Gaussian blobs. These can be rendered extremely fast (100+ FPS) while matching NeRF quality.

### Why It Matters
- **Real-time rendering** (100-200 FPS vs NeRF's seconds per frame)
- Same or better quality than NeRF
- Editable (move/delete individual Gaussians)
- Much faster training (minutes vs hours)
- Enabling real-time 3D streaming, VR/AR applications

### How It Works

```
Scene = Collection of 3D Gaussians

Each Gaussian has:
- Position (μ): Where is it? (x, y, z)
- Covariance (Σ): What shape/size? (3D ellipsoid)
- Color (c): What color? (RGB, or spherical harmonics for view-dependent)
- Opacity (α): How transparent? (0-1)

Rendering (Splatting):
1. Project each 3D Gaussian onto the 2D image plane
2. Sort by depth (front to back)
3. Alpha-blend overlapping Gaussians

    3D Scene                     2D Image
    ○ ○ ○ ○                    ┌────────────┐
    ○ ● ○ ○   Project &       │  ○         │
    ○ ○ ○ ○   Splat     →     │    ●       │
    ○ ○ ○ ●                    │      ○  ●  │
                               └────────────┘
    
    Each ○/● is a Gaussian "splat"
```

**Key Difference from NeRF:**

| Aspect | NeRF | 3D Gaussian Splatting |
|--------|------|---------------------|
| Representation | Neural network (implicit) | Explicit 3D Gaussians |
| Rendering | Ray marching (slow) | Rasterization (fast!) |
| Training | Hours | Minutes |
| Inference | Seconds/frame | Milliseconds/frame |
| Editability | Difficult | Easy (move Gaussians) |
| Memory | Small (network weights) | Large (millions of Gaussians) |

### Code Example (Conceptual)

```python
# ============================================
# 3D Gaussian Splatting — Conceptual Implementation
# ============================================

import torch
import torch.nn as nn
import numpy as np

class GaussianModel:
    """
    Represents a 3D scene as a collection of Gaussians.
    Each Gaussian: position, covariance, color, opacity.
    """
    
    def __init__(self, num_points=100000):
        # Initialize from SfM point cloud (COLMAP output)
        self.positions = torch.randn(num_points, 3, requires_grad=True)     # μ
        self.scales = torch.zeros(num_points, 3, requires_grad=True)         # log(scale)
        self.rotations = torch.zeros(num_points, 4, requires_grad=True)      # quaternion
        self.colors_sh = torch.randn(num_points, 48, requires_grad=True)     # Spherical harmonics
        self.opacities = torch.zeros(num_points, 1, requires_grad=True)      # logit(α)
        
        # Initialize rotations to identity quaternion
        self.rotations.data[:, 0] = 1.0
    
    @property
    def get_opacity(self):
        return torch.sigmoid(self.opacities)
    
    @property
    def get_scaling(self):
        return torch.exp(self.scales)
    
    @property
    def get_covariance(self):
        """Compute 3D covariance from scale and rotation."""
        S = torch.diag_embed(self.get_scaling)  # Scale matrix
        R = self.quaternion_to_rotation(self.rotations)  # Rotation matrix
        # Covariance = R * S * S^T * R^T
        return R @ S @ S.transpose(-1, -2) @ R.transpose(-1, -2)
    
    def quaternion_to_rotation(self, q):
        """Convert quaternion to rotation matrix."""
        # Normalize
        q = q / q.norm(dim=-1, keepdim=True)
        w, x, y, z = q[:, 0], q[:, 1], q[:, 2], q[:, 3]
        
        R = torch.stack([
            1 - 2*y*y - 2*z*z, 2*x*y - 2*w*z, 2*x*z + 2*w*y,
            2*x*y + 2*w*z, 1 - 2*x*x - 2*z*z, 2*y*z - 2*w*x,
            2*x*z - 2*w*y, 2*y*z + 2*w*x, 1 - 2*x*x - 2*y*y
        ], dim=-1).reshape(-1, 3, 3)
        
        return R
    
    def densification(self, grad_threshold=0.0002):
        """
        Adaptive density control (key to quality!):
        - Clone Gaussians with large positional gradients (under-reconstruction)
        - Split large Gaussians with large gradients (over-reconstruction)
        - Prune transparent Gaussians (α < threshold)
        """
        grads = self.positions.grad.norm(dim=-1)
        
        # Clone: small Gaussians in high-gradient areas
        clone_mask = (grads > grad_threshold) & (self.get_scaling.max(dim=-1)[0] < 0.01)
        
        # Split: large Gaussians in high-gradient areas
        split_mask = (grads > grad_threshold) & (self.get_scaling.max(dim=-1)[0] >= 0.01)
        
        # Prune: nearly invisible Gaussians
        prune_mask = self.get_opacity.squeeze() < 0.005
        
        return clone_mask, split_mask, prune_mask


def render_gaussians(gaussians, camera, image_size):
    """
    Render Gaussians to an image (simplified).
    
    Real implementation uses custom CUDA kernels for:
    1. Frustum culling (skip off-screen Gaussians)
    2. Tile-based rasterization (process in 16×16 tiles)
    3. Front-to-back alpha blending
    """
    H, W = image_size
    
    # 1. Project 3D Gaussians to 2D
    # Transform to camera space
    positions_cam = camera.world_to_cam(gaussians.positions)
    
    # Project to 2D (perspective projection)
    positions_2d = camera.project(positions_cam)  # (N, 2) pixel coords
    
    # Compute 2D covariance (project 3D covariance)
    # J = Jacobian of projection, Σ_2d = J * Σ_3d * J^T
    
    # 2. Sort by depth (front to back)
    depths = positions_cam[:, 2]
    sorted_idx = torch.argsort(depths)
    
    # 3. Tile-based rasterization
    # For each 16×16 tile, find overlapping Gaussians
    # Alpha-blend in depth order
    
    # 4. Alpha blending (simplified)
    image = torch.zeros(H, W, 3)
    alpha_accum = torch.zeros(H, W)
    
    # In practice, this is done with CUDA kernels for speed
    # The key equation at each pixel:
    # C = Σ cᵢ · αᵢ · Π(1 - αⱼ) for j < i
    
    return image

# Training loop (simplified):
# 1. Render image from random training view
# 2. Compare with ground truth (L1 + D-SSIM loss)
# 3. Backpropagate gradients to Gaussian parameters
# 4. Every N iterations: densify (clone/split) and prune
```

```python
# ============================================
# Using Gaussian Splatting in Practice
# ============================================

# Option 1: Original implementation
# git clone https://github.com/graphdeco-inria/gaussian-splatting
# pip install -r requirements.txt
# python train.py -s <path_to_colmap_data>
# python render.py -m <path_to_model>

# Option 2: Nerfstudio's implementation (easier)
# pip install nerfstudio gsplat
# ns-train splatfacto --data ./processed_data/

# Option 3: gsplat (standalone, minimal)
# pip install gsplat

# Key hyperparameters:
# - Initial points: Start from COLMAP SfM points
# - Densification interval: Every 100 iterations
# - Opacity reset: Every 3000 iterations  
# - Total iterations: 30,000 (usually converges in 15-20 min)
```

---

## 8. Applications & Deployment

### Depth Estimation in Production

```python
# ============================================
# Real-time Depth for Mobile/Edge Deployment
# ============================================

import onnxruntime as ort
import numpy as np
import cv2

class DepthEstimatorONNX:
    """Optimized depth estimation for deployment."""
    
    def __init__(self, model_path="depth_anything_v2_small.onnx"):
        # Use ONNX Runtime for fast inference
        self.session = ort.InferenceSession(
            model_path,
            providers=['CUDAExecutionProvider', 'CPUExecutionProvider']
        )
        self.input_name = self.session.get_inputs()[0].name
        self.input_size = (518, 518)  # Model's expected input
    
    def preprocess(self, image):
        """Preprocess image for the model."""
        # Resize
        img = cv2.resize(image, self.input_size)
        # Normalize
        img = img.astype(np.float32) / 255.0
        img = (img - [0.485, 0.456, 0.406]) / [0.229, 0.224, 0.225]
        # HWC → CHW → NCHW
        img = img.transpose(2, 0, 1)[np.newaxis, ...]
        return img.astype(np.float32)
    
    def predict(self, image):
        """Run inference and return depth map."""
        input_tensor = self.preprocess(image)
        output = self.session.run(None, {self.input_name: input_tensor})[0]
        
        # Resize depth to original image size
        depth = output[0, 0]  # Remove batch and channel dims
        depth = cv2.resize(depth, (image.shape[1], image.shape[0]))
        
        return depth
    
    def get_3d_point(self, depth_map, pixel_x, pixel_y, intrinsics):
        """Convert a pixel + depth to 3D world coordinates."""
        Z = depth_map[pixel_y, pixel_x]
        X = (pixel_x - intrinsics['cx']) * Z / intrinsics['fx']
        Y = (pixel_y - intrinsics['cy']) * Z / intrinsics['fy']
        return np.array([X, Y, Z])
```

---

## 9. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Using monocular depth as absolute distance | Monocular gives RELATIVE depth only | Use stereo/LiDAR for metric, or metric-specific models (ZoeDepth) |
| Not calibrating stereo cameras | Incorrect baseline/focal length → wrong depth | Always calibrate with checkerboard pattern |
| Training NeRF with too few images | Underconstrained → artifacts, floaters | Use 50-200+ images with good coverage |
| Ignoring camera pose quality for NeRF | Bad poses → blurry, inconsistent results | Use COLMAP carefully, verify poses |
| Applying PointNet to large-scale scenes | PointNet processes ALL points globally | Use PointNet++ (hierarchical) or voxel-based methods |
| Not normalizing point clouds | Different scales confuse the network | Center and scale to unit sphere |
| Expecting NeRF to work for dynamic scenes | Standard NeRF is for static scenes only | Use D-NeRF, Nerfies, or 4D Gaussians |
| Ignoring near/far bounds in NeRF | Sampling outside scene → wasted capacity | Set tight bounds based on your scene |
| Using raw depth for point cloud without filtering | Sensor noise creates outliers | Apply statistical/radius outlier removal |
| Not considering occlusions in multi-view | Missing data in occluded regions | Use more views, or fill with learned priors |

---

## 10. Interview Questions

### Conceptual

**Q1: Explain how monocular depth estimation works despite being an ill-posed problem.**
> It's "ill-posed" because infinite 3D scenes can produce the same 2D image. Deep networks learn statistical priors from training data — they've seen millions of images and learn that floors are below, skies are far, objects have typical sizes, texture gradients indicate distance, etc. The output is relative depth (ordering) which is usually sufficient.

**Q2: What is the epipolar constraint in stereo vision?**
> Given a point in the left image, its corresponding point in the right image must lie on a specific line (the epipolar line). This reduces the matching search from 2D to 1D, making stereo matching tractable. Mathematically: $x_R^T F x_L = 0$ where F is the fundamental matrix.

**Q3: Why does NeRF use positional encoding?**
> Neural networks with standard activations (ReLU, sigmoid) are biased toward learning smooth, low-frequency functions. Scenes have high-frequency details (sharp edges, fine textures). Positional encoding maps coordinates to higher dimensions using sinusoids: $\gamma(x) = [\sin(2^k \pi x), \cos(2^k \pi x)]$, enabling the network to represent high frequencies.

**Q4: Compare NeRF vs 3D Gaussian Splatting.**
> NeRF: Implicit (MLP), ray marching for rendering (slow), compact storage, hard to edit. 3DGS: Explicit (Gaussian primitives), rasterization (100× faster rendering), larger storage, easily editable. 3DGS achieves similar/better quality at real-time framerates. NeRF is better when storage is limited or for continuous representation.

**Q5: How does PointNet achieve permutation invariance?**
> Point clouds have no canonical ordering — shuffling points shouldn't change the prediction. PointNet processes each point independently through shared MLPs, then aggregates with max pooling (a symmetric function). Max pooling gives the same output regardless of input order: max{a,b,c} = max{c,a,b}.

**Q6: What are the main challenges in 3D reconstruction from images?**
> (1) Textureless regions — no features to match. (2) Reflective/transparent surfaces — violate brightness constancy. (3) Repetitive patterns — ambiguous matches. (4) Occlusions — some surfaces not visible. (5) Scale ambiguity — monocular can't determine absolute scale. (6) Computational cost for large scenes.

### System Design

**Q7: Design a 3D scanning system for e-commerce products.**
> (1) Capture: Turntable with camera, or user walks around object with phone. (2) COLMAP for camera poses and sparse reconstruction. (3) NeRF/3DGS for high-quality reconstruction. (4) Extract mesh for rendering. (5) Texture baking for lightweight deployment. (6) Serve as GLTF/USDZ for web/AR. Key: controlled lighting, sufficient coverage (100+ angles).

---

## 11. Quick Reference

### Depth Estimation Models

| Model | Type | Speed | Quality | Metric? |
|-------|------|-------|---------|---------|
| MiDaS v3.1 | Monocular | Fast | Good | No (relative) |
| Depth Anything v2 | Monocular | Fast | Excellent | Optional |
| ZoeDepth | Monocular | Medium | Great | Yes |
| SGBM (OpenCV) | Stereo | Fast | Medium | Yes |
| RAFT-Stereo | Stereo | Slow | Excellent | Yes |
| UniMatch | Stereo | Medium | SOTA | Yes |

### 3D Reconstruction Methods

| Method | Input | Output | Speed | Quality |
|--------|-------|--------|-------|---------|
| COLMAP (SfM+MVS) | Images | Point Cloud/Mesh | Hours | Good |
| NeRF | Images + Poses | Neural Field | Hours | Excellent |
| Instant-NGP | Images + Poses | Hash Grid | Minutes | Great |
| 3D Gaussian Splatting | Images + Poses | Gaussians | Minutes | Excellent |
| DUSt3R | Image pairs | Point Cloud | Seconds | Good |

### Point Cloud Networks

| Model | Year | Task | Key Idea |
|-------|------|------|----------|
| PointNet | 2017 | Classification | Shared MLP + MaxPool |
| PointNet++ | 2017 | Segmentation | Hierarchical grouping |
| DGCNN | 2019 | Classification | Dynamic graph on points |
| Point Transformer | 2021 | Segmentation | Self-attention on points |
| PointNeXt | 2022 | Classification | Improved PointNet++ |

### Key Libraries

| Library | Use Case |
|---------|----------|
| Open3D | Point cloud processing, visualization |
| COLMAP | Structure from Motion, MVS |
| Nerfstudio | NeRF training and rendering |
| gsplat | Gaussian Splatting |
| PyTorch3D | 3D deep learning (Facebook) |
| trimesh | Mesh processing |
| Depth Anything | Monocular depth |

### Camera Intrinsics Cheat Sheet

$$\mathbf{K} = \begin{bmatrix} f_x & 0 & c_x \\ 0 & f_y & c_y \\ 0 & 0 & 1 \end{bmatrix}$$

| Parameter | What It Is | Typical Value |
|-----------|-----------|---------------|
| $f_x, f_y$ | Focal length (pixels) | 500-1000 for phone cameras |
| $c_x, c_y$ | Principal point (image center) | ~width/2, ~height/2 |
| Distortion | Lens curvature correction | Calibrate with checkerboard |

---

> **Key Takeaway:** 3D vision is rapidly evolving. Depth estimation is now "solved" for most applications (use Depth Anything v2). For 3D reconstruction, 3D Gaussian Splatting has largely replaced NeRF for real-time applications. Point cloud processing is essential for LiDAR/robotics. Start with existing tools (COLMAP, Nerfstudio, Open3D) before building custom solutions.
