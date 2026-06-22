# Chapter 08: OpenCV Advanced — Image Processing, Feature Detection, Contours, Morphology & Video I/O

---

## Table of Contents
1. [OpenCV Fundamentals Recap](#1-opencv-fundamentals-recap)
2. [Image Filtering & Convolutions](#2-image-filtering--convolutions)
3. [Edge Detection](#3-edge-detection)
4. [Morphological Operations](#4-morphological-operations)
5. [Contour Detection & Analysis](#5-contour-detection--analysis)
6. [Feature Detection & Matching](#6-feature-detection--matching)
7. [Image Transformations](#7-image-transformations)
8. [Video I/O & Real-time Processing](#8-video-io--real-time-processing)
9. [Advanced Techniques](#9-advanced-techniques)
10. [Common Mistakes](#10-common-mistakes)
11. [Interview Questions](#11-interview-questions)
12. [Quick Reference](#12-quick-reference)

---

## 1. OpenCV Fundamentals Recap

### What It Is
OpenCV (Open Source Computer Vision Library) is THE standard library for computer vision. It provides 2500+ algorithms for image/video processing, from basic operations to advanced ML. Think of it as the "Swiss Army knife" of computer vision.

### Why It Matters
- Used in nearly every CV application in production
- Extremely optimized (C++ backend with Python bindings)
- Covers everything: I/O, preprocessing, feature detection, tracking, calibration
- Free, open-source, cross-platform

### Core Concepts

```python
import cv2
import numpy as np

# ============================================
# Image Basics
# ============================================

# Reading images
img = cv2.imread('image.jpg')           # BGR format (not RGB!)
img_gray = cv2.imread('image.jpg', 0)   # Grayscale
img_unchanged = cv2.imread('image.jpg', cv2.IMREAD_UNCHANGED)  # With alpha

# Image properties
print(f"Shape: {img.shape}")    # (height, width, channels)
print(f"Dtype: {img.dtype}")    # uint8 (0-255)
print(f"Size: {img.size}")      # total pixels × channels

# Color space conversions
rgb = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
hsv = cv2.cvtColor(img, cv2.COLOR_BGR2HSV)
lab = cv2.cvtColor(img, cv2.COLOR_BGR2LAB)

# IMPORTANT: OpenCV uses BGR, matplotlib uses RGB!
# Always convert when displaying with matplotlib:
# plt.imshow(cv2.cvtColor(img, cv2.COLOR_BGR2RGB))

# Pixel access
pixel = img[100, 200]          # (B, G, R) at row=100, col=200
blue = img[100, 200, 0]       # Blue channel value

# ROI (Region of Interest)
roi = img[50:200, 100:300]    # Slice: rows 50-200, cols 100-300

# Saving
cv2.imwrite('output.png', img)  # Format from extension
cv2.imwrite('output.jpg', img, [cv2.IMWRITE_JPEG_QUALITY, 95])
```

```
Memory Layout:
OpenCV images are NumPy arrays!

img.shape = (H, W, C) = (480, 640, 3)

┌─────────────────────────────────────────────────┐
│ Row 0: [B,G,R][B,G,R][B,G,R]...[B,G,R] (640 px)│
│ Row 1: [B,G,R][B,G,R][B,G,R]...[B,G,R]         │
│ ...                                              │
│ Row 479: [B,G,R][B,G,R]...[B,G,R]               │
└─────────────────────────────────────────────────┘

dtype: uint8 (0-255) for display
       float32 (0.0-1.0) for computations
```

---

## 2. Image Filtering & Convolutions

### What It Is
Image filtering means modifying pixel values based on their neighborhood. A **kernel** (small matrix) slides over the image, computing a weighted sum at each position. This is the foundation of ALL image processing.

### Why It Matters
- Noise reduction (smoothing/blurring)
- Edge enhancement (sharpening)
- Feature extraction (used inside CNNs!)
- Preprocessing for detection/segmentation

### How It Works

```
Convolution Operation:

Input Image      Kernel (3×3)       Output
┌───┬───┬───┐   ┌───┬───┬───┐
│ 1 │ 2 │ 3 │   │ 0 │-1 │ 0 │
├───┼───┼───┤   ├───┼───┼───┤
│ 4 │ 5 │ 6 │ * │-1 │ 5 │-1 │  = 5×5 + (-1)(2+4+6+8) = 25-20 = 5
├───┼───┼───┤   ├───┼───┼───┤
│ 7 │ 8 │ 9 │   │ 0 │-1 │ 0 │
└───┴───┴───┘   └───┴───┴───┘

The kernel "slides" across the entire image.
Each output pixel = sum of element-wise multiplication.
```

### Code Examples

```python
import cv2
import numpy as np

# ============================================
# Blurring / Smoothing
# ============================================

img = cv2.imread('noisy_image.jpg')

# 1. Average Blur (box filter)
# Every pixel in kernel has equal weight
blur_avg = cv2.blur(img, (5, 5))  # 5×5 kernel

# 2. Gaussian Blur (center-weighted)
# More natural — center pixels contribute more
blur_gauss = cv2.GaussianBlur(img, (5, 5), sigmaX=1.5)
# sigmaX: standard deviation (larger = more blur)
# Kernel size must be ODD

# 3. Median Blur (best for salt & pepper noise!)
# Replaces pixel with MEDIAN of neighborhood
# Preserves edges better than Gaussian
blur_median = cv2.medianBlur(img, 5)

# 4. Bilateral Filter (blur while preserving edges!)
# The holy grail: smooth flat areas, keep sharp edges
blur_bilateral = cv2.bilateralFilter(
    img,
    d=9,              # Diameter of pixel neighborhood
    sigmaColor=75,    # How different colors can be to still average
    sigmaSpace=75     # How far pixels can be to still influence
)
# sigmaColor small → only VERY similar pixels averaged (edge preserved)
# sigmaColor large → more averaging (more blur)

# Pro Tip: Use bilateralFilter for portrait smoothing / beauty filters
```

```python
# ============================================
# Sharpening
# ============================================

# Method 1: Unsharp mask (Gaussian blur → subtract → add)
def unsharp_mask(image, sigma=1.0, strength=1.5):
    """
    Sharpen by enhancing edges.
    sharp = original + strength * (original - blurred)
    """
    blurred = cv2.GaussianBlur(image, (0, 0), sigma)
    sharpened = cv2.addWeighted(image, 1 + strength, blurred, -strength, 0)
    return sharpened

# Method 2: Custom sharpening kernel
kernel_sharpen = np.array([
    [ 0, -1,  0],
    [-1,  5, -1],
    [ 0, -1,  0]
], dtype=np.float32)

sharpened = cv2.filter2D(img, -1, kernel_sharpen)

# Method 3: Laplacian-based sharpening
laplacian = cv2.Laplacian(img, cv2.CV_64F)
sharp = img - 0.7 * laplacian  # Subtract edges to sharpen
sharp = np.clip(sharp, 0, 255).astype(np.uint8)
```

```python
# ============================================
# Custom Kernels with filter2D
# ============================================

# Emboss effect
kernel_emboss = np.array([
    [-2, -1, 0],
    [-1,  1, 1],
    [ 0,  1, 2]
], dtype=np.float32)
embossed = cv2.filter2D(img, -1, kernel_emboss)

# Motion blur (simulates camera movement)
def motion_blur(image, size=15, angle=0):
    """Apply directional motion blur."""
    kernel = np.zeros((size, size))
    kernel[size//2, :] = 1.0 / size  # Horizontal line
    # Rotate kernel for different angles
    M = cv2.getRotationMatrix2D((size/2, size/2), angle, 1.0)
    kernel = cv2.warpAffine(kernel, M, (size, size))
    return cv2.filter2D(image, -1, kernel)

# Large kernel Gaussian (for bokeh-like effect)
large_blur = cv2.GaussianBlur(img, (51, 51), 0)
```

> **Pro Tip:** For real-time applications, use `cv2.sepFilter2D()` for separable kernels. A 5×5 Gaussian can be decomposed into two 1×5 filters — 10 operations instead of 25!

---

## 3. Edge Detection

### What It Is
Edge detection finds boundaries in images — places where pixel intensity changes rapidly. Edges correspond to object boundaries, texture changes, and depth discontinuities.

### Why It Matters
- Object boundary detection
- Shape recognition
- Line/lane detection for driving
- Document scanning (find paper edges)
- Input to many higher-level algorithms

### How It Works

```
Gradient-based Edge Detection:

An edge = rapid change in intensity
         = HIGH GRADIENT

Smooth area:    Edge area:
... 50 50 50    ... 50 50 200 200 200
    gradient≈0       gradient=HIGH

Gradient magnitude: √(Gx² + Gy²)
Gradient direction: arctan(Gy/Gx)
```

**Comparison of Edge Detectors:**

| Detector | Method | Pros | Cons |
|----------|--------|------|------|
| Sobel | 1st derivative | Fast, directional | Thick edges, noisy |
| Laplacian | 2nd derivative | Thin edges | Very noise-sensitive |
| Canny | Multi-stage | Thin, connected, best quality | Slower, 2 thresholds to tune |
| Scharr | Enhanced Sobel | Better rotational symmetry | Same limitations as Sobel |

### Code Examples

```python
import cv2
import numpy as np

img = cv2.imread('image.jpg', cv2.IMREAD_GRAYSCALE)

# ============================================
# Sobel Edge Detection (Gradient-based)
# ============================================

# Sobel X: detects VERTICAL edges (horizontal gradients)
sobel_x = cv2.Sobel(img, cv2.CV_64F, 1, 0, ksize=3)

# Sobel Y: detects HORIZONTAL edges (vertical gradients)
sobel_y = cv2.Sobel(img, cv2.CV_64F, 0, 1, ksize=3)

# Combined magnitude
sobel_magnitude = np.sqrt(sobel_x**2 + sobel_y**2)
sobel_magnitude = np.uint8(np.clip(sobel_magnitude, 0, 255))

# Gradient direction (useful for oriented edge detection)
gradient_direction = np.arctan2(sobel_y, sobel_x) * 180 / np.pi

# ============================================
# Canny Edge Detection (Gold Standard)
# ============================================

"""
Canny Algorithm Steps:
1. Gaussian blur (reduce noise)
2. Compute gradient magnitude and direction (Sobel)
3. Non-Maximum Suppression (thin edges to 1px)
4. Double thresholding (strong/weak edges)
5. Edge tracking by hysteresis (connect weak to strong)
"""

# Basic usage
edges = cv2.Canny(img, threshold1=50, threshold2=150)
# threshold1 (low): edges below this are discarded
# threshold2 (high): edges above this are kept
# Between: kept only if connected to a strong edge

# With automatic thresholds (Otsu-based)
def auto_canny(image, sigma=0.33):
    """Automatically determine Canny thresholds."""
    median = np.median(image)
    lower = int(max(0, (1.0 - sigma) * median))
    upper = int(min(255, (1.0 + sigma) * median))
    return cv2.Canny(image, lower, upper)

edges_auto = auto_canny(img)

# With pre-blurring (recommended for noisy images)
blurred = cv2.GaussianBlur(img, (5, 5), 1.4)
edges_clean = cv2.Canny(blurred, 50, 150)

# ============================================
# Laplacian (2nd derivative — zero crossings = edges)
# ============================================

laplacian = cv2.Laplacian(img, cv2.CV_64F)
laplacian_abs = np.uint8(np.absolute(laplacian))

# ============================================
# Practical: Line Detection with Hough Transform
# ============================================

def detect_lines(image):
    """Detect straight lines using Hough Transform."""
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    edges = cv2.Canny(gray, 50, 150)
    
    # Probabilistic Hough Transform (more practical)
    lines = cv2.HoughLinesP(
        edges,
        rho=1,              # Distance resolution (pixels)
        theta=np.pi/180,    # Angle resolution (radians)
        threshold=100,      # Min votes to be a line
        minLineLength=50,   # Min length of line segment
        maxLineGap=10       # Max gap between line segments
    )
    
    # Draw detected lines
    result = image.copy()
    if lines is not None:
        for line in lines:
            x1, y1, x2, y2 = line[0]
            cv2.line(result, (x1, y1), (x2, y2), (0, 255, 0), 2)
    
    return result, lines

# Circle detection
def detect_circles(image):
    """Detect circles using Hough Circle Transform."""
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    gray = cv2.medianBlur(gray, 5)  # Reduce noise
    
    circles = cv2.HoughCircles(
        gray,
        cv2.HOUGH_GRADIENT,
        dp=1,           # Inverse ratio of accumulator resolution
        minDist=50,     # Min distance between circle centers
        param1=100,     # Canny high threshold
        param2=30,      # Accumulator threshold (lower = more circles)
        minRadius=10,   # Min radius
        maxRadius=100   # Max radius
    )
    
    result = image.copy()
    if circles is not None:
        circles = np.uint16(np.around(circles))
        for circle in circles[0, :]:
            center = (circle[0], circle[1])
            radius = circle[2]
            cv2.circle(result, center, radius, (0, 255, 0), 2)
            cv2.circle(result, center, 2, (0, 0, 255), 3)  # Center
    
    return result
```

---

## 4. Morphological Operations

### What It Is
Morphological operations modify the shape of features in binary/grayscale images using a structuring element (kernel). They're like "sculpting" an image — eroding thin structures, filling gaps, or extracting boundaries.

### Why It Matters
- Remove noise from binary masks
- Separate touching objects
- Fill holes in detected regions
- Extract boundaries/skeletons
- Critical preprocessing for OCR, document analysis, medical imaging

### How It Works

```
Structuring Element (SE):       Operations:
┌───┬───┬───┐
│ 1 │ 1 │ 1 │                  Erosion: Shrink white regions
├───┼───┼───┤                  (pixel is white only if ALL
│ 1 │ 1 │ 1 │                   neighbors under SE are white)
├───┼───┼───┤
│ 1 │ 1 │ 1 │                  Dilation: Grow white regions
└───┴───┴───┘                  (pixel is white if ANY
  3×3 square                    neighbor under SE is white)

Original:     Erosion:      Dilation:
██████████    ░░████░░      ████████████
██████████    ░░████░░      ████████████
██░░░░░░██    ░░░░░░░░░░    ████░░░░████
██░░░░░░██    ░░░░░░░░░░    ████░░░░████
██████████    ░░████░░      ████████████
██████████    ░░████░░      ████████████

Opening = Erosion → Dilation (removes small noise)
Closing = Dilation → Erosion (fills small holes)
```

### Code Examples

```python
import cv2
import numpy as np

# Create binary image (threshold)
img = cv2.imread('document.jpg', cv2.IMREAD_GRAYSCALE)
_, binary = cv2.threshold(img, 127, 255, cv2.THRESH_BINARY)

# ============================================
# Structuring Elements
# ============================================

# Different shapes for different purposes
kernel_rect = cv2.getStructuringElement(cv2.MORPH_RECT, (5, 5))
kernel_ellipse = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (5, 5))
kernel_cross = cv2.getStructuringElement(cv2.MORPH_CROSS, (5, 5))

# Custom kernel (e.g., horizontal line for text extraction)
kernel_hline = np.ones((1, 30), dtype=np.uint8)  # Detect horizontal lines
kernel_vline = np.ones((30, 1), dtype=np.uint8)  # Detect vertical lines

# ============================================
# Basic Operations
# ============================================

# Erosion: Shrink white regions, remove small noise
eroded = cv2.erode(binary, kernel_rect, iterations=1)

# Dilation: Grow white regions, fill small gaps
dilated = cv2.dilate(binary, kernel_rect, iterations=1)

# Opening: Erosion → Dilation (remove small white noise)
# Use when: Small bright spots/noise on dark background
opened = cv2.morphologyEx(binary, cv2.MORPH_OPEN, kernel_rect)

# Closing: Dilation → Erosion (fill small holes)
# Use when: Small dark spots/holes in white regions
closed = cv2.morphologyEx(binary, cv2.MORPH_CLOSE, kernel_rect)

# ============================================
# Advanced Operations
# ============================================

# Gradient: Dilation - Erosion (outlines/boundaries)
gradient = cv2.morphologyEx(binary, cv2.MORPH_GRADIENT, kernel_rect)

# Top Hat: Original - Opening (bright features on dark background)
tophat = cv2.morphologyEx(img, cv2.MORPH_TOPHAT, kernel_rect)

# Black Hat: Closing - Original (dark features on bright background)
blackhat = cv2.morphologyEx(img, cv2.MORPH_BLACKHAT, kernel_rect)

# ============================================
# Practical Applications
# ============================================

def remove_noise_from_mask(mask, kernel_size=3):
    """Clean up a segmentation mask."""
    kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (kernel_size, kernel_size))
    
    # Remove small noise (opening)
    cleaned = cv2.morphologyEx(mask, cv2.MORPH_OPEN, kernel, iterations=2)
    # Fill small holes (closing)
    cleaned = cv2.morphologyEx(cleaned, cv2.MORPH_CLOSE, kernel, iterations=2)
    
    return cleaned

def separate_touching_objects(binary_mask):
    """
    Separate objects that are touching using erosion + distance transform.
    Common in cell counting, coin detection, etc.
    """
    # Distance transform: value = distance to nearest background pixel
    dist = cv2.distanceTransform(binary_mask, cv2.DIST_L2, 5)
    
    # Threshold distance to find object centers
    _, sure_fg = cv2.threshold(dist, 0.5 * dist.max(), 255, 0)
    sure_fg = sure_fg.astype(np.uint8)
    
    # Sure background (dilated)
    sure_bg = cv2.dilate(binary_mask, None, iterations=3)
    
    # Unknown region (between sure foreground and background)
    unknown = cv2.subtract(sure_bg, sure_fg)
    
    # Watershed markers
    _, markers = cv2.connectedComponents(sure_fg)
    markers = markers + 1  # Ensure background is not 0
    markers[unknown == 255] = 0  # Mark unknown regions
    
    return markers, dist

def extract_text_lines(document_img):
    """Extract horizontal text lines from a document image."""
    gray = cv2.cvtColor(document_img, cv2.COLOR_BGR2GRAY)
    _, binary = cv2.threshold(gray, 0, 255, cv2.THRESH_BINARY_INV + cv2.THRESH_OTSU)
    
    # Horizontal kernel to connect characters in a line
    kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (40, 1))
    
    # Dilate to connect text in same line
    dilated = cv2.dilate(binary, kernel, iterations=1)
    
    # Find contours of text lines
    contours, _ = cv2.findContours(dilated, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    
    # Sort top to bottom
    bounding_boxes = [cv2.boundingRect(c) for c in contours]
    bounding_boxes.sort(key=lambda b: b[1])  # Sort by y coordinate
    
    return bounding_boxes
```

---

## 5. Contour Detection & Analysis

### What It Is
Contours are curves joining all continuous points along a boundary with the same color/intensity. They're like the "outline" of objects. Contour analysis extracts properties like area, perimeter, shape, and hierarchy.

### Why It Matters
- Object detection without deep learning (for simple shapes)
- Shape analysis and recognition
- Area/perimeter measurements
- Object counting
- Document scanning (find document edges)

### How It Works

```
Binary Image → findContours → List of Contours

Contour = list of (x, y) boundary points

Original Image:           Contours Found:
┌──────────────────┐     ┌──────────────────┐
│                  │     │                  │
│   ████████       │     │   ┌──────┐       │
│   █      █       │     │   │      │       │
│   █      █       │  →  │   │      │       │
│   ████████       │     │   └──────┘       │
│                  │     │                  │
│      ●●●●        │     │      ○○○○        │
│     ●    ●       │     │     ○    ○       │
│      ●●●●        │     │      ○○○○        │
└──────────────────┘     └──────────────────┘
                         2 contours found!
```

### Code Examples

```python
import cv2
import numpy as np

# ============================================
# Finding and Drawing Contours
# ============================================

img = cv2.imread('shapes.jpg')
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

# Threshold to binary (required for contour detection)
_, binary = cv2.threshold(gray, 127, 255, cv2.THRESH_BINARY)

# Find contours
contours, hierarchy = cv2.findContours(
    binary,
    cv2.RETR_TREE,          # Retrieval mode (hierarchy)
    cv2.CHAIN_APPROX_SIMPLE  # Approximation method
)

# Retrieval modes:
# RETR_EXTERNAL: Only outermost contours
# RETR_LIST:     All contours, no hierarchy
# RETR_TREE:     Full hierarchy (parent-child relationships)
# RETR_CCOMP:    Two-level hierarchy

# Draw all contours
result = img.copy()
cv2.drawContours(result, contours, -1, (0, 255, 0), 2)  # -1 = all

# Draw specific contour
cv2.drawContours(result, contours, 0, (0, 0, 255), 3)  # First contour

print(f"Found {len(contours)} contours")

# ============================================
# Contour Properties
# ============================================

for i, contour in enumerate(contours):
    # Area
    area = cv2.contourArea(contour)
    
    # Perimeter (arc length)
    perimeter = cv2.arcLength(contour, closed=True)
    
    # Bounding rectangle (axis-aligned)
    x, y, w, h = cv2.boundingRect(contour)
    
    # Rotated bounding rectangle (minimum area)
    rect = cv2.minAreaRect(contour)  # ((cx,cy), (w,h), angle)
    box = cv2.boxPoints(rect)
    box = np.int32(box)
    
    # Minimum enclosing circle
    (cx, cy), radius = cv2.minEnclosingCircle(contour)
    
    # Centroid (center of mass)
    M = cv2.moments(contour)
    if M["m00"] != 0:
        centroid_x = int(M["m10"] / M["m00"])
        centroid_y = int(M["m01"] / M["m00"])
    
    # Aspect ratio
    aspect_ratio = float(w) / h
    
    # Extent (contour area / bounding rect area)
    extent = area / (w * h)
    
    # Solidity (contour area / convex hull area)
    hull = cv2.convexHull(contour)
    hull_area = cv2.contourArea(hull)
    solidity = area / hull_area if hull_area > 0 else 0
    
    # Circularity: 4π × Area / Perimeter²
    # Perfect circle = 1.0
    circularity = 4 * np.pi * area / (perimeter ** 2) if perimeter > 0 else 0
    
    print(f"Contour {i}: area={area:.0f}, perimeter={perimeter:.1f}, "
          f"circularity={circularity:.3f}, solidity={solidity:.3f}")

# ============================================
# Shape Detection (Polygon Approximation)
# ============================================

def detect_shape(contour):
    """Identify shape based on number of vertices."""
    # Approximate contour to polygon
    perimeter = cv2.arcLength(contour, True)
    epsilon = 0.02 * perimeter  # Approximation accuracy
    approx = cv2.approxPolyDP(contour, epsilon, True)
    
    num_vertices = len(approx)
    
    if num_vertices == 3:
        return "Triangle"
    elif num_vertices == 4:
        # Check if it's a square or rectangle
        x, y, w, h = cv2.boundingRect(approx)
        aspect_ratio = w / float(h)
        if 0.9 <= aspect_ratio <= 1.1:
            return "Square"
        else:
            return "Rectangle"
    elif num_vertices == 5:
        return "Pentagon"
    elif num_vertices == 6:
        return "Hexagon"
    else:
        # Check circularity
        area = cv2.contourArea(contour)
        perimeter = cv2.arcLength(contour, True)
        circularity = 4 * np.pi * area / (perimeter ** 2)
        if circularity > 0.8:
            return "Circle"
        else:
            return f"Polygon ({num_vertices} sides)"

# Detect and label shapes
for contour in contours:
    if cv2.contourArea(contour) < 100:  # Skip tiny contours (noise)
        continue
    
    shape = detect_shape(contour)
    M = cv2.moments(contour)
    if M["m00"] != 0:
        cx = int(M["m10"] / M["m00"])
        cy = int(M["m01"] / M["m00"])
        cv2.putText(result, shape, (cx - 30, cy),
                   cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 0, 0), 2)

# ============================================
# Practical: Document Scanner
# ============================================

def find_document(image):
    """
    Find document corners in an image (like CamScanner).
    Returns 4 corner points of the document.
    """
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    blurred = cv2.GaussianBlur(gray, (5, 5), 0)
    edges = cv2.Canny(blurred, 50, 150)
    
    # Dilate to close gaps in edges
    edges = cv2.dilate(edges, None, iterations=2)
    
    # Find contours
    contours, _ = cv2.findContours(edges, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    
    # Sort by area (document should be the largest contour)
    contours = sorted(contours, key=cv2.contourArea, reverse=True)
    
    for contour in contours[:5]:  # Check top 5 largest
        perimeter = cv2.arcLength(contour, True)
        approx = cv2.approxPolyDP(contour, 0.02 * perimeter, True)
        
        if len(approx) == 4:  # Found quadrilateral!
            return approx.reshape(4, 2)
    
    return None

def four_point_transform(image, points):
    """
    Perspective transform to get top-down view of document.
    """
    # Order points: top-left, top-right, bottom-right, bottom-left
    rect = order_points(points)
    tl, tr, br, bl = rect
    
    # Compute new image dimensions
    width = int(max(
        np.linalg.norm(br - bl),
        np.linalg.norm(tr - tl)
    ))
    height = int(max(
        np.linalg.norm(tr - br),
        np.linalg.norm(tl - bl)
    ))
    
    # Destination points
    dst = np.array([
        [0, 0], [width-1, 0],
        [width-1, height-1], [0, height-1]
    ], dtype=np.float32)
    
    # Perspective transform
    M = cv2.getPerspectiveTransform(rect.astype(np.float32), dst)
    warped = cv2.warpPerspective(image, M, (width, height))
    
    return warped

def order_points(pts):
    """Order points as: top-left, top-right, bottom-right, bottom-left."""
    rect = np.zeros((4, 2), dtype=np.float32)
    s = pts.sum(axis=1)
    rect[0] = pts[np.argmin(s)]   # Top-left (smallest x+y)
    rect[2] = pts[np.argmax(s)]   # Bottom-right (largest x+y)
    d = np.diff(pts, axis=1)
    rect[1] = pts[np.argmin(d)]   # Top-right (smallest y-x)
    rect[3] = pts[np.argmax(d)]   # Bottom-left (largest y-x)
    return rect
```

---

## 6. Feature Detection & Matching

### What It Is
Feature detection finds distinctive keypoints in images (corners, blobs) that can be reliably found again in other images. Feature matching finds correspondences between keypoints in different images of the same scene.

### Why It Matters
- Image stitching (panoramas)
- Object recognition (find known object in scene)
- Visual SLAM (robot navigation)
- 3D reconstruction (match points across views)
- Image registration (align medical scans)

### How It Works

```
Feature Detection Pipeline:

1. DETECT keypoints (where are interesting points?)
2. DESCRIBE each keypoint (what does it look like?)
3. MATCH descriptors between images

Image A:          Image B (different view):
┌──────────┐     ┌──────────┐
│  ●    ●  │     │    ●  ●  │
│      ●   │     │   ●      │
│  ●       │     │       ●  │
│     ●    │     │  ●    ●  │
└──────────┘     └──────────┘

After matching:
●─────────────────●  (same point, different view!)
   ●──────────●
        ●─────────●
```

**Feature Detector Comparison:**

| Detector | Type | Speed | Quality | Patent |
|----------|------|-------|---------|--------|
| Harris | Corner | Fast | Basic | Free |
| SIFT | Blob/Corner | Slow | Excellent | Free (2020+) |
| SURF | Blob | Medium | Very Good | Patented |
| ORB | Corner | Very Fast | Good | Free |
| AKAZE | Blob | Medium | Very Good | Free |
| SuperPoint | Learned | GPU-fast | Excellent | Free |

### Code Examples

```python
import cv2
import numpy as np

# ============================================
# Feature Detection
# ============================================

img = cv2.imread('building.jpg')
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

# --- ORB (Fast, free, good for real-time) ---
orb = cv2.ORB_create(nfeatures=1000)
keypoints_orb, descriptors_orb = orb.detectAndCompute(gray, None)
# descriptors: binary (Hamming distance for matching)

# --- SIFT (Slower but more robust) ---
sift = cv2.SIFT_create(nfeatures=1000)
keypoints_sift, descriptors_sift = sift.detectAndCompute(gray, None)
# descriptors: float32, 128-d (L2 distance for matching)

# --- AKAZE (Good balance of speed and quality) ---
akaze = cv2.AKAZE_create()
keypoints_akaze, descriptors_akaze = akaze.detectAndCompute(gray, None)

# Draw keypoints
img_keypoints = cv2.drawKeypoints(
    img, keypoints_sift, None,
    flags=cv2.DRAW_MATCHES_FLAGS_DRAW_RICH_KEYPOINTS  # Shows size & orientation
)

print(f"ORB: {len(keypoints_orb)} keypoints")
print(f"SIFT: {len(keypoints_sift)} keypoints")

# ============================================
# Feature Matching
# ============================================

img1 = cv2.imread('scene1.jpg', cv2.IMREAD_GRAYSCALE)
img2 = cv2.imread('scene2.jpg', cv2.IMREAD_GRAYSCALE)

# Detect and describe
sift = cv2.SIFT_create()
kp1, des1 = sift.detectAndCompute(img1, None)
kp2, des2 = sift.detectAndCompute(img2, None)

# --- Method 1: Brute-Force Matcher ---
bf = cv2.BFMatcher(cv2.NORM_L2, crossCheck=True)  # L2 for SIFT
matches = bf.match(des1, des2)
matches = sorted(matches, key=lambda x: x.distance)

# --- Method 2: FLANN Matcher (Faster for large datasets) ---
FLANN_INDEX_KDTREE = 1
index_params = dict(algorithm=FLANN_INDEX_KDTREE, trees=5)
search_params = dict(checks=50)

flann = cv2.FlannBasedMatcher(index_params, search_params)
matches_knn = flann.knnMatch(des1, des2, k=2)  # k=2 for ratio test

# --- Lowe's Ratio Test (CRITICAL for good matches!) ---
# Only keep matches where best match is significantly better than 2nd best
good_matches = []
for m, n in matches_knn:
    if m.distance < 0.7 * n.distance:  # 0.7 is Lowe's threshold
        good_matches.append(m)

print(f"Total matches: {len(matches_knn)}")
print(f"Good matches (ratio test): {len(good_matches)}")

# Draw matches
img_matches = cv2.drawMatches(
    img1, kp1, img2, kp2, good_matches[:50], None,
    flags=cv2.DrawMatchesFlags_NOT_DRAW_SINGLE_POINTS
)

# ============================================
# Homography Estimation (for panoramas, registration)
# ============================================

def find_homography_robust(kp1, kp2, good_matches, threshold=5.0):
    """
    Find perspective transformation between two images.
    Uses RANSAC to handle outlier matches.
    """
    if len(good_matches) < 4:
        return None, None
    
    # Extract matched point coordinates
    src_pts = np.float32([kp1[m.queryIdx].pt for m in good_matches]).reshape(-1, 1, 2)
    dst_pts = np.float32([kp2[m.trainIdx].pt for m in good_matches]).reshape(-1, 1, 2)
    
    # Find homography with RANSAC
    H, mask = cv2.findHomography(src_pts, dst_pts, cv2.RANSAC, threshold)
    
    # mask tells which matches are inliers (consistent with H)
    inlier_ratio = mask.sum() / len(mask) if mask is not None else 0
    print(f"Inlier ratio: {inlier_ratio:.2%}")
    
    return H, mask

# Usage for image stitching:
H, mask = find_homography_robust(kp1, kp2, good_matches)
if H is not None:
    # Warp img1 to align with img2
    h, w = img2.shape[:2]
    warped = cv2.warpPerspective(img1, H, (w * 2, h))
    # Blend warped with img2 for panorama
    warped[0:h, 0:w] = img2

# ============================================
# Template Matching (Simple object detection)
# ============================================

def template_match(image, template, threshold=0.8):
    """
    Find template (small image) inside a larger image.
    Good for: icons, logos, fixed-appearance objects.
    """
    result = cv2.matchTemplate(image, template, cv2.TM_CCOEFF_NORMED)
    
    # Find all locations above threshold
    locations = np.where(result >= threshold)
    
    # Get bounding boxes
    h, w = template.shape[:2]
    boxes = []
    for pt in zip(*locations[::-1]):  # (x, y) format
        boxes.append((pt[0], pt[1], pt[0] + w, pt[1] + h))
    
    # Non-maximum suppression (remove overlapping)
    boxes = non_max_suppression(boxes, overlapThresh=0.3)
    
    return boxes
```

---

## 7. Image Transformations

### What It Is
Image transformations change the geometry of images — rotating, scaling, flipping, warping, or changing perspective. Essential for data augmentation, alignment, and correcting distortions.

### Code Examples

```python
import cv2
import numpy as np

img = cv2.imread('image.jpg')
h, w = img.shape[:2]

# ============================================
# Affine Transformations
# ============================================

# Rotation
center = (w // 2, h // 2)
angle = 45  # degrees
scale = 1.0
M = cv2.getRotationMatrix2D(center, angle, scale)
rotated = cv2.warpAffine(img, M, (w, h))

# Rotation without cropping (expand canvas)
def rotate_no_crop(image, angle):
    """Rotate image without cutting off corners."""
    h, w = image.shape[:2]
    center = (w / 2, h / 2)
    M = cv2.getRotationMatrix2D(center, angle, 1.0)
    
    # Compute new bounding box
    cos = np.abs(M[0, 0])
    sin = np.abs(M[0, 1])
    new_w = int(h * sin + w * cos)
    new_h = int(h * cos + w * sin)
    
    # Adjust translation
    M[0, 2] += (new_w - w) / 2
    M[1, 2] += (new_h - h) / 2
    
    return cv2.warpAffine(image, M, (new_w, new_h))

# Translation (shifting)
tx, ty = 100, 50  # Shift right 100px, down 50px
M_translate = np.float32([[1, 0, tx], [0, 1, ty]])
translated = cv2.warpAffine(img, M_translate, (w, h))

# Scaling (resize)
resized = cv2.resize(img, (640, 480))  # Exact size
resized = cv2.resize(img, None, fx=0.5, fy=0.5)  # Half size

# Interpolation methods (for resize):
# INTER_NEAREST: Fastest, pixelated (good for masks)
# INTER_LINEAR:  Default, good balance
# INTER_CUBIC:   Higher quality upscaling
# INTER_LANCZOS4: Best quality, slowest
# INTER_AREA:    Best for downscaling

# Flipping
flipped_h = cv2.flip(img, 1)   # Horizontal flip
flipped_v = cv2.flip(img, 0)   # Vertical flip
flipped_both = cv2.flip(img, -1)  # Both

# ============================================
# Perspective Transform
# ============================================

def perspective_correction(image, src_points, width, height):
    """
    Correct perspective distortion (e.g., photo of a sign taken at angle).
    
    src_points: 4 points in the source image (what you see)
    width, height: desired output dimensions
    """
    src = np.float32(src_points)
    dst = np.float32([
        [0, 0], [width, 0],
        [width, height], [0, height]
    ])
    
    M = cv2.getPerspectiveTransform(src, dst)
    corrected = cv2.warpPerspective(image, M, (width, height))
    
    return corrected

# ============================================
# Histogram Equalization (contrast enhancement)
# ============================================

gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

# Global histogram equalization
equalized = cv2.equalizeHist(gray)

# CLAHE: Adaptive histogram equalization (better!)
# Prevents over-amplification of noise in uniform regions
clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8))
clahe_result = clahe.apply(gray)

# For color images: apply to L channel in LAB space
def equalize_color(image):
    """Apply CLAHE to color image (preserve hue)."""
    lab = cv2.cvtColor(image, cv2.COLOR_BGR2LAB)
    l, a, b = cv2.split(lab)
    
    clahe = cv2.createCLAHE(clipLimit=3.0, tileGridSize=(8, 8))
    l_eq = clahe.apply(l)
    
    lab_eq = cv2.merge([l_eq, a, b])
    return cv2.cvtColor(lab_eq, cv2.COLOR_LAB2BGR)

# ============================================
# Thresholding (Binary Segmentation)
# ============================================

# Simple threshold
_, binary = cv2.threshold(gray, 127, 255, cv2.THRESH_BINARY)

# Otsu's method (automatic threshold selection)
_, otsu = cv2.threshold(gray, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)

# Adaptive threshold (handles uneven lighting!)
adaptive = cv2.adaptiveThreshold(
    gray, 255,
    cv2.ADAPTIVE_THRESH_GAUSSIAN_C,  # Weighted mean of neighborhood
    cv2.THRESH_BINARY,
    blockSize=11,  # Neighborhood size (odd)
    C=2            # Constant subtracted from mean
)
# Use adaptive for: documents, text, uneven illumination
```

---

## 8. Video I/O & Real-time Processing

### What It Is
OpenCV's video module handles reading from files, cameras, streams, and writing processed video. Combined with processing pipelines, it enables real-time computer vision applications.

### Code Examples

```python
import cv2
import numpy as np
import time

# ============================================
# Video Capture (Files, Cameras, Streams)
# ============================================

# From file
cap = cv2.VideoCapture('video.mp4')

# From camera (0 = default, 1 = external)
cap = cv2.VideoCapture(0)

# From RTSP stream
cap = cv2.VideoCapture('rtsp://username:password@ip:port/stream')

# From HTTP stream
cap = cv2.VideoCapture('http://ip:port/video')

# Set camera properties
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)
cap.set(cv2.CAP_PROP_FPS, 30)
cap.set(cv2.CAP_PROP_AUTOFOCUS, 0)  # Disable autofocus

# Get properties
fps = cap.get(cv2.CAP_PROP_FPS)
width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))

# ============================================
# Video Writer
# ============================================

fourcc = cv2.VideoWriter_fourcc(*'mp4v')  # Codec
# Common codecs: 'mp4v', 'XVID', 'MJPG', 'H264'
writer = cv2.VideoWriter('output.mp4', fourcc, fps, (width, height))

# ============================================
# Real-time Processing Loop
# ============================================

def real_time_processing():
    """Template for real-time video processing."""
    cap = cv2.VideoCapture(0)
    
    # FPS counter
    fps_counter = 0
    fps_timer = time.time()
    display_fps = 0
    
    while True:
        ret, frame = cap.read()
        if not ret:
            break
        
        # === YOUR PROCESSING HERE ===
        processed = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        processed = cv2.Canny(processed, 50, 150)
        processed = cv2.cvtColor(processed, cv2.COLOR_GRAY2BGR)
        # ============================
        
        # Calculate FPS
        fps_counter += 1
        if time.time() - fps_timer >= 1.0:
            display_fps = fps_counter
            fps_counter = 0
            fps_timer = time.time()
        
        # Overlay FPS
        cv2.putText(processed, f"FPS: {display_fps}", (10, 30),
                   cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)
        
        cv2.imshow('Real-time', processed)
        
        # Exit on 'q' key
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    
    cap.release()
    cv2.destroyAllWindows()

# ============================================
# Multi-threaded Video Capture (for speed)
# ============================================

from threading import Thread
from queue import Queue

class ThreadedVideoCapture:
    """
    Separate video capture into its own thread.
    Prevents I/O from blocking processing.
    """
    
    def __init__(self, source=0, queue_size=3):
        self.cap = cv2.VideoCapture(source)
        self.queue = Queue(maxsize=queue_size)
        self.stopped = False
        
    def start(self):
        Thread(target=self._update, daemon=True).start()
        return self
    
    def _update(self):
        while not self.stopped:
            if not self.queue.full():
                ret, frame = self.cap.read()
                if not ret:
                    self.stopped = True
                    break
                self.queue.put(frame)
            else:
                # Drop frames if queue is full (real-time = latest frame matters)
                self.cap.grab()
    
    def read(self):
        return self.queue.get()
    
    @property
    def is_running(self):
        return not self.stopped
    
    def stop(self):
        self.stopped = True
        self.cap.release()

# Usage:
# cap = ThreadedVideoCapture(0).start()
# while cap.is_running:
#     frame = cap.read()
#     # process frame...
# cap.stop()

# ============================================
# Video Processing Pipeline (Production)
# ============================================

class VideoProcessor:
    """Configurable video processing pipeline."""
    
    def __init__(self, source, output=None):
        self.cap = cv2.VideoCapture(source)
        self.fps = self.cap.get(cv2.CAP_PROP_FPS) or 30
        self.width = int(self.cap.get(cv2.CAP_PROP_FRAME_WIDTH))
        self.height = int(self.cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
        
        self.writer = None
        if output:
            fourcc = cv2.VideoWriter_fourcc(*'mp4v')
            self.writer = cv2.VideoWriter(output, fourcc, self.fps, 
                                         (self.width, self.height))
        
        self.processors = []  # List of processing functions
    
    def add_processor(self, func):
        """Add a processing function to the pipeline."""
        self.processors.append(func)
        return self
    
    def run(self, display=True, max_frames=None):
        """Run the processing pipeline."""
        frame_count = 0
        
        while self.cap.isOpened():
            ret, frame = self.cap.read()
            if not ret:
                break
            
            # Apply all processors in order
            result = frame
            for proc in self.processors:
                result = proc(result)
            
            # Write output
            if self.writer:
                self.writer.write(result)
            
            # Display
            if display:
                cv2.imshow('Output', result)
                if cv2.waitKey(1) & 0xFF == ord('q'):
                    break
            
            frame_count += 1
            if max_frames and frame_count >= max_frames:
                break
        
        self.cap.release()
        if self.writer:
            self.writer.release()
        cv2.destroyAllWindows()

# Example usage:
# pipeline = VideoProcessor('input.mp4', 'output.mp4')
# pipeline.add_processor(lambda f: cv2.GaussianBlur(f, (5,5), 0))
# pipeline.add_processor(detect_objects)  # Your custom function
# pipeline.add_processor(draw_annotations)
# pipeline.run()
```

---

## 9. Advanced Techniques

### Color-based Segmentation (HSV)

```python
# ============================================
# Color Detection in HSV Space
# ============================================

def detect_color(image, lower_hsv, upper_hsv):
    """
    Detect objects of a specific color.
    HSV is better than RGB for color detection because
    it separates color (H) from brightness (V).
    """
    hsv = cv2.cvtColor(image, cv2.COLOR_BGR2HSV)
    
    # Create mask for color range
    mask = cv2.inRange(hsv, lower_hsv, upper_hsv)
    
    # Clean up mask
    kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (5, 5))
    mask = cv2.morphologyEx(mask, cv2.MORPH_OPEN, kernel)
    mask = cv2.morphologyEx(mask, cv2.MORPH_CLOSE, kernel)
    
    # Apply mask to image
    result = cv2.bitwise_and(image, image, mask=mask)
    
    return mask, result

# Common HSV ranges (OpenCV: H=0-179, S=0-255, V=0-255)
# RED (wraps around 0/180!):
# lower_red1 = np.array([0, 100, 100])
# upper_red1 = np.array([10, 255, 255])
# lower_red2 = np.array([160, 100, 100])
# upper_red2 = np.array([179, 255, 255])

# GREEN:
lower_green = np.array([35, 100, 100])
upper_green = np.array([85, 255, 255])

# BLUE:
lower_blue = np.array([100, 100, 100])
upper_blue = np.array([130, 255, 255])

# YELLOW:
lower_yellow = np.array([20, 100, 100])
upper_yellow = np.array([35, 255, 255])
```

### Background Subtraction

```python
# ============================================
# Background Subtraction (Moving Object Detection)
# ============================================

def detect_motion(video_source=0):
    """Detect moving objects in video using background subtraction."""
    cap = cv2.VideoCapture(video_source)
    
    # MOG2 background subtractor (handles lighting changes)
    bg_subtractor = cv2.createBackgroundSubtractorMOG2(
        history=500,           # Frames used for background model
        varThreshold=16,       # Threshold for foreground detection
        detectShadows=True     # Detect and mark shadows
    )
    
    # Alternative: KNN background subtractor
    # bg_subtractor = cv2.createBackgroundSubtractorKNN(
    #     history=500, dist2Threshold=400, detectShadows=True
    # )
    
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        # Apply background subtraction
        fg_mask = bg_subtractor.apply(frame)
        
        # Remove shadows (marked as 127 in MOG2)
        _, fg_mask = cv2.threshold(fg_mask, 200, 255, cv2.THRESH_BINARY)
        
        # Clean up
        kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (5, 5))
        fg_mask = cv2.morphologyEx(fg_mask, cv2.MORPH_OPEN, kernel)
        fg_mask = cv2.morphologyEx(fg_mask, cv2.MORPH_CLOSE, kernel)
        
        # Find moving objects
        contours, _ = cv2.findContours(fg_mask, cv2.RETR_EXTERNAL, 
                                       cv2.CHAIN_APPROX_SIMPLE)
        
        for contour in contours:
            if cv2.contourArea(contour) < 500:  # Min area filter
                continue
            x, y, w, h = cv2.boundingRect(contour)
            cv2.rectangle(frame, (x, y), (x+w, y+h), (0, 255, 0), 2)
        
        cv2.imshow('Motion Detection', frame)
        if cv2.waitKey(30) & 0xFF == ord('q'):
            break
    
    cap.release()
    cv2.destroyAllWindows()
```

### Image Inpainting & Blending

```python
# ============================================
# Image Inpainting (Remove objects from images)
# ============================================

def inpaint_region(image, mask):
    """
    Remove an object and fill the region naturally.
    mask: white (255) where you want to remove
    """
    # Method 1: Navier-Stokes based
    result_ns = cv2.inpaint(image, mask, inpaintRadius=3, 
                            flags=cv2.INPAINT_NS)
    
    # Method 2: Fast Marching Method (usually better)
    result_fmm = cv2.inpaint(image, mask, inpaintRadius=3,
                             flags=cv2.INPAINT_TELEA)
    
    return result_fmm

# ============================================
# Seamless Cloning (blend objects naturally)
# ============================================

def seamless_clone_object(source, target, mask, center):
    """
    Clone an object from source into target with seamless blending.
    The edges are automatically smoothed — no visible seam!
    """
    # Normal clone: gradient-domain blending
    result = cv2.seamlessClone(
        source, target, mask, center,
        cv2.NORMAL_CLONE  # or MIXED_CLONE for texture transfer
    )
    return result

# ============================================
# Image Stitching (Panorama)
# ============================================

def create_panorama(images):
    """
    Create a panorama from multiple overlapping images.
    Uses OpenCV's built-in stitcher.
    """
    stitcher = cv2.Stitcher_create(cv2.Stitcher_PANORAMA)
    
    status, panorama = stitcher.stitch(images)
    
    if status == cv2.Stitcher_OK:
        print("Panorama created successfully!")
        return panorama
    else:
        error_msgs = {
            cv2.Stitcher_ERR_NEED_MORE_IMGS: "Need more images",
            cv2.Stitcher_ERR_HOMOGRAPHY_EST_FAIL: "Homography estimation failed",
            cv2.Stitcher_ERR_CAMERA_PARAMS_ADJUST_FAIL: "Camera params failed"
        }
        print(f"Stitching failed: {error_msgs.get(status, 'Unknown error')}")
        return None
```

---

## 10. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Forgetting BGR→RGB conversion | Matplotlib/PIL use RGB, OpenCV uses BGR | Always `cvtColor` when switching libraries |
| Using `img = cv2.imread()` without checking None | Path typo → silent None, crashes later | Always check `if img is None: raise` |
| Modifying image without `.copy()` | NumPy arrays are views — modifies original | Use `img.copy()` for independent copy |
| Wrong kernel size (even numbers) | Most OpenCV filters require ODD kernel size | Use 3, 5, 7, 9, etc. |
| Applying contour detection to non-binary image | `findContours` expects binary (0 or 255) | Always threshold or Canny first |
| Not converting to float for math operations | uint8 overflows: 200 + 100 = 44, not 300 | Convert to float32, process, clip back |
| Using wrong norm for feature matching | SIFT→L2, ORB→Hamming | Match norm to descriptor type |
| Ignoring `waitKey()` in display loop | Window won't update, appears frozen | Always `cv2.waitKey(1)` in loop |
| Saving video without releasing writer | File is corrupted/incomplete | Always `writer.release()` |
| Not handling variable image sizes | Processing fails on unexpected dimensions | Resize or check shape before processing |
| Using JPG for masks/binary images | JPEG compression corrupts binary values | Use PNG for masks (lossless) |
| Processing full-res when not needed | Wastes compute, slow pipeline | Resize early for speed |

---

## 11. Interview Questions

### Conceptual

**Q1: What's the difference between correlation and convolution?**
> Convolution flips the kernel 180° before sliding. Correlation doesn't flip. In practice, for symmetric kernels (Gaussian, box), they're identical. OpenCV's `filter2D` does correlation (not true convolution). For asymmetric kernels, this matters for edge direction.

**Q2: Why is Gaussian blur preferred over box blur?**
> Gaussian blur weights nearby pixels more (bell curve). Box blur weights all pixels equally, which can create visible artifacts (blocky edges). Gaussian also has the nice property of being **separable** (can decompose 2D filter into two 1D filters) and **cascadable** (two Gaussian blurs = one larger Gaussian blur).

**Q3: Explain how Canny edge detection works step by step.**
> (1) Gaussian blur to reduce noise. (2) Compute gradient magnitude and direction using Sobel. (3) Non-maximum suppression: thin edges to 1 pixel by keeping only local maxima along gradient direction. (4) Double thresholding: classify pixels as strong, weak, or non-edges. (5) Hysteresis: keep weak edges only if connected to strong edges.

**Q4: When would you use morphological operations?**
> Opening: remove small noise (white specks on black). Closing: fill small holes (black specks on white). Gradient: extract boundaries. Top hat: find bright features on dark, uneven background. Practical uses: cleaning segmentation masks, separating touching cells, extracting text from documents.

**Q5: What is RANSAC and why is it used in feature matching?**
> RANSAC (Random Sample Consensus) finds the best model despite outlier matches. Steps: (1) Randomly select minimum points for model (4 for homography). (2) Fit model. (3) Count inliers (matches consistent with model). (4) Repeat many times, keep best model. Used because feature matching always has wrong matches (outliers).

**Q6: Explain the ratio test for feature matching.**
> Lowe's ratio test: For each feature, find the 2 closest matches. If the best match is significantly closer than the second-best (ratio < 0.7-0.8), it's a good match. Intuition: A correct match should be uniquely close. If two matches are similarly close, neither is reliable.

### Coding

**Q7: Write code to count coins in an image.**
> Steps: (1) Convert to grayscale. (2) Apply Gaussian blur. (3) Use HoughCircles or threshold+contour detection. (4) Filter by area/circularity. (5) Count valid contours.

**Q8: How would you detect and correct document skew?**
> (1) Edge detection (Canny). (2) Hough line detection. (3) Find dominant angle from detected lines. (4) Rotate image by negative of that angle. Alternative: Use `minAreaRect` on text contours and correct the rotation angle.

---

## 12. Quick Reference

### Key Functions Cheat Sheet

| Operation | Function | Key Parameters |
|-----------|----------|---------------|
| Read | `cv2.imread(path, flag)` | 0=gray, 1=color, -1=unchanged |
| Write | `cv2.imwrite(path, img)` | [IMWRITE_JPEG_QUALITY, 95] |
| Resize | `cv2.resize(img, (w,h))` | interpolation flag |
| Blur | `cv2.GaussianBlur(img, (k,k), σ)` | k must be odd |
| Edge | `cv2.Canny(img, low, high)` | L2gradient=True for accuracy |
| Threshold | `cv2.threshold(img, thresh, max, type)` | +THRESH_OTSU for auto |
| Contours | `cv2.findContours(binary, mode, method)` | Returns contours, hierarchy |
| Morph | `cv2.morphologyEx(img, op, kernel)` | OPEN, CLOSE, GRADIENT |
| Features | `sift.detectAndCompute(gray, mask)` | Returns keypoints, descriptors |
| Match | `bf.match(des1, des2)` | or knnMatch for ratio test |
| Transform | `cv2.warpAffine(img, M, (w,h))` | M from getRotationMatrix2D |
| Perspective | `cv2.warpPerspective(img, H, (w,h))` | H from getPerspectiveTransform |

### Color Space Guide

| Space | Use Case | Range (OpenCV) |
|-------|----------|----------------|
| BGR | Default OpenCV format | B,G,R: 0-255 |
| RGB | Display (matplotlib, PIL) | R,G,B: 0-255 |
| HSV | Color detection/segmentation | H:0-179, S:0-255, V:0-255 |
| LAB | Perceptual color operations | L:0-255, A:0-255, B:0-255 |
| Gray | Many CV algorithms | 0-255 |
| YCrCb | Skin detection | Y:0-255, Cr:0-255, Cb:0-255 |

### Kernel Quick Reference

```python
# Blur (average)
kernel_blur = np.ones((5,5), np.float32) / 25

# Sharpen
kernel_sharp = np.array([[0,-1,0],[-1,5,-1],[0,-1,0]], np.float32)

# Edge detection (Laplacian)
kernel_edge = np.array([[0,1,0],[1,-4,1],[0,1,0]], np.float32)

# Emboss
kernel_emboss = np.array([[-2,-1,0],[-1,1,1],[0,1,2]], np.float32)

# Sobel X
kernel_sx = np.array([[-1,0,1],[-2,0,2],[-1,0,1]], np.float32)

# Sobel Y  
kernel_sy = np.array([[-1,-2,-1],[0,0,0],[1,2,1]], np.float32)
```

### Performance Tips

| Technique | Speedup | When to Use |
|-----------|---------|-------------|
| Resize early | 4-16× | When full resolution isn't needed |
| Use ROI | Variable | Process only region of interest |
| Separate thread for I/O | 1.5-2× | When I/O is the bottleneck |
| Use `cv2.UMat` (OpenCL) | 2-5× | When GPU available but no CUDA |
| Batch operations | 2-3× | Multiple similar operations |
| `cv2.sepFilter2D` | ~2× | For separable kernels |
| Skip frames | N× | Real-time when accuracy allows |
| JPEG decode with flags | 1.5× | `IMREAD_REDUCED_*` for thumbnails |

### Common Resolution Reference

| Name | Resolution | Megapixels | Aspect |
|------|-----------|------------|--------|
| VGA | 640×480 | 0.3 | 4:3 |
| HD (720p) | 1280×720 | 0.9 | 16:9 |
| Full HD (1080p) | 1920×1080 | 2.1 | 16:9 |
| 2K | 2560×1440 | 3.7 | 16:9 |
| 4K (UHD) | 3840×2160 | 8.3 | 16:9 |

---

> **Key Takeaway:** OpenCV is the workhorse of computer vision. Master the fundamentals (filtering, edges, contours, morphology, features) and you can build 80% of CV applications without deep learning. For production: always profile (I/O is usually the bottleneck), use threading, process at the lowest resolution that meets your accuracy needs, and combine OpenCV preprocessing with deep learning models.
