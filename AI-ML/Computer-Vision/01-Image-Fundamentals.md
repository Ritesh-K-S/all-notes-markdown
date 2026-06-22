# Chapter 01: Image Fundamentals

## Table of Contents
- [1.1 What is a Digital Image?](#11-what-is-a-digital-image)
- [1.2 Pixels — The Building Blocks](#12-pixels--the-building-blocks)
- [1.3 Color Spaces](#13-color-spaces)
- [1.4 Image Channels](#14-image-channels)
- [1.5 Resolution and Image Size](#15-resolution-and-image-size)
- [1.6 Image File Formats](#16-image-file-formats)
- [1.7 Image Preprocessing](#17-image-preprocessing)
- [1.8 Geometric Transformations](#18-geometric-transformations)
- [1.9 Image Filtering and Convolutions](#19-image-filtering-and-convolutions)
- [1.10 Edge Detection](#110-edge-detection)
- [1.11 Histograms and Intensity Analysis](#111-histograms-and-intensity-analysis)
- [1.12 Morphological Operations](#112-morphological-operations)
- [1.13 Image Noise and Denoising](#113-image-noise-and-denoising)
- [Quick Reference](#quick-reference)

---

## 1.1 What is a Digital Image?

### What It Is
A digital image is simply a **grid of numbers**. Imagine a spreadsheet where each cell contains a number between 0 and 255 — that's essentially what a grayscale image is. For color images, each cell has 3 numbers (Red, Green, Blue).

Think of it like a mosaic: from far away you see a picture, but up close it's just tiny colored tiles arranged in rows and columns.

### Why It Matters
Everything in computer vision starts here. Before you can detect objects, classify images, or generate art with AI — you need to understand what the computer actually "sees." Spoiler: it sees numbers, not pictures.

### How It Works

```
A 4x4 grayscale image (simplified):
┌─────┬─────┬─────┬─────┐
│ 120 │ 130 │ 125 │ 110 │  ← Row 0
├─────┼─────┼─────┼─────┤
│  80 │  90 │  85 │  75 │  ← Row 1
├─────┼─────┼─────┼─────┤
│  40 │  50 │  45 │  35 │  ← Row 2
├─────┼─────┼─────┼─────┤
│  10 │  20 │  15 │   5 │  ← Row 3
└─────┴─────┴─────┴─────┘
  Col0  Col1  Col2  Col3

0 = Pure Black
255 = Pure White
Values in between = Shades of gray
```

**Mathematical Representation:**

An image $I$ is a function:
$$I: \mathbb{Z}^2 \rightarrow \mathbb{R}^c$$

Where:
- $(x, y)$ are pixel coordinates
- $c$ is the number of channels (1 for grayscale, 3 for RGB)
- The output is the intensity value(s) at that location

For a discrete digital image with dimensions $H \times W \times C$:
$$I \in \mathbb{R}^{H \times W \times C}$$

### Code Examples

```python
import numpy as np
import cv2
from PIL import Image
import matplotlib.pyplot as plt

# === Creating an image from scratch ===
# A 100x100 grayscale image filled with gray (128)
gray_image = np.full((100, 100), 128, dtype=np.uint8)

# A 100x100 RGB image (red)
red_image = np.zeros((100, 100, 3), dtype=np.uint8)
red_image[:, :, 2] = 255  # OpenCV uses BGR, so channel 2 = Red

# === Reading an image ===
# Using OpenCV (returns BGR by default)
img_cv = cv2.imread('photo.jpg')  # Shape: (H, W, 3) in BGR

# Using PIL/Pillow (returns RGB by default)
img_pil = Image.open('photo.jpg')  # PIL Image object
img_array = np.array(img_pil)       # Convert to NumPy array (H, W, 3) in RGB

# === Key properties ===
print(f"Shape: {img_cv.shape}")       # (height, width, channels)
print(f"Data type: {img_cv.dtype}")   # uint8 (0-255)
print(f"Size in memory: {img_cv.nbytes / 1024:.1f} KB")
print(f"Min pixel: {img_cv.min()}, Max pixel: {img_cv.max()}")

# === Accessing a single pixel ===
pixel = img_cv[50, 30]     # Row 50, Column 30 → [B, G, R] values
blue_value = img_cv[50, 30, 0]  # Just the blue channel
```

> **⚠️ Important:** OpenCV uses BGR order, not RGB! This is the #1 source of color bugs for beginners.

### Common Mistakes
| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Mixing up RGB and BGR | OpenCV reads as BGR, matplotlib expects RGB | Use `cv2.cvtColor(img, cv2.COLOR_BGR2RGB)` |
| Forgetting `dtype=np.uint8` | Neural networks may output float32, display needs uint8 | Cast with `img.astype(np.uint8)` or scale first |
| Confusing (H, W) vs (W, H) | NumPy uses (rows, cols) = (H, W), but OpenCV resize uses (W, H) | Always check: `img.shape` → (H, W, C) |
| Modifying views vs copies | NumPy slicing creates views, not copies | Use `.copy()` when you need independent data |

---

## 1.2 Pixels — The Building Blocks

### What It Is
A pixel (picture element) is the **smallest addressable unit** in an image. It's like a single LEGO brick in a LEGO mosaic — individually just one color, but together they form a picture.

### Why It Matters
Understanding pixel values, their ranges, and data types is critical because:
- **Model input:** Neural networks consume pixel values
- **Preprocessing:** Normalization depends on knowing value ranges
- **Memory:** Data type determines memory usage and precision

### How It Works

**Bit Depth and Value Ranges:**

| Bit Depth | Data Type | Value Range | Use Case |
|-----------|-----------|-------------|----------|
| 1-bit | bool | 0 or 1 | Binary masks |
| 8-bit | uint8 | 0–255 | Standard images (JPEG, PNG) |
| 16-bit | uint16 | 0–65,535 | Medical imaging, HDR |
| 32-bit | float32 | 0.0–1.0 | Neural network processing |
| 64-bit | float64 | 0.0–1.0 | Scientific computing |

**Memory Calculation:**
$$\text{Memory (bytes)} = H \times W \times C \times \text{bytes\_per\_channel}$$

Example: A 1920×1080 RGB image at 8-bit:
$$1920 \times 1080 \times 3 \times 1 = 6,220,800 \text{ bytes} \approx 5.93 \text{ MB}$$

### Code Examples

```python
import numpy as np
import cv2

# === Pixel Data Types ===
img_uint8 = np.array([[200, 100], [50, 25]], dtype=np.uint8)
img_float = img_uint8.astype(np.float32) / 255.0  # Normalize to [0, 1]

print(f"uint8 range: [{img_uint8.min()}, {img_uint8.max()}]")
print(f"float range: [{img_float.min():.3f}, {img_float.max():.3f}]")

# === Overflow danger with uint8 ===
a = np.uint8(250)
b = np.uint8(10)
print(f"250 + 10 with uint8 = {np.uint8(a + b)}")  # Wraps to 4! (260 % 256)

# Safe addition using OpenCV (saturates at 255)
result = cv2.add(np.array([[250]], dtype=np.uint8), 
                 np.array([[10]], dtype=np.uint8))
print(f"cv2.add(250, 10) = {result[0,0]}")  # 255 (clamped)

# === Normalization techniques ===
img = cv2.imread('photo.jpg').astype(np.float32)

# Method 1: Simple [0, 1] normalization
img_norm1 = img / 255.0

# Method 2: ImageNet normalization (for pretrained models)
mean = np.array([0.485, 0.456, 0.406])  # RGB means
std = np.array([0.229, 0.224, 0.225])    # RGB stds
img_norm2 = (img / 255.0 - mean) / std

# Method 3: Min-Max normalization (per-image)
img_norm3 = (img - img.min()) / (img.max() - img.min())

# === Region of Interest (ROI) ===
img = cv2.imread('photo.jpg')
roi = img[100:200, 150:300]  # Crop: rows 100-199, cols 150-299
# This is a VIEW — modifying roi modifies img!
roi_copy = img[100:200, 150:300].copy()  # Independent copy
```

> **Pro Tip:** When using pretrained models (ResNet, VGG, etc.), always use the SAME normalization that was used during training. For ImageNet models, that's `mean=[0.485, 0.456, 0.406]`, `std=[0.229, 0.224, 0.225]` in RGB order.

---

## 1.3 Color Spaces

### What It Is
A color space is a **system for representing colors as numbers**. Just like you can describe a location using different coordinate systems (latitude/longitude vs street address), you can describe colors using different systems (RGB, HSV, LAB, etc.).

### Why It Matters
- **Object detection:** HSV makes it trivial to detect objects by color
- **Image segmentation:** LAB separates lightness from color → better thresholding
- **Style transfer/generation:** Different spaces capture different perceptual properties
- **Preprocessing:** Some tasks work dramatically better in specific color spaces

### How It Works

#### RGB (Red, Green, Blue)
The default for displays. Mixes light (additive color model).

```
         R=255
          /\
         /  \
        / Y  \
       /______\
G=255 /   W    \ B=255

R + G = Yellow
R + B = Magenta  
G + B = Cyan
R + G + B = White
```

#### HSV (Hue, Saturation, Value)
Intuitive for humans. Hue = color wheel angle, Saturation = purity, Value = brightness.

```
        H (Hue): 0-179 in OpenCV (0-360° mapped)
        ┌──────────────────────────────┐
        │  0°=Red  60°=Yellow  120°=Green  180°=Cyan  240°=Blue  300°=Magenta
        └──────────────────────────────┘
        
        S (Saturation): 0-255
        Gray ←────────────────→ Pure Color
        
        V (Value): 0-255
        Black ←───────────────→ Bright
```

#### LAB (Lightness, A-channel, B-channel)
Designed to be **perceptually uniform** — equal numerical changes = equal perceived changes.

- **L:** Lightness (0 = black, 100 = white)
- **a:** Green ↔ Red axis (-128 to +127)
- **b:** Blue ↔ Yellow axis (-128 to +127)

#### YCrCb (Luminance, Chrominance)
Separates brightness from color. Used in JPEG compression and skin detection.

### Code Examples

```python
import cv2
import numpy as np
import matplotlib.pyplot as plt

img_bgr = cv2.imread('photo.jpg')

# === Convert between color spaces ===
img_rgb = cv2.cvtColor(img_bgr, cv2.COLOR_BGR2RGB)
img_hsv = cv2.cvtColor(img_bgr, cv2.COLOR_BGR2HSV)
img_lab = cv2.cvtColor(img_bgr, cv2.COLOR_BGR2LAB)
img_gray = cv2.cvtColor(img_bgr, cv2.COLOR_BGR2GRAY)
img_ycrcb = cv2.cvtColor(img_bgr, cv2.COLOR_BGR2YCrCb)

# === Color-based object detection using HSV ===
# Detect blue objects
lower_blue = np.array([100, 50, 50])    # Lower HSV bound
upper_blue = np.array([130, 255, 255])  # Upper HSV bound
mask = cv2.inRange(img_hsv, lower_blue, upper_blue)  # Binary mask
blue_objects = cv2.bitwise_and(img_bgr, img_bgr, mask=mask)

# Common HSV ranges for color detection (OpenCV H: 0-179)
color_ranges = {
    'red_low':    ([0, 100, 100], [10, 255, 255]),
    'red_high':   ([160, 100, 100], [179, 255, 255]),  # Red wraps around!
    'green':      ([35, 100, 100], [85, 255, 255]),
    'blue':       ([100, 100, 100], [130, 255, 255]),
    'yellow':     ([20, 100, 100], [35, 255, 255]),
    'orange':     ([10, 100, 100], [20, 255, 255]),
}

# === Skin detection using YCrCb ===
img_ycrcb = cv2.cvtColor(img_bgr, cv2.COLOR_BGR2YCrCb)
lower_skin = np.array([0, 133, 77])
upper_skin = np.array([255, 173, 127])
skin_mask = cv2.inRange(img_ycrcb, lower_skin, upper_skin)

# === Grayscale conversion (weighted formula) ===
# OpenCV uses: Gray = 0.299*R + 0.587*G + 0.114*B
# This matches human perception (we're most sensitive to green)
gray = cv2.cvtColor(img_bgr, cv2.COLOR_BGR2GRAY)

# Manual grayscale (to understand the formula)
b, g, r = img_bgr[:,:,0], img_bgr[:,:,1], img_bgr[:,:,2]
gray_manual = (0.299 * r + 0.587 * g + 0.114 * b).astype(np.uint8)
```

> **⚠️ Warning:** Red in HSV wraps around 0°/180°. You need TWO ranges to detect red objects, then combine the masks with `cv2.bitwise_or()`.

### When to Use Which Color Space

| Color Space | Best For | Why |
|-------------|----------|-----|
| RGB | Display, neural networks | Standard input format |
| HSV | Color-based detection | Separates color from brightness |
| LAB | Color difference, segmentation | Perceptually uniform |
| Grayscale | Edge detection, features | Reduces complexity |
| YCrCb | Skin detection, compression | Separates luminance from chrominance |

---

## 1.4 Image Channels

### What It Is
Channels are the **layers** of a color image. Think of it like printing: a color printer uses separate CMYK ink layers. Similarly, a color image is made of separate layers (Red, Green, Blue) stacked together.

### Why It Matters
- **Model architecture:** Input channels determine the first layer's shape
- **Feature engineering:** Sometimes individual channels carry more info than combined
- **Alpha channel:** Transparency requires a 4th channel (RGBA)
- **Multi-spectral:** Satellite images can have 10+ channels (infrared, etc.)

### How It Works

```
RGB Image (H x W x 3):

Layer 0 (Blue):     Layer 1 (Green):    Layer 2 (Red):
┌───────────┐       ┌───────────┐       ┌───────────┐
│  B values │       │  G values │       │  R values │
│  0 - 255  │       │  0 - 255  │       │  0 - 255  │
└───────────┘       └───────────┘       └───────────┘
      │                   │                   │
      └───────────────────┼───────────────────┘
                          ↓
                   Combined Color Image
```

### Code Examples

```python
import cv2
import numpy as np

img = cv2.imread('photo.jpg')  # Shape: (H, W, 3) — BGR

# === Split channels ===
b, g, r = cv2.split(img)      # Each is (H, W) — grayscale
# Or using NumPy (faster for large images):
b, g, r = img[:,:,0], img[:,:,1], img[:,:,2]

# === Merge channels ===
merged = cv2.merge([b, g, r])  # Back to (H, W, 3)

# === Working with alpha channel (transparency) ===
img_rgba = cv2.imread('logo.png', cv2.IMREAD_UNCHANGED)  # Load with alpha
if img_rgba.shape[2] == 4:
    b, g, r, alpha = cv2.split(img_rgba)
    # alpha: 0 = fully transparent, 255 = fully opaque

# === Create custom multi-channel images ===
# Stack grayscale into 3-channel (for models expecting RGB input)
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
pseudo_rgb = cv2.merge([gray, gray, gray])  # (H, W, 3)

# === Channel manipulation for effect ===
# Keep only red channel (zero out blue and green)
red_only = img.copy()
red_only[:, :, 0] = 0  # Zero blue
red_only[:, :, 1] = 0  # Zero green

# === Multi-spectral example (satellite imagery) ===
# Simulating a 4-band image (R, G, B, NIR)
nir_band = np.random.randint(0, 256, (100, 100), dtype=np.uint8)
multi_spectral = np.dstack([img[:100, :100], nir_band])  # (100, 100, 4)
print(f"Multi-spectral shape: {multi_spectral.shape}")
```

---

## 1.5 Resolution and Image Size

### What It Is
Resolution is the **number of pixels** in an image (width × height). Higher resolution = more detail, but also more memory and compute. It's like the difference between a 4K TV and an old standard-definition TV.

### Why It Matters
- **Model input:** Most CNNs require fixed-size input (224×224, 416×416, etc.)
- **Performance:** Doubling resolution → 4× more pixels → 4× more compute
- **Quality vs Speed:** You must balance detail against processing time
- **Aspect ratio:** Careless resizing distorts objects → hurts model accuracy

### How It Works

**Common input sizes for models:**

| Model | Input Size | Why |
|-------|-----------|-----|
| ResNet/VGG | 224 × 224 | ImageNet standard |
| YOLO v5 | 640 × 640 | Balanced speed/accuracy |
| EfficientNet-B7 | 600 × 600 | Higher accuracy |
| Stable Diffusion | 512 × 512 | Memory constraint |

**Resizing strategies:**

$$\text{Scale factor} = \frac{\text{target size}}{\text{original size}}$$

### Code Examples

```python
import cv2
import numpy as np

img = cv2.imread('photo.jpg')  # e.g., (1080, 1920, 3)

# === Basic resize ===
resized = cv2.resize(img, (640, 480))  # (width, height) — NOTE: opposite of shape!

# === Resize maintaining aspect ratio ===
def resize_keep_aspect(img, target_size=640):
    """Resize longest side to target_size, maintain aspect ratio."""
    h, w = img.shape[:2]
    scale = target_size / max(h, w)
    new_w, new_h = int(w * scale), int(h * scale)
    return cv2.resize(img, (new_w, new_h))

# === Letterboxing (padding to square — used in YOLO) ===
def letterbox(img, new_shape=(640, 640), color=(114, 114, 114)):
    """Resize + pad to exact shape without distortion."""
    h, w = img.shape[:2]
    # Calculate scale
    scale = min(new_shape[0] / h, new_shape[1] / w)
    new_h, new_w = int(h * scale), int(w * scale)
    
    # Resize
    img_resized = cv2.resize(img, (new_w, new_h))
    
    # Calculate padding
    dh = (new_shape[0] - new_h) / 2  # Height padding
    dw = (new_shape[1] - new_w) / 2  # Width padding
    top, bottom = int(dh), int(dh + 0.5)
    left, right = int(dw), int(dw + 0.5)
    
    # Add padding
    img_padded = cv2.copyMakeBorder(img_resized, top, bottom, left, right,
                                     cv2.BORDER_CONSTANT, value=color)
    return img_padded, scale, (dw, dh)

# === Interpolation methods comparison ===
# cv2.INTER_NEAREST  — Fastest, blocky (good for masks)
# cv2.INTER_LINEAR   — Default, balanced (bilinear)
# cv2.INTER_CUBIC    — Smoother (bicubic, good for upscaling)
# cv2.INTER_AREA     — Best for downscaling (anti-aliasing)
# cv2.INTER_LANCZOS4 — Highest quality, slowest

# Downscaling: use INTER_AREA
small = cv2.resize(img, (320, 240), interpolation=cv2.INTER_AREA)

# Upscaling: use INTER_CUBIC or INTER_LANCZOS4
large = cv2.resize(img, (3840, 2160), interpolation=cv2.INTER_CUBIC)

# === Center crop (common in classification) ===
def center_crop(img, crop_size):
    h, w = img.shape[:2]
    start_y = (h - crop_size) // 2
    start_x = (w - crop_size) // 2
    return img[start_y:start_y+crop_size, start_x:start_x+crop_size]

cropped = center_crop(img, 224)
```

> **Pro Tip:** For object detection, NEVER just resize to square — use letterboxing. Distorting aspect ratio changes object proportions and confuses the model.

---

## 1.6 Image File Formats

### What It Is
File formats determine how pixel data is stored on disk. Like the difference between .doc and .pdf — same content, different encoding, different trade-offs.

### Why It Matters
- **JPEG:** Lossy compression → small files but artifacts (bad for masks)
- **PNG:** Lossless → perfect quality, supports transparency, larger files
- **Training data:** Wrong format can corrupt labels (JPEG for masks = disaster)
- **Deployment:** Format choice affects loading speed and bandwidth

### Comparison Table

| Format | Compression | Transparency | Best For | Avoid For |
|--------|-------------|-------------|----------|-----------|
| JPEG | Lossy | ❌ | Photos, training images | Masks, text, logos |
| PNG | Lossless | ✅ | Masks, logos, screenshots | Large photo datasets |
| TIFF | Both | ✅ | Medical/scientific imaging | Web deployment |
| BMP | None | ❌ | Raw pixel access | Storage/transfer |
| WebP | Both | ✅ | Web deployment | Compatibility issues |

### Code Examples

```python
import cv2
import numpy as np

img = cv2.imread('photo.jpg')

# === Save with quality settings ===
# JPEG: quality 0-100 (higher = better quality, larger file)
cv2.imwrite('output.jpg', img, [cv2.IMWRITE_JPEG_QUALITY, 95])

# PNG: compression 0-9 (higher = smaller file, slower write)
cv2.imwrite('output.png', img, [cv2.IMWRITE_PNG_COMPRESSION, 6])

# === Save mask correctly ===
mask = np.zeros((100, 100), dtype=np.uint8)
mask[25:75, 25:75] = 255
cv2.imwrite('mask.png', mask)   # ✅ Lossless — values stay 0 or 255
# cv2.imwrite('mask.jpg', mask) # ❌ JPEG artifacts change 0/255 to other values!

# === Check file size ===
import os
jpeg_size = os.path.getsize('output.jpg')
png_size = os.path.getsize('output.png')
print(f"JPEG: {jpeg_size/1024:.1f} KB | PNG: {png_size/1024:.1f} KB")
print(f"PNG is {png_size/jpeg_size:.1f}x larger than JPEG")
```

---

## 1.7 Image Preprocessing

### What It Is
Preprocessing transforms raw images into a format suitable for models or analysis. It's like cleaning and preparing ingredients before cooking — the same raw material, made ready for the specific recipe.

### Why It Matters
- **Model convergence:** Normalized inputs train faster and more stably
- **Consistency:** Real-world images vary wildly in size, brightness, contrast
- **Augmentation:** Artificial variety prevents overfitting
- **Pipeline correctness:** Wrong preprocessing = model sees garbage

### How It Works

**Standard preprocessing pipeline:**
```
Raw Image → Resize → Normalize → Augment → Tensor → Model
```

### Code Examples

```python
import cv2
import numpy as np

# === Complete preprocessing pipeline ===
def preprocess_for_classification(img_path, target_size=224):
    """Standard preprocessing for ImageNet-pretrained models."""
    # 1. Read image
    img = cv2.imread(img_path)
    img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)  # Convert BGR → RGB
    
    # 2. Resize (maintain aspect ratio + center crop)
    h, w = img.shape[:2]
    scale = 256 / min(h, w)  # Resize shortest side to 256
    img = cv2.resize(img, (int(w*scale), int(h*scale)))
    
    # 3. Center crop to target_size
    h, w = img.shape[:2]
    y_start = (h - target_size) // 2
    x_start = (w - target_size) // 2
    img = img[y_start:y_start+target_size, x_start:x_start+target_size]
    
    # 4. Convert to float and normalize
    img = img.astype(np.float32) / 255.0
    
    # 5. ImageNet normalization
    mean = np.array([0.485, 0.456, 0.406])
    std = np.array([0.229, 0.224, 0.225])
    img = (img - mean) / std
    
    # 6. Convert to CHW format (channels first) for PyTorch
    img = np.transpose(img, (2, 0, 1))  # (H, W, C) → (C, H, W)
    
    return img

# === Brightness and contrast adjustment ===
def adjust_brightness_contrast(img, brightness=0, contrast=0):
    """
    brightness: -127 to +127
    contrast: -127 to +127
    """
    # Formula: output = alpha * input + beta
    # alpha = contrast factor, beta = brightness offset
    alpha = 1.0 + contrast / 127.0  # 0.0 to 2.0
    beta = brightness
    return cv2.convertScaleAbs(img, alpha=alpha, beta=beta)

# === CLAHE (Contrast Limited Adaptive Histogram Equalization) ===
# Much better than global histogram equalization
def apply_clahe(img):
    """Apply CLAHE to improve local contrast."""
    if len(img.shape) == 3:
        # Convert to LAB, apply CLAHE to L channel only
        lab = cv2.cvtColor(img, cv2.COLOR_BGR2LAB)
        l, a, b = cv2.split(lab)
        clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8))
        l = clahe.apply(l)
        lab = cv2.merge([l, a, b])
        return cv2.cvtColor(lab, cv2.COLOR_LAB2BGR)
    else:
        clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8))
        return clahe.apply(img)

# === Thresholding ===
gray = cv2.cvtColor(cv2.imread('photo.jpg'), cv2.COLOR_BGR2GRAY)

# Simple threshold
_, binary = cv2.threshold(gray, 127, 255, cv2.THRESH_BINARY)

# Otsu's method (automatically finds optimal threshold)
_, otsu = cv2.threshold(gray, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)

# Adaptive threshold (handles uneven illumination)
adaptive = cv2.adaptiveThreshold(gray, 255, cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
                                  cv2.THRESH_BINARY, 11, 2)
```

> **Pro Tip:** CLAHE is almost always better than simple histogram equalization. It prevents over-amplifying noise in homogeneous regions by limiting contrast amplification.

---

## 1.8 Geometric Transformations

### What It Is
Geometric transformations change the **spatial arrangement** of pixels — moving, rotating, flipping, or warping them. Like moving furniture around a room without changing the furniture itself.

### Why It Matters
- **Data augmentation:** Random flips/rotations → more robust models
- **Image alignment:** Registration for panoramas, medical imaging
- **Perspective correction:** Document scanning, overhead view transformation
- **Object tracking:** Understanding how objects move between frames

### How It Works

**Transformation types hierarchy:**

| Type | DOF | Preserves | Examples |
|------|-----|-----------|----------|
| Translation | 2 | Everything | Shift left/right/up/down |
| Rigid (Euclidean) | 3 | Distances, angles | Rotate + translate |
| Similarity | 4 | Angles, ratios | Scale + rotate + translate |
| Affine | 6 | Parallelism | Shear + scale + rotate + translate |
| Projective (Homography) | 8 | Straight lines | Perspective transform |

**Affine transformation matrix (2×3):**

$$\begin{bmatrix} x' \\ y' \end{bmatrix} = \begin{bmatrix} a_{11} & a_{12} & t_x \\ a_{21} & a_{22} & t_y \end{bmatrix} \begin{bmatrix} x \\ y \\ 1 \end{bmatrix}$$

**Rotation matrix:**

$$R(\theta) = \begin{bmatrix} \cos\theta & -\sin\theta \\ \sin\theta & \cos\theta \end{bmatrix}$$

### Code Examples

```python
import cv2
import numpy as np

img = cv2.imread('photo.jpg')
h, w = img.shape[:2]

# === Translation (shift) ===
tx, ty = 100, 50  # Shift right 100px, down 50px
M_translate = np.float32([[1, 0, tx], [0, 1, ty]])
translated = cv2.warpAffine(img, M_translate, (w, h))

# === Rotation ===
angle = 45  # Degrees counter-clockwise
center = (w // 2, h // 2)  # Rotate around center
scale = 1.0
M_rotate = cv2.getRotationMatrix2D(center, angle, scale)
rotated = cv2.warpAffine(img, M_rotate, (w, h))

# Rotation without cropping (expand canvas)
def rotate_no_crop(img, angle):
    h, w = img.shape[:2]
    center = (w // 2, h // 2)
    M = cv2.getRotationMatrix2D(center, angle, 1.0)
    # Calculate new bounding box
    cos = abs(M[0, 0])
    sin = abs(M[0, 1])
    new_w = int(h * sin + w * cos)
    new_h = int(h * cos + w * sin)
    # Adjust translation
    M[0, 2] += (new_w - w) / 2
    M[1, 2] += (new_h - h) / 2
    return cv2.warpAffine(img, M, (new_w, new_h))

# === Flipping ===
flipped_h = cv2.flip(img, 1)   # Horizontal flip (mirror)
flipped_v = cv2.flip(img, 0)   # Vertical flip
flipped_both = cv2.flip(img, -1)  # Both

# === Affine transformation (requires 3 point pairs) ===
pts_src = np.float32([[50, 50], [200, 50], [50, 200]])
pts_dst = np.float32([[10, 100], [200, 50], [100, 250]])
M_affine = cv2.getAffineTransform(pts_src, pts_dst)
affine_result = cv2.warpAffine(img, M_affine, (w, h))

# === Perspective transformation (requires 4 point pairs) ===
# Example: correct perspective of a document
pts_src = np.float32([[56, 65], [368, 52], [28, 387], [389, 390]])
pts_dst = np.float32([[0, 0], [300, 0], [0, 400], [300, 400]])
M_perspective = cv2.getPerspectiveTransform(pts_src, pts_dst)
warped = cv2.warpPerspective(img, M_perspective, (300, 400))

# === Data augmentation with random transforms ===
def random_augment(img):
    """Apply random geometric augmentations."""
    h, w = img.shape[:2]
    
    # Random horizontal flip (50% chance)
    if np.random.random() > 0.5:
        img = cv2.flip(img, 1)
    
    # Random rotation (-15° to +15°)
    angle = np.random.uniform(-15, 15)
    M = cv2.getRotationMatrix2D((w//2, h//2), angle, 1.0)
    img = cv2.warpAffine(img, M, (w, h))
    
    # Random scale (0.8x to 1.2x)
    scale = np.random.uniform(0.8, 1.2)
    img = cv2.resize(img, None, fx=scale, fy=scale)
    
    return img
```

---

## 1.9 Image Filtering and Convolutions

### What It Is
Filtering is **sliding a small matrix (kernel) over an image** and computing a weighted sum at each position. It's like examining a photo through a tiny window, doing math on what you see, and writing the result in a new image.

This is THE fundamental operation in CNNs — understanding it here means you understand the core of deep learning for vision.

### Why It Matters
- **CNNs:** Convolutional layers ARE this operation (with learned kernels)
- **Blurring:** Noise reduction before processing
- **Sharpening:** Enhance details for better feature extraction
- **Edge detection:** Foundation of many classical CV algorithms

### How It Works

**Convolution operation:**

$$G(x,y) = \sum_{i=-k}^{k} \sum_{j=-k}^{k} K(i,j) \cdot I(x+i, y+j)$$

Where:
- $I$ = input image
- $K$ = kernel (filter)
- $G$ = output image
- $k$ = kernel radius (for 3×3 kernel, $k=1$)

```
Image patch:        Kernel:           Convolution:
┌───┬───┬───┐     ┌───┬───┬───┐    
│ 1 │ 2 │ 3 │     │ 0 │-1 │ 0 │    Output pixel = 
├───┼───┼───┤  *  ├───┼───┼───┤    (1×0 + 2×-1 + 3×0 +
│ 4 │ 5 │ 6 │     │-1 │ 5 │-1 │     4×-1 + 5×5 + 6×-1 +
├───┼───┼───┤     ├───┼───┼───┤     7×0 + 8×-1 + 9×0)
│ 7 │ 8 │ 9 │     │ 0 │-1 │ 0 │    = 0-2+0-4+25-6+0-8+0 = 5
└───┴───┴───┘     └───┴───┴───┘
```

**Common kernels:**

```
Blur (Box):         Sharpen:            Edge (Laplacian):
┌────┬────┬────┐   ┌────┬────┬────┐   ┌────┬────┬────┐
│1/9 │1/9 │1/9 │   │ 0  │ -1 │  0 │   │ 0  │  1 │  0 │
├────┼────┼────┤   ├────┼────┼────┤   ├────┼────┼────┤
│1/9 │1/9 │1/9 │   │-1  │  5 │ -1 │   │ 1  │ -4 │  1 │
├────┼────┼────┤   ├────┼────┼────┤   ├────┼────┼────┤
│1/9 │1/9 │1/9 │   │ 0  │ -1 │  0 │   │ 0  │  1 │  0 │
└────┴────┴────┘   └────┴────┴────┘   └────┴────┴────┘
```

### Code Examples

```python
import cv2
import numpy as np

img = cv2.imread('photo.jpg')
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

# === Box blur (averaging) ===
blur_box = cv2.blur(img, (5, 5))  # 5x5 averaging kernel

# === Gaussian blur (weighted, center has more influence) ===
# sigma controls spread: larger σ = more blur
blur_gauss = cv2.GaussianBlur(img, (5, 5), sigmaX=1.5)

# === Median blur (best for salt-and-pepper noise) ===
blur_median = cv2.medianBlur(img, 5)  # kernel size must be odd

# === Bilateral filter (blur while preserving edges) ===
# d=diameter, sigmaColor=color similarity, sigmaSpace=spatial proximity
blur_bilateral = cv2.bilateralFilter(img, d=9, sigmaColor=75, sigmaSpace=75)

# === Custom kernel convolution ===
# Sharpening kernel
kernel_sharpen = np.array([[ 0, -1,  0],
                           [-1,  5, -1],
                           [ 0, -1,  0]], dtype=np.float32)
sharpened = cv2.filter2D(img, -1, kernel_sharpen)

# Emboss kernel
kernel_emboss = np.array([[-2, -1, 0],
                          [-1,  1, 1],
                          [ 0,  1, 2]], dtype=np.float32)
embossed = cv2.filter2D(gray, -1, kernel_emboss)

# === Understanding padding ===
# Without padding: output is smaller (valid convolution)
# With padding: output same size (same convolution)
# OpenCV default: reflects border pixels

# Explicit padding example
padded = cv2.copyMakeBorder(gray, 1, 1, 1, 1, cv2.BORDER_REFLECT)
# Now a 3x3 convolution gives same-size output

# === Separable filters (much faster for large kernels) ===
# A 2D Gaussian can be decomposed into two 1D filters
# cv2.GaussianBlur does this automatically
# Manual example:
kernel_1d = cv2.getGaussianKernel(5, 1.5)  # Column vector
# 2D kernel = outer product of 1D kernels
kernel_2d = kernel_1d @ kernel_1d.T
# But applying two 1D filters is O(n) vs O(n²) for the 2D version
```

> **Pro Tip:** `cv2.GaussianBlur` is implemented as a separable filter internally, making it much faster than applying the equivalent 2D kernel with `cv2.filter2D`. For a $k \times k$ kernel, separable = $2k$ operations per pixel vs $k^2$ for non-separable.

---

## 1.10 Edge Detection

### What It Is
Edge detection finds boundaries where pixel intensity changes sharply — like tracing the outline of objects in a coloring book. Mathematically, edges are where the **gradient** (rate of change) of intensity is high.

### Why It Matters
- **Object boundaries:** Edges define where objects are
- **Feature extraction:** Many classical CV methods start with edges
- **Segmentation:** Edges help separate regions
- **Understanding CNNs:** Early CNN layers learn edge-like filters

### How It Works

**The Gradient:**
$$\nabla I = \begin{bmatrix} \frac{\partial I}{\partial x} \\ \frac{\partial I}{\partial y} \end{bmatrix}$$

**Gradient magnitude (edge strength):**
$$|\nabla I| = \sqrt{\left(\frac{\partial I}{\partial x}\right)^2 + \left(\frac{\partial I}{\partial y}\right)^2}$$

**Gradient direction:**
$$\theta = \arctan\left(\frac{\partial I / \partial y}{\partial I / \partial x}\right)$$

**Sobel operators:**
$$G_x = \begin{bmatrix} -1 & 0 & 1 \\ -2 & 0 & 2 \\ -1 & 0 & 1 \end{bmatrix} \quad G_y = \begin{bmatrix} -1 & -2 & -1 \\ 0 & 0 & 0 \\ 1 & 2 & 1 \end{bmatrix}$$

**Canny edge detection pipeline:**
```
Input → Gaussian Blur → Gradient (Sobel) → Non-Max Suppression → Hysteresis Thresholding → Edges
```

### Code Examples

```python
import cv2
import numpy as np

img = cv2.imread('photo.jpg')
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

# === Sobel edge detection ===
# Detect horizontal edges (gradient in Y direction)
sobel_y = cv2.Sobel(gray, cv2.CV_64F, 0, 1, ksize=3)
# Detect vertical edges (gradient in X direction)
sobel_x = cv2.Sobel(gray, cv2.CV_64F, 1, 0, ksize=3)
# Combined magnitude
sobel_mag = np.sqrt(sobel_x**2 + sobel_y**2)
sobel_mag = np.uint8(sobel_mag / sobel_mag.max() * 255)

# === Canny edge detection (the gold standard) ===
# threshold1 (low), threshold2 (high) for hysteresis
edges = cv2.Canny(gray, threshold1=50, threshold2=150)

# Auto-threshold Canny using median
def auto_canny(image, sigma=0.33):
    """Automatically determine Canny thresholds from median."""
    median = np.median(image)
    lower = int(max(0, (1.0 - sigma) * median))
    upper = int(min(255, (1.0 + sigma) * median))
    return cv2.Canny(image, lower, upper)

edges_auto = auto_canny(gray)

# === Laplacian (second derivative — detects edges in all directions) ===
laplacian = cv2.Laplacian(gray, cv2.CV_64F)
laplacian = np.uint8(np.abs(laplacian))

# === Practical: Edge-based contour detection ===
edges = cv2.Canny(gray, 50, 150)
contours, hierarchy = cv2.findContours(edges, cv2.RETR_EXTERNAL, 
                                        cv2.CHAIN_APPROX_SIMPLE)
# Draw contours on original image
result = img.copy()
cv2.drawContours(result, contours, -1, (0, 255, 0), 2)
```

### Edge Detection Methods Comparison

| Method | Strengths | Weaknesses | Use When |
|--------|-----------|------------|----------|
| Sobel | Directional, smooth | Thick edges | Need gradient direction |
| Canny | Thin edges, less noise | Two thresholds to tune | General edge detection |
| Laplacian | All directions at once | Sensitive to noise | Zero-crossing detection |
| Scharr | More accurate than Sobel | Same limitations | Need precise gradients |

---

## 1.11 Histograms and Intensity Analysis

### What It Is
A histogram shows the **distribution of pixel intensities** in an image — how many pixels have each brightness level. Like a bar chart showing how many students got each grade in a class.

### Why It Matters
- **Exposure analysis:** Is the image too dark/bright?
- **Contrast enhancement:** Histogram equalization dramatically improves visibility
- **Thresholding:** Histogram shape tells you where to set thresholds
- **Image comparison:** Histograms can measure image similarity

### Code Examples

```python
import cv2
import numpy as np
import matplotlib.pyplot as plt

img = cv2.imread('photo.jpg')
gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

# === Calculate histogram ===
hist = cv2.calcHist([gray], [0], None, [256], [0, 256])
# Parameters: [image], [channel], mask, [bins], [range]

# Color histogram (per channel)
colors = ('b', 'g', 'r')
for i, color in enumerate(colors):
    hist = cv2.calcHist([img], [i], None, [256], [0, 256])
    plt.plot(hist, color=color)

# === Histogram equalization (global) ===
equalized = cv2.equalizeHist(gray)

# === CLAHE (much better — local adaptive) ===
clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8))
clahe_result = clahe.apply(gray)

# === Histogram comparison (image similarity) ===
hist1 = cv2.calcHist([img1], [0, 1, 2], None, [8, 8, 8], [0, 256, 0, 256, 0, 256])
hist2 = cv2.calcHist([img2], [0, 1, 2], None, [8, 8, 8], [0, 256, 0, 256, 0, 256])
cv2.normalize(hist1, hist1)
cv2.normalize(hist2, hist2)

# Methods: CORREL (1=identical), CHI_SQR (0=identical), INTERSECT, BHATTACHARYYA
similarity = cv2.compareHist(hist1, hist2, cv2.HISTCMP_CORREL)
print(f"Similarity: {similarity:.3f}")  # 1.0 = identical

# === Back projection (find regions matching a color histogram) ===
# Useful for tracking objects by color
roi_hist = cv2.calcHist([roi_hsv], [0, 1], None, [180, 256], [0, 180, 0, 256])
cv2.normalize(roi_hist, roi_hist, 0, 255, cv2.NORM_MINMAX)
back_proj = cv2.calcBackProject([target_hsv], [0, 1], roi_hist, [0, 180, 0, 256], 1)
```

---

## 1.12 Morphological Operations

### What It Is
Morphological operations are **shape-based operations** on binary/grayscale images using a structuring element (small shape template). Like using cookie cutters on dough — you can erode edges, expand regions, or clean up shapes.

### Why It Matters
- **Mask cleanup:** Remove noise from segmentation masks
- **Object separation:** Split touching objects (watershed prep)
- **Feature extraction:** Detect lines, corners, specific shapes
- **Post-processing:** Clean up model outputs before use

### How It Works

```
Erosion (shrinks white regions):     Dilation (expands white regions):
Original:    After Erosion:          Original:    After Dilation:
██████████   ░░████████░░            ██████████   ████████████
██████████   ░░████████░░            ██░░░░░░██   ████████████
██░░░░░░██   ░░░░░░░░░░░░            ██░░░░░░██   ████████████
██░░░░░░██   ░░░░░░░░░░░░            ██████████   ████████████
██████████   ░░████████░░

Opening = Erosion → Dilation (removes small white noise)
Closing = Dilation → Erosion (fills small black holes)
```

### Code Examples

```python
import cv2
import numpy as np

# Create a binary mask (simulating a segmentation output)
mask = np.zeros((200, 200), dtype=np.uint8)
cv2.rectangle(mask, (50, 50), (150, 150), 255, -1)
# Add some noise
noise = np.random.randint(0, 2, mask.shape).astype(np.uint8) * 255
mask_noisy = cv2.bitwise_or(mask, noise * (np.random.random(mask.shape) > 0.95).astype(np.uint8) * 255)

# === Structuring elements ===
kernel_rect = cv2.getStructuringElement(cv2.MORPH_RECT, (5, 5))
kernel_ellipse = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (5, 5))
kernel_cross = cv2.getStructuringElement(cv2.MORPH_CROSS, (5, 5))

# === Basic operations ===
eroded = cv2.erode(mask_noisy, kernel_rect, iterations=1)
dilated = cv2.dilate(mask_noisy, kernel_rect, iterations=1)

# === Compound operations ===
opened = cv2.morphologyEx(mask_noisy, cv2.MORPH_OPEN, kernel_rect)    # Remove noise
closed = cv2.morphologyEx(mask_noisy, cv2.MORPH_CLOSE, kernel_rect)   # Fill holes
gradient = cv2.morphologyEx(mask, cv2.MORPH_GRADIENT, kernel_rect)     # Edge/outline
tophat = cv2.morphologyEx(mask, cv2.MORPH_TOPHAT, kernel_rect)        # Bright spots
blackhat = cv2.morphologyEx(mask, cv2.MORPH_BLACKHAT, kernel_rect)    # Dark spots

# === Practical: Clean segmentation mask ===
def clean_mask(mask, kernel_size=5):
    """Standard mask cleanup pipeline."""
    kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (kernel_size, kernel_size))
    # Remove small noise (opening)
    cleaned = cv2.morphologyEx(mask, cv2.MORPH_OPEN, kernel, iterations=2)
    # Fill small holes (closing)
    cleaned = cv2.morphologyEx(cleaned, cv2.MORPH_CLOSE, kernel, iterations=2)
    return cleaned

# === Practical: Separate touching objects ===
def separate_objects(binary_mask):
    """Use distance transform + watershed to split touching objects."""
    # Distance transform
    dist = cv2.distanceTransform(binary_mask, cv2.DIST_L2, 5)
    # Threshold to find sure foreground
    _, sure_fg = cv2.threshold(dist, 0.5 * dist.max(), 255, 0)
    sure_fg = sure_fg.astype(np.uint8)
    # Sure background (dilated mask)
    sure_bg = cv2.dilate(binary_mask, None, iterations=3)
    # Unknown region
    unknown = cv2.subtract(sure_bg, sure_fg)
    return sure_fg, sure_bg, unknown
```

---

## 1.13 Image Noise and Denoising

### What It Is
Noise is **random variation in pixel values** that wasn't in the original scene. Like static on an old TV or grain in a photo taken in low light. Denoising removes this while trying to preserve actual details.

### Why It Matters
- **Model accuracy:** Noisy inputs confuse classifiers
- **Preprocessing:** Denoising before edge detection prevents false edges
- **Medical imaging:** Noise in X-rays/MRIs can mask critical details
- **Low-light photography:** Essential for night-time camera systems

### Types of Noise

| Type | Appearance | Cause | Best Removal |
|------|-----------|-------|--------------|
| Gaussian | Smooth grain | Sensor electronics | Gaussian blur, NLM |
| Salt & Pepper | Random black/white dots | Bit errors, dead pixels | Median filter |
| Poisson (Shot) | Intensity-dependent | Low photon count | Specialized filters |
| Speckle | Multiplicative grain | Radar/ultrasound | Lee filter |

### Code Examples

```python
import cv2
import numpy as np

img = cv2.imread('photo.jpg')

# === Add synthetic noise (for testing) ===
def add_gaussian_noise(img, mean=0, std=25):
    noise = np.random.normal(mean, std, img.shape).astype(np.float32)
    noisy = np.clip(img.astype(np.float32) + noise, 0, 255).astype(np.uint8)
    return noisy

def add_salt_pepper(img, amount=0.02):
    noisy = img.copy()
    # Salt (white pixels)
    num_salt = int(amount * img.size * 0.5)
    coords = tuple(np.random.randint(0, d, num_salt) for d in img.shape[:2])
    noisy[coords] = 255
    # Pepper (black pixels)
    num_pepper = int(amount * img.size * 0.5)
    coords = tuple(np.random.randint(0, d, num_pepper) for d in img.shape[:2])
    noisy[coords] = 0
    return noisy

# === Denoising methods ===
noisy = add_gaussian_noise(img, std=30)

# Method 1: Gaussian blur (simple but loses detail)
denoised_gauss = cv2.GaussianBlur(noisy, (5, 5), 1.5)

# Method 2: Median filter (best for salt & pepper)
denoised_median = cv2.medianBlur(noisy, 5)

# Method 3: Bilateral filter (preserves edges)
denoised_bilateral = cv2.bilateralFilter(noisy, 9, 75, 75)

# Method 4: Non-Local Means (BEST quality, but slow)
# h = filter strength (higher = more denoising)
denoised_nlm = cv2.fastNlMeansDenoisingColored(noisy, None, h=10, 
                                                 hForColorComponents=10,
                                                 templateWindowSize=7, 
                                                 searchWindowSize=21)

# === Measure denoising quality ===
def psnr(original, denoised):
    """Peak Signal-to-Noise Ratio (higher = better, >30 dB is good)."""
    mse = np.mean((original.astype(float) - denoised.astype(float)) ** 2)
    if mse == 0:
        return float('inf')
    return 10 * np.log10(255**2 / mse)

print(f"PSNR (Gaussian blur): {psnr(img, denoised_gauss):.2f} dB")
print(f"PSNR (NLM): {psnr(img, denoised_nlm):.2f} dB")
```

> **Pro Tip:** In production, `cv2.fastNlMeansDenoisingColored` gives the best results but is slow. For real-time applications, use bilateral filter or a learned denoising model (like DnCNN).

---

## Interview Questions

1. **Q: What is a digital image in computer terms?**
   - A: A multi-dimensional array (tensor) of numbers. Grayscale = 2D (H×W), Color = 3D (H×W×C). Each number represents intensity at that spatial location.

2. **Q: Why does OpenCV use BGR instead of RGB?**
   - A: Historical reason from early camera hardware. The original developers chose BGR to match the byte order of certain camera hardware in the 1990s. It stuck for backward compatibility.

3. **Q: When would you use HSV over RGB?**
   - A: Color-based detection/segmentation. HSV separates color (Hue) from lighting (Value), so you can detect "red objects" regardless of shadows or brightness.

4. **Q: Explain the difference between correlation and convolution.**
   - A: Convolution flips the kernel 180° before sliding; correlation doesn't. In deep learning, "convolution" layers actually perform correlation (no flipping). For symmetric kernels, they're identical.

5. **Q: Why is Gaussian blur preferred over box blur?**
   - A: Gaussian gives more weight to the center pixel (natural, smooth blur). Box blur weights all equally (can create artifacts). Gaussian is also separable (faster for large kernels).

6. **Q: What normalization would you use for a ResNet pretrained on ImageNet?**
   - A: Divide by 255, then subtract mean `[0.485, 0.456, 0.406]` and divide by std `[0.229, 0.224, 0.225]` (RGB order). Using different normalization degrades performance.

7. **Q: How does CLAHE differ from histogram equalization?**
   - A: CLAHE applies equalization locally (in tiles) and clips the histogram to limit over-amplification. This handles images with varying illumination much better than global equalization.

8. **Q: What happens if you save a binary mask as JPEG?**
   - A: JPEG compression adds artifacts around edges, turning clean 0/255 values into intermediate values. This corrupts the mask. Always use PNG for masks.

---

## Quick Reference

| Operation | OpenCV Function | Key Parameters |
|-----------|----------------|----------------|
| Read image | `cv2.imread(path)` | Returns BGR, uint8 |
| Write image | `cv2.imwrite(path, img)` | Format from extension |
| Color convert | `cv2.cvtColor(img, code)` | `COLOR_BGR2RGB`, `COLOR_BGR2GRAY`, etc. |
| Resize | `cv2.resize(img, (W, H))` | Note: (width, height) order! |
| Gaussian blur | `cv2.GaussianBlur(img, (k,k), σ)` | k must be odd |
| Canny edges | `cv2.Canny(img, low, high)` | Typical ratio: high = 2-3× low |
| Threshold | `cv2.threshold(img, thresh, max, type)` | Use OTSU for auto |
| Erode/Dilate | `cv2.erode(img, kernel)` | iterations parameter |
| Find contours | `cv2.findContours(binary, mode, method)` | Returns list of point arrays |
| Rotate | `cv2.getRotationMatrix2D(center, angle, scale)` | + `warpAffine` |
| Normalize | `img.astype(np.float32) / 255.0` | Then apply mean/std |

### Memory Cheat Sheet

| Image Type | Formula | Example (1080p) |
|------------|---------|-----------------|
| Grayscale 8-bit | H × W × 1 | ~2 MB |
| RGB 8-bit | H × W × 3 | ~6 MB |
| RGB float32 | H × W × 3 × 4 | ~24 MB |
| Batch of 32 RGB | 32 × H × W × 3 × 4 | ~760 MB |

### Interpolation Cheat Sheet

| Method | Speed | Quality | Use For |
|--------|-------|---------|---------|
| NEAREST | ★★★★★ | ★ | Masks, labels |
| LINEAR | ★★★★ | ★★★ | Default |
| AREA | ★★★ | ★★★★ | Downscaling |
| CUBIC | ★★ | ★★★★ | Upscaling |
| LANCZOS4 | ★ | ★★★★★ | Maximum quality |
