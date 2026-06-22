# Chapter 02: Matplotlib Advanced

## Table of Contents
- [2.1 Advanced Customization & Styling](#21-advanced-customization--styling)
- [2.2 Annotations and Text](#22-annotations-and-text)
- [2.3 Color Maps and Color Bars](#23-color-maps-and-color-bars)
- [2.4 3D Plots](#24-3d-plots)
- [2.5 Animations](#25-animations)
- [2.6 Built-in Styles and Themes](#26-built-in-styles-and-themes)
- [2.7 Advanced Plot Types](#27-advanced-plot-types)
- [2.8 Performance & Production Tips](#28-performance--production-tips)
- [2.9 Common Mistakes](#29-common-mistakes)
- [2.10 Interview Questions](#210-interview-questions)
- [2.11 Quick Reference](#211-quick-reference)

---

## 2.1 Advanced Customization & Styling

### What It Is
Beyond basic labels and colors, advanced customization gives you pixel-level control over every element — ticks, spines, fonts, spacing, and even mathematical typesetting. It's the difference between a student's homework chart and a New York Times data graphic.

### Why It Matters
- **Publication requirements** demand specific fonts, sizes, and formats
- **Branding** requires consistent colors and styles
- **Accessibility** needs proper contrast and sizing for all audiences
- **Storytelling** — good design draws attention to the right data

### rcParams: Global Configuration

```python
import matplotlib.pyplot as plt
import matplotlib as mpl
import numpy as np

# rcParams controls ALL default settings globally
# Set once at the top of your script/notebook

plt.rcParams.update({
    'figure.figsize': (10, 6),       # Default figure size
    'figure.dpi': 100,               # Screen resolution
    'savefig.dpi': 300,              # Save resolution
    'font.size': 12,                 # Base font size
    'font.family': 'sans-serif',     # Font family
    'axes.titlesize': 14,            # Title font size
    'axes.labelsize': 12,            # Axis label size
    'xtick.labelsize': 10,           # X tick label size
    'ytick.labelsize': 10,           # Y tick label size
    'legend.fontsize': 11,           # Legend font size
    'lines.linewidth': 2,            # Default line width
    'axes.grid': True,               # Show grid by default
    'grid.alpha': 0.3,               # Grid transparency
    'axes.spines.top': False,        # Hide top spine
    'axes.spines.right': False,      # Hide right spine
})
```

> **Pro Tip:** Create a config file at `~/.config/matplotlib/matplotlibrc` for permanent defaults across all projects.

### Custom Tick Formatting

```python
import matplotlib.pyplot as plt
import matplotlib.ticker as ticker
import numpy as np

fig, axes = plt.subplots(2, 2, figsize=(12, 8))

x = np.linspace(0, 1000000, 100)
y = np.random.randn(100).cumsum()

# === Thousands separator ===
axes[0, 0].plot(x, y)
axes[0, 0].xaxis.set_major_formatter(ticker.FuncFormatter(
    lambda x, p: f'{x:,.0f}'))  # 1,000,000 format
axes[0, 0].set_title('Thousands Separator')

# === Percentage format ===
axes[0, 1].plot(np.linspace(0, 1, 100), np.random.rand(100))
axes[0, 1].yaxis.set_major_formatter(ticker.PercentFormatter(xmax=1))
axes[0, 1].set_title('Percentage Format')

# === Dollar format ===
revenue = np.random.uniform(1000, 5000, 100)
axes[1, 0].bar(range(10), revenue[:10])
axes[1, 0].yaxis.set_major_formatter(ticker.FuncFormatter(
    lambda x, p: f'${x:,.0f}'))
axes[1, 0].set_title('Dollar Format')

# === Scientific notation control ===
axes[1, 1].plot(x * 1e-6, y)
axes[1, 1].ticklabel_format(style='scientific', axis='x', scilimits=(0, 0))
axes[1, 1].set_title('Scientific Notation')

plt.tight_layout()
plt.show()
```

### Custom Fonts and LaTeX

```python
import matplotlib.pyplot as plt
import numpy as np

# Enable LaTeX-style rendering (doesn't require LaTeX installation)
plt.rcParams['text.usetex'] = False  # Set True if LaTeX is installed
plt.rcParams['mathtext.fontset'] = 'cm'  # Computer Modern (LaTeX look)

fig, ax = plt.subplots(figsize=(8, 5))

x = np.linspace(-2*np.pi, 2*np.pi, 200)
ax.plot(x, np.sin(x), label=r'$f(x) = \sin(x)$')
ax.plot(x, np.cos(x), label=r'$g(x) = \cos(x)$')

# LaTeX in title (use r-string prefix for raw string)
ax.set_title(r'Trigonometric Functions: $\sin(x)$ and $\cos(x)$', fontsize=14)
ax.set_xlabel(r'$\theta$ (radians)')
ax.set_ylabel(r'Amplitude')

# Add equation as text
ax.text(0.05, 0.95, r'$e^{i\pi} + 1 = 0$', transform=ax.transAxes,
        fontsize=16, verticalalignment='top',
        bbox=dict(boxstyle='round', facecolor='lightyellow'))

ax.legend(fontsize=12)
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

### Twin Axes (Two Y-Axes)

```python
import matplotlib.pyplot as plt
import numpy as np

fig, ax1 = plt.subplots(figsize=(10, 6))

# First y-axis (left)
months = np.arange(1, 13)
temperature = [5, 7, 12, 18, 23, 28, 31, 30, 25, 18, 11, 6]
color1 = '#E74C3C'
ax1.plot(months, temperature, color=color1, linewidth=2.5, marker='o', label='Temperature')
ax1.set_xlabel('Month', fontsize=12)
ax1.set_ylabel('Temperature (°C)', color=color1, fontsize=12)
ax1.tick_params(axis='y', labelcolor=color1)

# Second y-axis (right) — shares same x-axis
ax2 = ax1.twinx()
rainfall = [80, 60, 55, 45, 40, 30, 20, 25, 45, 70, 85, 90]
color2 = '#3498DB'
ax2.bar(months, rainfall, alpha=0.3, color=color2, label='Rainfall')
ax2.set_ylabel('Rainfall (mm)', color=color2, fontsize=12)
ax2.tick_params(axis='y', labelcolor=color2)

# Combine legends from both axes
lines1, labels1 = ax1.get_legend_handles_labels()
lines2, labels2 = ax2.get_legend_handles_labels()
ax1.legend(lines1 + lines2, labels1 + labels2, loc='upper left')

ax1.set_title('Temperature vs Rainfall by Month', fontsize=14)
plt.tight_layout()
plt.show()
```

> **Warning:** Twin axes can be misleading! The two scales are independent, so viewers might think there's a relationship that doesn't exist. Always add clear labels and consider if a single-axis plot would be clearer.

---

## 2.2 Annotations and Text

### What It Is
Annotations add explanatory text, arrows, and markers to highlight specific data points or regions. They turn a plot from "here's some data" into "here's the story in the data."

### Why It Matters
- Guides the reader's eye to key insights
- Explains anomalies or important events
- Makes plots self-explanatory (no external caption needed)
- Used extensively in reports, papers, and presentations

### Code Examples

```python
import matplotlib.pyplot as plt
import numpy as np

# === ANNOTATING SPECIFIC POINTS ===
x = np.linspace(0, 10, 100)
y = np.sin(x) * np.exp(-x/10)

fig, ax = plt.subplots(figsize=(10, 6))
ax.plot(x, y, 'b-', linewidth=2)

# Find and annotate the maximum
max_idx = np.argmax(y)
max_x, max_y = x[max_idx], y[max_idx]

ax.annotate(f'Peak: ({max_x:.2f}, {max_y:.2f})',
            xy=(max_x, max_y),          # Point to annotate (arrow tip)
            xytext=(max_x + 2, max_y + 0.2),  # Text position
            fontsize=11,
            arrowprops=dict(
                arrowstyle='->',         # Arrow style
                color='red',
                lw=2,                    # Arrow line width
                connectionstyle='arc3,rad=0.3'  # Curved arrow
            ),
            bbox=dict(boxstyle='round,pad=0.3', facecolor='yellow', alpha=0.7))

# Annotate a region
ax.axvspan(4, 6, alpha=0.1, color='green', label='Region of Interest')
ax.annotate('Decay region', xy=(5, 0.2), fontsize=11, ha='center',
            color='green', fontweight='bold')

ax.set_title('Damped Sine Wave with Annotations')
ax.legend()
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

```python
# === ARROW STYLES SHOWCASE ===
fig, ax = plt.subplots(figsize=(10, 8))
ax.set_xlim(0, 10)
ax.set_ylim(0, 10)

arrow_styles = ['->', '-[', '|-|', '-|>', '<->', '<|-|>', 
                'fancy', 'simple', 'wedge']

for i, style in enumerate(arrow_styles):
    y = 9 - i
    try:
        ax.annotate(f'arrowstyle = "{style}"',
                    xy=(7, y), xytext=(2, y),
                    fontsize=10, va='center',
                    arrowprops=dict(arrowstyle=style, color='navy', lw=2))
    except:
        ax.text(2, y, f'{style} (requires specific params)', fontsize=10)

ax.set_title('Matplotlib Arrow Styles', fontsize=14)
ax.axis('off')
plt.tight_layout()
plt.show()
```

```python
# === REAL-WORLD EXAMPLE: Stock Price with Events ===
np.random.seed(42)
days = np.arange(0, 252)  # Trading days in a year
price = 100 + np.cumsum(np.random.randn(252) * 1.5)  # Random walk

fig, ax = plt.subplots(figsize=(12, 6))
ax.plot(days, price, 'b-', linewidth=1.5)

# Mark significant events
events = {
    50: ('Earnings Beat', 'green'),
    120: ('Product Launch', 'blue'),
    180: ('Market Crash', 'red'),
    220: ('Recovery Begins', 'orange')
}

for day, (event, color) in events.items():
    ax.axvline(x=day, color=color, linestyle='--', alpha=0.5)
    ax.annotate(event,
                xy=(day, price[day]),
                xytext=(day + 10, price[day] + 5),
                fontsize=9, color=color, fontweight='bold',
                arrowprops=dict(arrowstyle='->', color=color))

# Add shaded region
ax.fill_between(days[170:200], price[170:200], alpha=0.2, color='red',
                label='Crash Period')

ax.set_xlabel('Trading Day')
ax.set_ylabel('Stock Price ($)')
ax.set_title('Stock Price with Annotated Events')
ax.legend()
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

### Annotation Coordinate Systems

| System | `xycoords=` | Meaning |
|--------|------------|---------|
| Data coords | `'data'` (default) | Uses x,y data values |
| Axes fraction | `'axes fraction'` | (0,0) = bottom-left, (1,1) = top-right of axes |
| Figure fraction | `'figure fraction'` | Relative to entire figure |
| Axes pixels | `'axes pixels'` | Pixel offset from axes origin |

---

## 2.3 Color Maps and Color Bars

### What It Is
Colormaps map numerical values to colors. Instead of just "red" or "blue," a colormap gives you a gradient that represents a range (like a temperature scale from cold-blue to hot-red).

### Why It Matters
- Encodes a **3rd dimension** in 2D plots (scatter color, heatmaps)
- Critical for scientific visualization (MRI scans, terrain maps, weather)
- Poor colormap choice can **mislead** or be **inaccessible** to colorblind viewers

### Types of Colormaps

```
SEQUENTIAL          DIVERGING           QUALITATIVE
(low → high)       (neg ← 0 → pos)    (categories)

░▒▓█████           ████░░░░████        ■ ■ ■ ■ ■
light → dark       blue → white → red   distinct colors
                                        
Use: heatmaps,     Use: correlations,  Use: categories,
density, counts    anomalies, +/-      groups, labels

Examples:          Examples:            Examples:
viridis, plasma    coolwarm, RdBu      Set2, tab10, Paired
```

### Code Examples

```python
import matplotlib.pyplot as plt
import numpy as np

# === HEATMAP WITH COLORBAR ===
np.random.seed(42)
data = np.random.randn(10, 12)  # 10x12 matrix

fig, ax = plt.subplots(figsize=(10, 6))
im = ax.imshow(data, 
               cmap='coolwarm',     # Diverging colormap
               aspect='auto',        # Fill available space
               interpolation='nearest')  # No smoothing

# Add colorbar
cbar = fig.colorbar(im, ax=ax, shrink=0.8, pad=0.02)
cbar.set_label('Z-Score', fontsize=11)

# Labels
ax.set_xticks(range(12))
ax.set_xticklabels(['Jan','Feb','Mar','Apr','May','Jun',
                    'Jul','Aug','Sep','Oct','Nov','Dec'])
ax.set_yticks(range(10))
ax.set_yticklabels([f'Metric {i+1}' for i in range(10)])
ax.set_title('Monthly Metrics Heatmap', fontsize=14)

plt.tight_layout()
plt.show()
```

```python
# === ADDING VALUES TO HEATMAP CELLS ===
np.random.seed(42)
data = np.random.randint(0, 100, (5, 5))

fig, ax = plt.subplots(figsize=(7, 6))
im = ax.imshow(data, cmap='YlOrRd')

# Add text annotations in each cell
for i in range(data.shape[0]):
    for j in range(data.shape[1]):
        # Choose text color based on background brightness
        text_color = 'white' if data[i, j] > 60 else 'black'
        ax.text(j, i, f'{data[i, j]}', ha='center', va='center',
                fontsize=12, color=text_color, fontweight='bold')

cbar = fig.colorbar(im, ax=ax)
ax.set_title('Heatmap with Values')
plt.tight_layout()
plt.show()
```

```python
# === CUSTOM COLORMAP ===
from matplotlib.colors import LinearSegmentedColormap

# Create custom colormap: white → company_blue
colors = ['#FFFFFF', '#E3F2FD', '#64B5F6', '#1565C0', '#0D47A1']
n_bins = 256
custom_cmap = LinearSegmentedColormap.from_list('company_blue', colors, N=n_bins)

# Use it
data = np.random.rand(8, 8)
fig, ax = plt.subplots(figsize=(7, 6))
im = ax.imshow(data, cmap=custom_cmap)
fig.colorbar(im, ax=ax)
ax.set_title('Custom Company Colormap')
plt.tight_layout()
plt.show()
```

### Colormap Selection Guide

| Data Type | Recommended | Avoid |
|-----------|-------------|-------|
| Sequential (0 → high) | `viridis`, `plasma`, `inferno` | `jet`, `rainbow` |
| Diverging (neg ↔ pos) | `coolwarm`, `RdBu`, `seismic` | `viridis` |
| Categorical | `Set2`, `tab10`, `Paired` | Any continuous cmap |
| Colorblind-safe | `viridis`, `cividis` | `red-green` schemes |

> **Critical:** Never use `jet` (rainbow). It's not perceptually uniform — equal differences in data don't look like equal differences in color. Use `viridis` (default since Matplotlib 2.0).

---

## 2.4 3D Plots

### What It Is
3D plots add a z-axis to visualize data in three dimensions. They can show surfaces, scatter points in 3D space, or wireframe meshes.

### Why It Matters
- Visualize functions of two variables: $z = f(x, y)$
- Useful for loss landscapes in machine learning
- Terrain/topography visualization
- Understanding higher-dimensional data

> **Important caveat:** 3D plots look impressive but are often LESS informative than 2D alternatives (contour plots, heatmaps). Use them sparingly and mainly for exploration/presentation, not analysis.

### Code Examples

```python
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D  # Required for 3D
import numpy as np

# === 3D SURFACE PLOT ===
fig = plt.figure(figsize=(10, 7))
ax = fig.add_subplot(111, projection='3d')  # Enable 3D

# Create mesh grid
x = np.linspace(-5, 5, 50)
y = np.linspace(-5, 5, 50)
X, Y = np.meshgrid(x, y)  # Create 2D grid from 1D arrays
Z = np.sin(np.sqrt(X**2 + Y**2))  # Function of X and Y

# Plot surface
surface = ax.plot_surface(X, Y, Z, 
                          cmap='viridis',    # Color based on Z value
                          alpha=0.8,
                          edgecolor='none')   # No wireframe edges

fig.colorbar(surface, ax=ax, shrink=0.6, label='Z value')
ax.set_xlabel('X')
ax.set_ylabel('Y')
ax.set_zlabel('Z')
ax.set_title('3D Surface: $z = \\sin(\\sqrt{x^2 + y^2})$')
ax.view_init(elev=30, azim=45)  # Camera angle (elevation, azimuth)
plt.tight_layout()
plt.show()
```

```python
# === 3D SCATTER PLOT ===
np.random.seed(42)
n = 300
x = np.random.randn(n)
y = np.random.randn(n)
z = np.random.randn(n)
colors = np.sqrt(x**2 + y**2 + z**2)  # Distance from origin

fig = plt.figure(figsize=(9, 7))
ax = fig.add_subplot(111, projection='3d')

scatter = ax.scatter(x, y, z, 
                     c=colors,        # Color by distance
                     cmap='plasma',
                     s=30, alpha=0.6)

fig.colorbar(scatter, ax=ax, shrink=0.6, label='Distance from Origin')
ax.set_xlabel('X')
ax.set_ylabel('Y')
ax.set_zlabel('Z')
ax.set_title('3D Scatter Plot')
plt.tight_layout()
plt.show()
```

```python
# === CONTOUR PLOT (2D alternative to 3D surface — often better!) ===
x = np.linspace(-3, 3, 100)
y = np.linspace(-3, 3, 100)
X, Y = np.meshgrid(x, y)
Z = X**2 + Y**2  # Simple bowl (like MSE loss landscape)

fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# Filled contour
cf = axes[0].contourf(X, Y, Z, levels=20, cmap='viridis')
fig.colorbar(cf, ax=axes[0])
axes[0].set_title('Filled Contour (contourf)')
axes[0].set_xlabel('X')
axes[0].set_ylabel('Y')
# Mark minimum
axes[0].plot(0, 0, 'r*', markersize=15, label='Minimum')
axes[0].legend()

# Line contour
cs = axes[1].contour(X, Y, Z, levels=15, cmap='plasma')
axes[1].clabel(cs, inline=True, fontsize=8)  # Add value labels on contour lines
axes[1].set_title('Line Contour (contour)')
axes[1].set_xlabel('X')
axes[1].set_ylabel('Y')

plt.tight_layout()
plt.show()
```

```python
# === ML USE CASE: Loss Landscape Visualization ===
# Simulating gradient descent on a loss surface

x = np.linspace(-3, 3, 100)
y = np.linspace(-3, 3, 100)
X, Y = np.meshgrid(x, y)
Z = (X - 1)**2 + (Y + 0.5)**2 + 0.5 * np.sin(3*X) * np.cos(3*Y)  # Non-convex loss

fig, ax = plt.subplots(figsize=(8, 6))
cf = ax.contourf(X, Y, Z, levels=30, cmap='RdYlBu_r', alpha=0.8)
ax.contour(X, Y, Z, levels=15, colors='black', alpha=0.3, linewidths=0.5)
fig.colorbar(cf, ax=ax, label='Loss')

# Simulate gradient descent path
path_x = [2.5]
path_y = [2.5]
lr = 0.1
for _ in range(30):
    # Approximate gradient (numerical)
    gx = 2*(path_x[-1] - 1) + 1.5*np.cos(3*path_x[-1])*np.cos(3*path_y[-1])
    gy = 2*(path_y[-1] + 0.5) - 1.5*np.sin(3*path_x[-1])*np.sin(3*path_y[-1])
    path_x.append(path_x[-1] - lr * gx)
    path_y.append(path_y[-1] - lr * gy)

ax.plot(path_x, path_y, 'w.-', markersize=5, linewidth=1.5, label='GD Path')
ax.plot(path_x[0], path_y[0], 'go', markersize=10, label='Start')
ax.plot(path_x[-1], path_y[-1], 'r*', markersize=15, label='End')

ax.set_xlabel('Weight 1')
ax.set_ylabel('Weight 2')
ax.set_title('Loss Landscape with Gradient Descent Path')
ax.legend(loc='upper left')
plt.tight_layout()
plt.show()
```

---

## 2.5 Animations

### What It Is
Matplotlib can create animated plots that show how data changes over time — like a flipbook of charts. Useful for presentations, social media, and understanding dynamic processes.

### Why It Matters
- Shows how data evolves over time (training curves, simulations)
- More engaging for presentations than static plots
- Can be saved as GIF or MP4 for sharing

### Code Examples

```python
import matplotlib.pyplot as plt
import matplotlib.animation as animation
import numpy as np

# === BASIC LINE ANIMATION ===
fig, ax = plt.subplots(figsize=(8, 5))
x = np.linspace(0, 2*np.pi, 200)
line, = ax.plot([], [], 'b-', linewidth=2)  # Empty line to animate

ax.set_xlim(0, 2*np.pi)
ax.set_ylim(-1.5, 1.5)
ax.set_title('Animated Sine Wave')
ax.grid(True, alpha=0.3)

def init():
    """Initialize empty frame"""
    line.set_data([], [])
    return line,

def animate(frame):
    """Update function called for each frame"""
    phase = frame * 0.1  # Phase shift increases each frame
    y = np.sin(x + phase)
    line.set_data(x, y)
    return line,

# Create animation
anim = animation.FuncAnimation(fig, animate, init_func=init,
                                frames=100,      # Total frames
                                interval=50,     # ms between frames
                                blit=True)       # Optimize rendering

# Save as GIF (requires pillow)
# anim.save('sine_wave.gif', writer='pillow', fps=20)

# Save as MP4 (requires ffmpeg)
# anim.save('sine_wave.mp4', writer='ffmpeg', fps=20)

plt.show()
```

```python
# === ANIMATED SCATTER (simulating particle movement) ===
fig, ax = plt.subplots(figsize=(8, 8))
ax.set_xlim(-10, 10)
ax.set_ylim(-10, 10)
ax.set_aspect('equal')

n_particles = 50
np.random.seed(42)
positions = np.random.randn(n_particles, 2) * 3
velocities = np.random.randn(n_particles, 2) * 0.3

scatter = ax.scatter(positions[:, 0], positions[:, 1], 
                     c='blue', s=50, alpha=0.7)

def animate(frame):
    global positions, velocities
    positions += velocities
    # Bounce off walls
    mask = np.abs(positions) > 9
    velocities[mask] *= -1
    positions = np.clip(positions, -9, 9)
    
    scatter.set_offsets(positions)
    colors = np.sqrt(positions[:, 0]**2 + positions[:, 1]**2)
    scatter.set_array(colors)
    return scatter,

anim = animation.FuncAnimation(fig, animate, frames=200, 
                                interval=30, blit=True)
plt.show()
```

```python
# === ML USE CASE: Animated Training Progress ===
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))

# Simulate training data
np.random.seed(42)
epochs = 100
train_loss = 2.0 * np.exp(-np.linspace(0, 5, epochs)) + np.random.randn(epochs) * 0.05
val_loss = 2.2 * np.exp(-np.linspace(0, 4.5, epochs)) + np.random.randn(epochs) * 0.08
train_acc = 1 - train_loss / 3
val_acc = 1 - val_loss / 3.2

line1, = ax1.plot([], [], 'b-', label='Train Loss')
line2, = ax1.plot([], [], 'r-', label='Val Loss')
ax1.set_xlim(0, epochs)
ax1.set_ylim(0, 2.5)
ax1.set_title('Loss')
ax1.legend()
ax1.grid(True, alpha=0.3)

line3, = ax2.plot([], [], 'b-', label='Train Acc')
line4, = ax2.plot([], [], 'r-', label='Val Acc')
ax2.set_xlim(0, epochs)
ax2.set_ylim(0, 1)
ax2.set_title('Accuracy')
ax2.legend()
ax2.grid(True, alpha=0.3)

def animate(frame):
    line1.set_data(range(frame), train_loss[:frame])
    line2.set_data(range(frame), val_loss[:frame])
    line3.set_data(range(frame), train_acc[:frame])
    line4.set_data(range(frame), val_acc[:frame])
    return line1, line2, line3, line4

anim = animation.FuncAnimation(fig, animate, frames=epochs,
                                interval=50, blit=True)
plt.tight_layout()
plt.show()
```

---

## 2.6 Built-in Styles and Themes

### What It Is
Matplotlib comes with pre-made style sheets that instantly change the look of all your plots — like CSS themes for websites. Instead of customizing every element manually, pick a style and go.

### Available Styles

```python
import matplotlib.pyplot as plt

# See all available styles
print(plt.style.available)
# Output: ['seaborn-v0_8', 'ggplot', 'dark_background', 'fivethirtyeight', ...]
```

### Code: Comparing Styles

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 10, 100)
styles_to_show = ['default', 'seaborn-v0_8', 'ggplot', 'dark_background',
                  'fivethirtyeight', 'bmh']

fig, axes = plt.subplots(2, 3, figsize=(15, 9))
axes = axes.flatten()

for ax, style_name in zip(axes, styles_to_show):
    with plt.style.context(style_name):  # Temporary style (doesn't affect other plots)
        ax.plot(x, np.sin(x), label='sin(x)')
        ax.plot(x, np.cos(x), label='cos(x)')
        ax.set_title(f'Style: {style_name}', fontsize=11)
        ax.legend(fontsize=8)

plt.suptitle('Matplotlib Style Comparison', fontsize=14)
plt.tight_layout()
plt.show()
```

```python
# === APPLYING A STYLE GLOBALLY ===
plt.style.use('seaborn-v0_8-whitegrid')  # Apply to all subsequent plots

# === TEMPORARY STYLE (context manager — preferred) ===
with plt.style.context('dark_background'):
    fig, ax = plt.subplots()
    ax.plot([1, 2, 3], [1, 4, 9])
    plt.show()
# After this block, style reverts to previous

# === COMBINING STYLES ===
plt.style.use(['seaborn-v0_8', 'seaborn-v0_8-whitegrid'])  # Later overrides earlier
```

### Style Recommendations

| Purpose | Style |
|---------|-------|
| Scientific papers | `'default'` or `'seaborn-v0_8-paper'` |
| Presentations | `'seaborn-v0_8-talk'` (larger fonts) |
| Blog posts | `'fivethirtyeight'` or `'ggplot'` |
| Dark-themed apps | `'dark_background'` |
| Clean & modern | `'seaborn-v0_8-whitegrid'` |

---

## 2.7 Advanced Plot Types

### Pie Charts (and why to avoid them)

```python
import matplotlib.pyplot as plt

# Pie chart — use SPARINGLY (bar charts are almost always better)
sizes = [35, 25, 20, 15, 5]
labels = ['Python', 'JavaScript', 'Java', 'C++', 'Other']
explode = (0.05, 0, 0, 0, 0)  # "Pull out" the first slice
colors = ['#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0', '#9966FF']

fig, ax = plt.subplots(figsize=(8, 8))
wedges, texts, autotexts = ax.pie(sizes, labels=labels, explode=explode,
                                   colors=colors, autopct='%1.1f%%',
                                   shadow=True, startangle=90,
                                   textprops={'fontsize': 11})
ax.set_title('Language Popularity', fontsize=14)
plt.tight_layout()
plt.show()
```

> **Best practice:** Limit pie charts to 4-5 categories max. If showing proportions, a horizontal stacked bar is usually clearer.

### Box Plot and Violin Plot

```python
import matplotlib.pyplot as plt
import numpy as np

np.random.seed(42)
data = [np.random.normal(0, std, 200) for std in [1, 2, 3, 1.5]]

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))

# Box plot — shows quartiles, median, outliers
bp = ax1.boxplot(data, labels=['A', 'B', 'C', 'D'], 
                 patch_artist=True,     # Fill boxes with color
                 medianprops={'color': 'red', 'linewidth': 2},
                 whiskerprops={'linewidth': 1.5},
                 notch=True)            # Confidence interval for median

colors = ['lightblue', 'lightgreen', 'lightyellow', 'lightpink']
for patch, color in zip(bp['boxes'], colors):
    patch.set_facecolor(color)

ax1.set_title('Box Plot')
ax1.set_ylabel('Value')
ax1.grid(axis='y', alpha=0.3)

# Violin plot — box plot + density distribution shape
vp = ax2.violinplot(data, positions=[1, 2, 3, 4], showmeans=True, showmedians=True)
ax2.set_xticks([1, 2, 3, 4])
ax2.set_xticklabels(['A', 'B', 'C', 'D'])
ax2.set_title('Violin Plot')
ax2.set_ylabel('Value')
ax2.grid(axis='y', alpha=0.3)

plt.tight_layout()
plt.show()
```

### Error Bars

```python
import matplotlib.pyplot as plt
import numpy as np

# Simulating experimental measurements with uncertainty
categories = ['Model A', 'Model B', 'Model C', 'Model D']
means = [0.85, 0.92, 0.88, 0.90]
std_errors = [0.03, 0.02, 0.04, 0.025]

fig, ax = plt.subplots(figsize=(8, 5))
bars = ax.bar(categories, means, yerr=std_errors,   # yerr = error bar heights
              capsize=5,                              # Cap width on error bars
              color='steelblue', edgecolor='navy', alpha=0.8,
              error_kw={'linewidth': 2, 'color': 'darkred'})

ax.set_ylabel('Accuracy')
ax.set_title('Model Comparison with Error Bars')
ax.set_ylim(0.7, 1.0)
ax.axhline(y=0.9, color='gray', linestyle='--', alpha=0.5, label='Threshold')
ax.legend()
ax.grid(axis='y', alpha=0.3)
plt.tight_layout()
plt.show()
```

### Fill Between (Confidence Bands)

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 10, 100)
y = np.sin(x)
y_upper = y + 0.3  # Upper confidence bound
y_lower = y - 0.3  # Lower confidence bound

fig, ax = plt.subplots(figsize=(10, 5))
ax.plot(x, y, 'b-', linewidth=2, label='Prediction')
ax.fill_between(x, y_lower, y_upper, alpha=0.2, color='blue', label='95% CI')

ax.set_xlabel('X')
ax.set_ylabel('Y')
ax.set_title('Prediction with Confidence Interval')
ax.legend()
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

---

## 2.8 Performance & Production Tips

### Saving Figures for Different Outputs

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(8, 5))
ax.plot([1, 2, 3], [1, 4, 9])

# For web (small file, good enough quality)
fig.savefig('plot_web.png', dpi=150, bbox_inches='tight')

# For print/papers (high quality, vector)
fig.savefig('plot_paper.pdf', bbox_inches='tight')  # Vector - infinite zoom
fig.savefig('plot_paper.svg', bbox_inches='tight')  # Vector - editable

# For presentations (high DPI PNG)
fig.savefig('plot_slides.png', dpi=300, bbox_inches='tight', 
            facecolor='white', transparent=False)
```

### Performance with Large Datasets

```python
import matplotlib.pyplot as plt
import numpy as np

# Problem: plotting 1M points is SLOW
n = 1_000_000
x = np.random.randn(n)
y = np.random.randn(n)

fig, axes = plt.subplots(1, 3, figsize=(15, 5))

# BAD: Scatter with 1M points (slow + overplotted)
# axes[0].scatter(x, y, alpha=0.01)  # Don't do this

# GOOD: Use hexbin for large datasets
axes[0].hexbin(x, y, gridsize=50, cmap='YlOrRd', mincnt=1)
axes[0].set_title('Hexbin (fast, informative)')

# GOOD: Use hist2d
axes[1].hist2d(x, y, bins=100, cmap='YlOrRd')
axes[1].set_title('2D Histogram (fast)')

# GOOD: Subsample for scatter
idx = np.random.choice(n, 5000, replace=False)  # Random 5000 points
axes[2].scatter(x[idx], y[idx], alpha=0.3, s=5)
axes[2].set_title('Subsampled Scatter (5000 pts)')

plt.tight_layout()
plt.show()
```

### Backend Selection

```python
import matplotlib
# Non-interactive backend (for scripts that save files, no display)
matplotlib.use('Agg')  # Must be called BEFORE importing pyplot

# Interactive backends:
# 'TkAgg'   - Default on most systems
# 'Qt5Agg'  - Qt-based (smooth)
# 'nbAgg'   - Jupyter notebooks
```

---

## 2.9 Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Using 3D when 2D suffices | 3D is harder to read, hides patterns | Use contour/heatmap instead |
| Jet/rainbow colormap | Not perceptually uniform, misleads | Use `viridis`, `cividis` |
| Too many annotations | Clutters the plot | Annotate only 2-3 key points |
| Not closing figures in loops | Memory leak crashes kernel | `plt.close(fig)` after saving |
| Ignoring DPI for saving | Blurry images in papers/slides | `dpi=300` for print |
| Twin axes without clear labels | Reader can't tell which axis is which | Color-code labels to match lines |
| Animation without `blit=True` | Extremely slow rendering | Always use `blit=True` |
| Using `plt.show()` in scripts that save | Shows blank if no display available | Use `Agg` backend + `savefig()` only |

---

## 2.10 Interview Questions

### Q1: How would you visualize a loss landscape for a neural network?
**Answer:** Use a 2D contour plot (`contourf`) rather than 3D surface, as it's easier to read. Plot the loss function over a 2D slice of parameter space, then overlay the gradient descent path using `ax.plot()`. Add the start point, end point, and colorbar showing loss values.

### Q2: What's the difference between `imshow()` and `pcolormesh()`?
**Answer:** 
- `imshow()`: Treats data as an image (equally spaced pixels). Faster, assumes regular grid.
- `pcolormesh()`: Handles non-uniform grids. Each cell can have different size. Slower but more flexible for irregular data.

### Q3: How do you prevent memory leaks when creating many plots in a loop?
**Answer:** Call `plt.close(fig)` after saving each figure, or use `plt.close('all')`. Alternatively, reuse the same figure/axes with `ax.clear()` instead of creating new ones each iteration.

### Q4: How would you make a plot accessible to colorblind viewers?
**Answer:** 
1. Use colorblind-safe colormaps (`viridis`, `cividis`)
2. Add different line styles/markers in addition to colors
3. Use direct labels instead of color-only legends
4. Avoid red-green combinations
5. Test with colorblindness simulators

### Q5: What's `tight_layout()` vs `constrained_layout`?
**Answer:** Both prevent element overlap, but:
- `tight_layout()`: Post-hoc adjustment, called explicitly. May fail with complex layouts.
- `constrained_layout=True`: Set at figure creation. Handles colorbars, suptitles, and complex GridSpec better. Preferred for new code.

---

## 2.11 Quick Reference

| Task | Code |
|------|------|
| Global style | `plt.rcParams.update({...})` |
| Temporary style | `with plt.style.context('ggplot'):` |
| Twin y-axis | `ax2 = ax1.twinx()` |
| Annotate point | `ax.annotate('text', xy=(x,y), xytext=(tx,ty), arrowprops={...})` |
| Shade region | `ax.fill_between(x, y1, y2, alpha=0.3)` |
| Vertical span | `ax.axvspan(x1, x2, alpha=0.1)` |
| Heatmap | `ax.imshow(data, cmap='viridis')` |
| Colorbar | `fig.colorbar(im, ax=ax)` |
| 3D plot | `ax = fig.add_subplot(111, projection='3d')` |
| Contour | `ax.contourf(X, Y, Z, levels=20, cmap='viridis')` |
| Box plot | `ax.boxplot(data, patch_artist=True)` |
| Violin plot | `ax.violinplot(data)` |
| Error bars | `ax.bar(x, y, yerr=errors, capsize=5)` |
| Animation | `animation.FuncAnimation(fig, animate, frames=N)` |
| Save high-res | `fig.savefig('plot.png', dpi=300, bbox_inches='tight')` |
| Close figure | `plt.close(fig)` |
| Format ticks | `ax.xaxis.set_major_formatter(ticker.FuncFormatter(func))` |
| LaTeX text | `ax.set_title(r'$\alpha + \beta$')` |
| Custom colormap | `LinearSegmentedColormap.from_list('name', colors)` |

---

*Previous: [01-Matplotlib-Fundamentals.md](01-Matplotlib-Fundamentals.md)*
*Next: [03-Seaborn-Statistical-Plots.md](03-Seaborn-Statistical-Plots.md)*
