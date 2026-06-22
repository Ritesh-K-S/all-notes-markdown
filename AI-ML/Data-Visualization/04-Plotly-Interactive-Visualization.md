# Chapter 04: Plotly — Interactive Visualization

## Table of Contents
- [4.1 Introduction to Plotly](#41-introduction-to-plotly)
- [4.2 Plotly Express — Quick & Beautiful Charts](#42-plotly-express--quick--beautiful-charts)
- [4.3 Graph Objects — Full Control](#43-graph-objects--full-control)
- [4.4 Interactive Features & Widgets](#44-interactive-features--widgets)
- [4.5 Subplots & Multiple Axes](#45-subplots--multiple-axes)
- [4.6 3D Visualizations](#46-3d-visualizations)
- [4.7 Maps & Geospatial Visualization](#47-maps--geospatial-visualization)
- [4.8 Animations & Transitions](#48-animations--transitions)
- [4.9 Dash — Building Interactive Dashboards](#49-dash--building-interactive-dashboards)
- [4.10 Exporting & Sharing](#410-exporting--sharing)
- [4.11 Common Mistakes](#411-common-mistakes)
- [4.12 Interview Questions](#412-interview-questions)
- [4.13 Quick Reference](#413-quick-reference)

---

## 4.1 Introduction to Plotly

### What It Is
Plotly is a Python library that creates **interactive** charts you can zoom, pan, hover over, and click — right in your browser or Jupyter notebook. Think of it like upgrading from a printed photo (Matplotlib) to a touchscreen (Plotly).

### Why It Matters
- **Dashboards**: Business teams need interactive reports, not static images
- **Exploration**: Hover over data points to see exact values without cluttering the chart
- **Presentation**: Impress stakeholders with professional, web-ready visualizations
- **Big Data**: Zoom into specific regions of large datasets without re-plotting

### How It Works — Architecture

```
┌─────────────────────────────────────────────┐
│                YOUR PYTHON CODE              │
├─────────────────────────────────────────────┤
│  Plotly Express (High-level, quick plots)    │
├─────────────────────────────────────────────┤
│  Graph Objects (Low-level, full control)     │
├─────────────────────────────────────────────┤
│  Plotly.js (JavaScript rendering engine)     │
├─────────────────────────────────────────────┤
│  Browser / Notebook (D3.js + WebGL)          │
└─────────────────────────────────────────────┘
```

**Analogy**: Plotly Express is like ordering a pizza from a menu (quick, predefined options). Graph Objects is like making pizza from scratch (full control over every ingredient).

### Two APIs — When to Use Which

| Feature | Plotly Express (`px`) | Graph Objects (`go`) |
|---------|----------------------|---------------------|
| Speed of coding | ⚡ Very fast | 🐢 More verbose |
| Customization | Medium | Full control |
| Learning curve | Beginner-friendly | Steeper |
| Best for | EDA, quick dashboards | Production, complex layouts |
| DataFrame integration | Native | Manual |

### Installation & Setup

```python
# Install Plotly
# pip install plotly

# For Jupyter notebook support
# pip install plotly nbformat

# For Dash (dashboards)
# pip install dash

# Import conventions
import plotly.express as px          # High-level API
import plotly.graph_objects as go    # Low-level API
from plotly.subplots import make_subplots  # Multi-panel figures
import plotly.io as pio              # Export/rendering settings

# Set default renderer for Jupyter
# pio.renderers.default = "notebook"  # or "browser", "vscode"
```

---

## 4.2 Plotly Express — Quick & Beautiful Charts

### What It Is
Plotly Express (`px`) is the **high-level** interface that creates complex, interactive charts in a single function call. It's the fastest way to go from data to visualization.

### Why It Matters
- One line of code → publication-quality interactive chart
- Handles color mapping, legends, hover info automatically
- Direct DataFrame column mapping (no manual array extraction)

### Line Charts

```python
import plotly.express as px
import pandas as pd
import numpy as np

# Create sample stock data
np.random.seed(42)
dates = pd.date_range('2023-01-01', periods=252, freq='B')  # Business days
df = pd.DataFrame({
    'Date': np.tile(dates, 3),
    'Price': np.concatenate([
        100 + np.cumsum(np.random.randn(252) * 2),   # Stock A
        150 + np.cumsum(np.random.randn(252) * 3),   # Stock B
        80 + np.cumsum(np.random.randn(252) * 1.5),  # Stock C
    ]),
    'Stock': np.repeat(['AAPL', 'GOOGL', 'MSFT'], 252)
})

# Single line to create a multi-line interactive chart
fig = px.line(
    df,
    x='Date',
    y='Price',
    color='Stock',              # Separate line per stock
    title='Stock Prices (2023)',
    labels={'Price': 'Price ($)'},  # Rename axis labels
    template='plotly_white'     # Clean white background
)

# Customize hover information
fig.update_traces(hovertemplate='%{x|%b %d}<br>$%{y:.2f}')
fig.show()
```

### Bar Charts

```python
import plotly.express as px
import pandas as pd

# Sales data
df_sales = pd.DataFrame({
    'Quarter': ['Q1', 'Q2', 'Q3', 'Q4'] * 3,
    'Revenue': [120, 150, 180, 200, 90, 110, 140, 160, 70, 85, 100, 130],
    'Region': ['North'] * 4 + ['South'] * 4 + ['West'] * 4
})

# Grouped bar chart
fig = px.bar(
    df_sales,
    x='Quarter',
    y='Revenue',
    color='Region',
    barmode='group',           # 'group' = side-by-side, 'stack' = stacked
    title='Quarterly Revenue by Region',
    text='Revenue',            # Show values on bars
    color_discrete_sequence=px.colors.qualitative.Set2  # Custom color palette
)

# Format text on bars
fig.update_traces(texttemplate='$%{text}M', textposition='outside')
fig.update_layout(yaxis_title='Revenue (Millions $)')
fig.show()
```

### Scatter Plots

```python
import plotly.express as px

# Use built-in dataset
df = px.data.gapminder().query("year == 2007")

# Bubble chart — 4 dimensions in one plot!
fig = px.scatter(
    df,
    x='gdpPercap',                    # X-axis
    y='lifeExp',                      # Y-axis
    size='pop',                       # Bubble size = population
    color='continent',                # Color = continent
    hover_name='country',             # Show country on hover
    log_x=True,                       # Log scale for GDP
    size_max=60,                      # Max bubble size
    title='Gapminder 2007: GDP vs Life Expectancy',
    labels={
        'gdpPercap': 'GDP per Capita ($)',
        'lifeExp': 'Life Expectancy (years)',
        'pop': 'Population'
    }
)

fig.show()
```

### Histograms & Distributions

```python
import plotly.express as px
import numpy as np
import pandas as pd

# Generate data from different distributions
np.random.seed(42)
df = pd.DataFrame({
    'Score': np.concatenate([
        np.random.normal(70, 10, 500),    # Class A
        np.random.normal(80, 8, 500),     # Class B
    ]),
    'Class': ['A'] * 500 + ['B'] * 500
})

# Overlapping histograms with KDE
fig = px.histogram(
    df,
    x='Score',
    color='Class',
    nbins=40,                         # Number of bins
    marginal='box',                   # Add marginal box plot on top
    opacity=0.7,                      # Transparency for overlap
    barmode='overlay',                # Overlay instead of stack
    title='Score Distribution by Class',
    histnorm='probability density'    # Normalize to density
)

fig.show()
```

### Heatmaps

```python
import plotly.express as px
import numpy as np
import pandas as pd

# Correlation matrix
np.random.seed(42)
data = np.random.randn(100, 5)
data[:, 1] = data[:, 0] * 0.8 + np.random.randn(100) * 0.5  # Correlated
df = pd.DataFrame(data, columns=['Revenue', 'Marketing', 'Users', 'Costs', 'Profit'])
corr = df.corr()

fig = px.imshow(
    corr,
    text_auto='.2f',              # Show values with 2 decimals
    color_continuous_scale='RdBu_r',  # Red-Blue diverging colormap
    zmin=-1, zmax=1,              # Fix color range
    title='Feature Correlation Matrix',
    aspect='auto'
)

fig.update_layout(width=600, height=500)
fig.show()
```

### Box Plots & Violin Plots

```python
import plotly.express as px
import pandas as pd
import numpy as np

# Employee salary data
np.random.seed(42)
df = pd.DataFrame({
    'Salary': np.concatenate([
        np.random.normal(60000, 10000, 200),
        np.random.normal(90000, 15000, 200),
        np.random.normal(120000, 20000, 200),
    ]),
    'Department': ['Engineering'] * 200 + ['Sales'] * 200 + ['Management'] * 200,
    'Gender': np.random.choice(['Male', 'Female'], 600)
})

# Violin plot — shows full distribution shape
fig = px.violin(
    df,
    x='Department',
    y='Salary',
    color='Gender',
    box=True,                    # Add box plot inside violin
    points='outliers',           # Show only outlier points
    title='Salary Distribution by Department & Gender',
    labels={'Salary': 'Annual Salary ($)'}
)

fig.update_layout(yaxis_tickformat='$,.0f')  # Format as currency
fig.show()
```

### Pie & Sunburst Charts

```python
import plotly.express as px
import pandas as pd

# Hierarchical data — company structure
df = pd.DataFrame({
    'Department': ['Engineering', 'Engineering', 'Engineering',
                   'Sales', 'Sales', 'Marketing', 'Marketing'],
    'Team': ['Backend', 'Frontend', 'ML', 'Enterprise', 'SMB', 'Digital', 'Brand'],
    'Headcount': [30, 25, 15, 20, 35, 18, 12]
})

# Sunburst — hierarchical pie chart (much better than plain pie!)
fig = px.sunburst(
    df,
    path=['Department', 'Team'],   # Hierarchy levels
    values='Headcount',
    title='Company Headcount by Department & Team',
    color='Headcount',
    color_continuous_scale='Blues'
)

fig.show()
```

> **Pro Tip**: Avoid pie charts for more than 5 categories. Use sunburst for hierarchical data, treemap for showing proportions of many categories.

---

## 4.3 Graph Objects — Full Control

### What It Is
`plotly.graph_objects` (commonly imported as `go`) gives you **complete control** over every element of your chart. Every Plotly Express chart is actually built on top of Graph Objects internally.

### Why It Matters
- Need to combine different chart types (bar + line on same axes)
- Want pixel-perfect control over annotations, shapes, fonts
- Building reusable chart templates for production
- Complex multi-axis, multi-layer visualizations

### Basic Structure

```python
import plotly.graph_objects as go
import numpy as np

# Every chart starts with a Figure object
fig = go.Figure()

# Add traces (layers of data) one by one
x = np.linspace(0, 10, 100)

fig.add_trace(go.Scatter(
    x=x,
    y=np.sin(x),
    mode='lines',              # 'lines', 'markers', 'lines+markers'
    name='sin(x)',             # Legend label
    line=dict(color='blue', width=2, dash='solid')  # Line styling
))

fig.add_trace(go.Scatter(
    x=x,
    y=np.cos(x),
    mode='lines+markers',
    name='cos(x)',
    line=dict(color='red', width=2, dash='dash'),
    marker=dict(size=4, symbol='circle')
))

# Update layout (everything about the figure that isn't data)
fig.update_layout(
    title=dict(text='Trigonometric Functions', font=dict(size=20)),
    xaxis=dict(title='x', gridcolor='lightgray'),
    yaxis=dict(title='f(x)', gridcolor='lightgray'),
    template='plotly_white',
    legend=dict(x=0.02, y=0.98),  # Position legend
    width=800,
    height=450
)

fig.show()
```

### Combining Chart Types

```python
import plotly.graph_objects as go
import numpy as np

months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun',
          'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec']
revenue = [45, 52, 48, 61, 55, 67, 72, 69, 75, 80, 78, 85]
growth_rate = [None, 15.5, -7.7, 27.1, -9.8, 21.8, 7.5, -4.2, 8.7, 6.7, -2.5, 9.0]

fig = go.Figure()

# Bar chart for revenue
fig.add_trace(go.Bar(
    x=months,
    y=revenue,
    name='Revenue ($M)',
    marker_color='steelblue',
    opacity=0.7,
    yaxis='y'                   # Primary Y-axis
))

# Line chart for growth rate on secondary Y-axis
fig.add_trace(go.Scatter(
    x=months,
    y=growth_rate,
    name='Growth Rate (%)',
    mode='lines+markers',
    line=dict(color='orangered', width=3),
    marker=dict(size=8),
    yaxis='y2'                  # Secondary Y-axis
))

# Add zero line for growth rate
fig.add_hline(y=0, line_dash='dot', line_color='gray', opacity=0.5)

fig.update_layout(
    title='Monthly Revenue & Growth Rate',
    yaxis=dict(title='Revenue ($M)', side='left'),
    yaxis2=dict(title='Growth Rate (%)', side='right', overlaying='y'),
    legend=dict(x=0.01, y=0.99),
    template='plotly_white'
)

fig.show()
```

### Annotations & Shapes

```python
import plotly.graph_objects as go
import numpy as np

x = np.linspace(0, 10, 200)
y = np.sin(x) * np.exp(-x / 5)

fig = go.Figure()
fig.add_trace(go.Scatter(x=x, y=y, mode='lines', name='Damped Sine'))

# Add annotation pointing to a specific point
fig.add_annotation(
    x=1.57,                    # Arrow points TO here
    y=np.sin(1.57) * np.exp(-1.57/5),
    text="Peak amplitude",
    showarrow=True,
    arrowhead=2,
    arrowsize=1.5,
    ax=-40, ay=-40,            # Arrow tail offset (pixels)
    font=dict(size=12, color='darkred'),
    bordercolor='darkred',
    borderwidth=1,
    borderpad=4,
    bgcolor='lightyellow'
)

# Add a shaded region
fig.add_vrect(
    x0=3, x1=7,
    fillcolor='lightblue',
    opacity=0.3,
    line_width=0,
    annotation_text="Region of Interest",
    annotation_position="top left"
)

# Add a horizontal threshold line
fig.add_hline(
    y=0.2,
    line_dash='dash',
    line_color='green',
    annotation_text="Threshold",
    annotation_position="right"
)

fig.update_layout(title='Annotated Damped Sine Wave', template='plotly_white')
fig.show()
```

### Custom Hover Templates

```python
import plotly.graph_objects as go
import pandas as pd
import numpy as np

# Product data
np.random.seed(42)
df = pd.DataFrame({
    'Product': [f'Product {chr(65+i)}' for i in range(10)],
    'Revenue': np.random.uniform(50, 200, 10).round(1),
    'Units': np.random.randint(100, 5000, 10),
    'Rating': np.random.uniform(3.5, 5.0, 10).round(2),
    'Category': np.random.choice(['Electronics', 'Fashion', 'Food'], 10)
})

fig = go.Figure()

# Custom hover template with HTML formatting
fig.add_trace(go.Scatter(
    x=df['Revenue'],
    y=df['Units'],
    mode='markers',
    marker=dict(
        size=df['Rating'] * 8,           # Size by rating
        color=df['Revenue'],             # Color by revenue
        colorscale='Viridis',
        showscale=True,
        colorbar=dict(title='Revenue ($K)')
    ),
    text=df['Product'],
    customdata=np.stack([df['Category'], df['Rating']], axis=-1),
    hovertemplate=(
        '<b>%{text}</b><br>'             # Product name (bold)
        'Revenue: $%{x:.1f}K<br>'       # X value
        'Units Sold: %{y:,}<br>'        # Y value with commas
        'Category: %{customdata[0]}<br>' # Extra data
        'Rating: %{customdata[1]}⭐<br>'# Extra data
        '<extra></extra>'                # Remove trace name from hover
    )
))

fig.update_layout(
    title='Product Performance Dashboard',
    xaxis_title='Revenue ($K)',
    yaxis_title='Units Sold',
    template='plotly_white'
)
fig.show()
```

---

## 4.4 Interactive Features & Widgets

### What It Is
Plotly charts are interactive by default — zoom, pan, hover, select. You can add buttons, sliders, and dropdowns to create mini-apps without writing JavaScript.

### Buttons — Toggle Views

```python
import plotly.graph_objects as go
import numpy as np

x = np.linspace(0, 2*np.pi, 100)

fig = go.Figure()
fig.add_trace(go.Scatter(x=x, y=np.sin(x), name='sin(x)'))
fig.add_trace(go.Scatter(x=x, y=np.cos(x), name='cos(x)', visible=False))
fig.add_trace(go.Scatter(x=x, y=np.tan(x), name='tan(x)', visible=False))

# Add dropdown buttons to toggle between functions
fig.update_layout(
    updatemenus=[
        dict(
            type='buttons',
            direction='right',
            x=0.1, y=1.15,
            buttons=[
                dict(label='sin(x)',
                     method='update',
                     args=[{'visible': [True, False, False]},
                           {'title': 'Sine Function'}]),
                dict(label='cos(x)',
                     method='update',
                     args=[{'visible': [False, True, False]},
                           {'title': 'Cosine Function'}]),
                dict(label='tan(x)',
                     method='update',
                     args=[{'visible': [False, False, True]},
                           {'title': 'Tangent Function'}]),
                dict(label='All',
                     method='update',
                     args=[{'visible': [True, True, True]},
                           {'title': 'All Functions'}]),
            ]
        )
    ],
    title='Sine Function'
)

fig.show()
```

### Sliders — Animate Parameters

```python
import plotly.graph_objects as go
import numpy as np

# Create frames for different frequencies
x = np.linspace(0, 2*np.pi, 200)
frames = []
slider_steps = []

for freq in np.arange(1, 6, 0.5):
    frames.append(go.Frame(
        data=[go.Scatter(x=x, y=np.sin(freq * x), mode='lines')],
        name=str(freq)
    ))
    slider_steps.append(dict(
        method='animate',
        args=[[str(freq)], dict(mode='immediate', frame=dict(duration=0))],
        label=f'{freq:.1f}'
    ))

# Initial plot
fig = go.Figure(
    data=[go.Scatter(x=x, y=np.sin(x), mode='lines')],
    frames=frames
)

fig.update_layout(
    sliders=[dict(
        active=0,
        steps=slider_steps,
        currentvalue=dict(prefix='Frequency: '),
        pad=dict(t=50)
    )],
    title='Interactive Sine Wave — Adjust Frequency',
    template='plotly_white',
    yaxis=dict(range=[-1.5, 1.5])  # Fix y-axis for smooth animation
)

fig.show()
```

### Range Slider & Selector (Time Series)

```python
import plotly.graph_objects as go
import pandas as pd
import numpy as np

# Generate time series
np.random.seed(42)
dates = pd.date_range('2020-01-01', '2023-12-31', freq='D')
price = 100 + np.cumsum(np.random.randn(len(dates)) * 2)

fig = go.Figure()
fig.add_trace(go.Scatter(x=dates, y=price, mode='lines', name='Price'))

# Add range slider and preset time range buttons
fig.update_xaxes(
    rangeslider_visible=True,          # Show range slider below
    rangeselector=dict(                # Preset range buttons
        buttons=[
            dict(count=1, label='1M', step='month', stepmode='backward'),
            dict(count=6, label='6M', step='month', stepmode='backward'),
            dict(count=1, label='YTD', step='year', stepmode='todate'),
            dict(count=1, label='1Y', step='year', stepmode='backward'),
            dict(step='all', label='All')
        ]
    )
)

fig.update_layout(
    title='Stock Price with Range Selector',
    template='plotly_white',
    height=500
)
fig.show()
```

---

## 4.5 Subplots & Multiple Axes

### What It Is
Arrange multiple charts in a grid layout within a single figure, similar to Matplotlib's subplots but with interactive features preserved.

### Basic Subplots

```python
from plotly.subplots import make_subplots
import plotly.graph_objects as go
import numpy as np

# Create 2x2 grid
fig = make_subplots(
    rows=2, cols=2,
    subplot_titles=('Scatter', 'Line', 'Bar', 'Histogram'),
    vertical_spacing=0.12,     # Space between rows
    horizontal_spacing=0.08    # Space between columns
)

x = np.linspace(0, 10, 50)

# Top-left: Scatter
fig.add_trace(
    go.Scatter(x=np.random.randn(100), y=np.random.randn(100),
               mode='markers', marker=dict(color='steelblue')),
    row=1, col=1
)

# Top-right: Line
fig.add_trace(
    go.Scatter(x=x, y=np.sin(x), mode='lines',
               line=dict(color='orangered')),
    row=1, col=2
)

# Bottom-left: Bar
fig.add_trace(
    go.Bar(x=['A', 'B', 'C', 'D'], y=[23, 45, 12, 37],
            marker_color='seagreen'),
    row=2, col=1
)

# Bottom-right: Histogram
fig.add_trace(
    go.Histogram(x=np.random.normal(0, 1, 500),
                 marker_color='mediumpurple'),
    row=2, col=2
)

fig.update_layout(height=600, showlegend=False, title_text='Subplot Grid Demo')
fig.show()
```

### Mixed Subplot Types

```python
from plotly.subplots import make_subplots
import plotly.graph_objects as go
import numpy as np

# Mix different chart types with specs
fig = make_subplots(
    rows=2, cols=2,
    specs=[
        [{"type": "scatter"}, {"type": "pie"}],       # Row 1
        [{"type": "bar", "colspan": 2}, None]         # Row 2: bar spans both cols
    ],
    subplot_titles=('Trend', 'Distribution', 'Comparison')
)

# Scatter
fig.add_trace(go.Scatter(x=[1,2,3,4,5], y=[2,4,3,5,4], mode='lines+markers'),
              row=1, col=1)

# Pie
fig.add_trace(go.Pie(labels=['A','B','C'], values=[30,50,20]),
              row=1, col=2)

# Bar spanning full width
fig.add_trace(go.Bar(x=['Jan','Feb','Mar','Apr'], y=[10,15,12,18]),
              row=2, col=1)

fig.update_layout(height=600, title_text='Mixed Chart Types')
fig.show()
```

### Shared Axes

```python
from plotly.subplots import make_subplots
import plotly.graph_objects as go
import numpy as np

# Shared X-axis — great for comparing time-aligned metrics
fig = make_subplots(
    rows=3, cols=1,
    shared_xaxes=True,         # All share the same X-axis
    vertical_spacing=0.05,
    subplot_titles=('Price', 'Volume', 'RSI'),
    row_heights=[0.5, 0.25, 0.25]  # Different heights
)

x = np.arange(100)
price = 100 + np.cumsum(np.random.randn(100))
volume = np.random.randint(1000, 10000, 100)
rsi = 50 + np.cumsum(np.random.randn(100) * 3)
rsi = np.clip(rsi, 0, 100)

fig.add_trace(go.Scatter(x=x, y=price, name='Price'), row=1, col=1)
fig.add_trace(go.Bar(x=x, y=volume, name='Volume', marker_color='gray'), row=2, col=1)
fig.add_trace(go.Scatter(x=x, y=rsi, name='RSI', line=dict(color='purple')), row=3, col=1)

# Add RSI threshold lines
fig.add_hline(y=70, line_dash='dash', line_color='red', row=3, col=1)
fig.add_hline(y=30, line_dash='dash', line_color='green', row=3, col=1)

fig.update_layout(height=700, title='Stock Analysis Dashboard')
fig.show()
```

---

## 4.6 3D Visualizations

### What It Is
Plotly renders true 3D charts using WebGL — you can rotate, zoom, and pan in 3D space interactively.

### 3D Scatter Plot

```python
import plotly.graph_objects as go
import numpy as np

# Generate 3D cluster data
np.random.seed(42)
n = 200

# Three clusters
clusters = []
for center, color, name in [([2,2,2], 'red', 'A'),
                              ([8,8,8], 'blue', 'B'),
                              ([5,2,8], 'green', 'C')]:
    data = np.random.randn(n, 3) + center
    clusters.append((data, color, name))

fig = go.Figure()

for data, color, name in clusters:
    fig.add_trace(go.Scatter3d(
        x=data[:, 0], y=data[:, 1], z=data[:, 2],
        mode='markers',
        marker=dict(size=3, color=color, opacity=0.7),
        name=f'Cluster {name}'
    ))

fig.update_layout(
    title='3D Cluster Visualization',
    scene=dict(
        xaxis_title='Feature 1',
        yaxis_title='Feature 2',
        zaxis_title='Feature 3'
    ),
    width=700, height=600
)
fig.show()
```

### 3D Surface Plot

```python
import plotly.graph_objects as go
import numpy as np

# Create a mathematical surface
x = np.linspace(-5, 5, 50)
y = np.linspace(-5, 5, 50)
X, Y = np.meshgrid(x, y)

# Saddle point surface: z = x² - y²
Z = X**2 - Y**2

fig = go.Figure(data=[go.Surface(
    x=X, y=Y, z=Z,
    colorscale='RdBu',
    contours=dict(              # Add contour lines
        z=dict(show=True, usecolormap=True, project_z=True)
    )
)])

fig.update_layout(
    title='Saddle Point: z = x² - y²',
    scene=dict(
        xaxis_title='X',
        yaxis_title='Y',
        zaxis_title='Z',
        camera=dict(eye=dict(x=1.5, y=1.5, z=1.0))  # Initial camera angle
    ),
    width=700, height=600
)
fig.show()
```

> **Pro Tip**: 3D plots look impressive but are often harder to read than 2D alternatives. Use them when the 3rd dimension truly adds information (e.g., loss surfaces in ML, geographic terrain). For most data, a heatmap or contour plot is more readable.

---

## 4.7 Maps & Geospatial Visualization

### What It Is
Plotly can create interactive maps — choropleth maps (colored regions), scatter maps (points on a map), and more.

### Choropleth Map (World)

```python
import plotly.express as px
import pandas as pd

# Use built-in gapminder data
df = px.data.gapminder().query("year == 2007")

fig = px.choropleth(
    df,
    locations='iso_alpha',             # Country codes (ISO 3166)
    color='lifeExp',                   # Color by life expectancy
    hover_name='country',
    color_continuous_scale='Viridis',
    title='Life Expectancy by Country (2007)',
    labels={'lifeExp': 'Life Expectancy'}
)

fig.update_layout(
    geo=dict(showframe=False, showcoastlines=True),
    width=900, height=500
)
fig.show()
```

### Scatter Map with Mapbox

```python
import plotly.express as px
import pandas as pd

# US cities data
df = pd.DataFrame({
    'City': ['New York', 'Los Angeles', 'Chicago', 'Houston', 'Phoenix'],
    'Lat': [40.71, 34.05, 41.88, 29.76, 33.45],
    'Lon': [-74.01, -118.24, -87.63, -95.37, -112.07],
    'Population': [8336817, 3979576, 2693976, 2320268, 1680992],
    'Category': ['Finance', 'Entertainment', 'Industry', 'Energy', 'Tech']
})

fig = px.scatter_mapbox(
    df,
    lat='Lat',
    lon='Lon',
    size='Population',
    color='Category',
    hover_name='City',
    size_max=40,
    zoom=3,
    mapbox_style='open-street-map',   # Free, no token needed
    title='Top US Cities by Population'
)

fig.update_layout(height=500)
fig.show()
```

---

## 4.8 Animations & Transitions

### What It Is
Plotly can animate transitions between data states — perfect for showing how data evolves over time.

### Animated Scatter (Gapminder Style)

```python
import plotly.express as px

# The famous Hans Rosling animation
df = px.data.gapminder()

fig = px.scatter(
    df,
    x='gdpPercap',
    y='lifeExp',
    size='pop',
    color='continent',
    hover_name='country',
    log_x=True,
    size_max=55,
    animation_frame='year',            # Animate over years
    animation_group='country',         # Track same country across frames
    range_x=[100, 100000],             # Fix axes so animation is smooth
    range_y=[25, 90],
    title='Gapminder: The Story of Global Development'
)

fig.update_layout(
    xaxis_title='GDP per Capita ($, log scale)',
    yaxis_title='Life Expectancy (years)'
)
fig.show()
```

### Animated Bar Chart Race

```python
import plotly.express as px
import pandas as pd
import numpy as np

# Simulate growing company revenues
np.random.seed(42)
companies = ['Apple', 'Google', 'Microsoft', 'Amazon', 'Meta']
years = range(2018, 2024)

records = []
base_rev = [265, 137, 110, 233, 56]
for year in years:
    for i, company in enumerate(companies):
        rev = base_rev[i] * (1 + np.random.uniform(0.05, 0.25)) ** (year - 2018)
        records.append({'Company': company, 'Year': year, 'Revenue': round(rev, 1)})

df = pd.DataFrame(records)

fig = px.bar(
    df,
    x='Revenue',
    y='Company',
    color='Company',
    animation_frame='Year',
    orientation='h',
    range_x=[0, 700],
    title='Tech Company Revenue Growth',
    labels={'Revenue': 'Revenue ($B)'}
)

fig.update_layout(showlegend=False, yaxis={'categoryorder': 'total ascending'})
fig.show()
```

---

## 4.9 Dash — Building Interactive Dashboards

### What It Is
Dash is Plotly's framework for building **web-based analytical dashboards** in pure Python — no JavaScript, HTML, or CSS required (though you can use them). Think of it as "Streamlit's more powerful cousin."

### Why It Matters
- Deploy interactive dashboards as web apps
- Connect to live data sources
- Add callbacks (user interaction → chart updates)
- Production-ready (used by Fortune 500 companies)

### Architecture

```
┌─────────────────────────────────────────┐
│              DASH APP                     │
├─────────────────────────────────────────┤
│  Layout (HTML + Plotly components)       │
│    ├── dcc.Graph (Plotly charts)         │
│    ├── dcc.Dropdown, Slider, Input      │
│    └── html.Div, H1, P (HTML elements)  │
├─────────────────────────────────────────┤
│  Callbacks (Python functions)            │
│    Input → Process → Output              │
├─────────────────────────────────────────┤
│  Flask Server (auto-managed)             │
└─────────────────────────────────────────┘
```

### Basic Dash App

```python
# Save as app.py and run: python app.py
# Then open http://127.0.0.1:8050 in browser

from dash import Dash, html, dcc, callback, Output, Input
import plotly.express as px
import pandas as pd

# Load data
df = px.data.gapminder()

# Initialize the Dash app
app = Dash(__name__)

# Layout — what the user sees
app.layout = html.Div([
    html.H1('Gapminder Explorer', style={'textAlign': 'center'}),

    # Dropdown for continent selection
    html.Div([
        html.Label('Select Continent:'),
        dcc.Dropdown(
            id='continent-dropdown',
            options=[{'label': c, 'value': c} for c in df['continent'].unique()],
            value='Asia',               # Default value
            clearable=False
        )
    ], style={'width': '30%', 'margin': '0 auto'}),

    # Year slider
    html.Div([
        html.Label('Select Year:'),
        dcc.Slider(
            id='year-slider',
            min=df['year'].min(),
            max=df['year'].max(),
            step=5,
            value=2007,
            marks={str(y): str(y) for y in df['year'].unique()}
        )
    ], style={'width': '80%', 'margin': '20px auto'}),

    # The chart
    dcc.Graph(id='scatter-chart')
])

# Callback — when inputs change, update the chart
@callback(
    Output('scatter-chart', 'figure'),      # What to update
    Input('continent-dropdown', 'value'),    # Trigger 1
    Input('year-slider', 'value')            # Trigger 2
)
def update_chart(selected_continent, selected_year):
    # Filter data based on user selections
    filtered = df[(df['continent'] == selected_continent) &
                  (df['year'] == selected_year)]

    fig = px.scatter(
        filtered,
        x='gdpPercap', y='lifeExp',
        size='pop', hover_name='country',
        log_x=True, size_max=50,
        title=f'{selected_continent} in {selected_year}'
    )
    return fig

# Run the server
if __name__ == '__main__':
    app.run(debug=True)
```

### Multi-Page Dash App (Structure)

```python
# For larger apps, use this project structure:
#
# my_dashboard/
# ├── app.py              # Main entry point
# ├── pages/
# │   ├── overview.py     # Page 1
# │   ├── analysis.py     # Page 2
# │   └── settings.py     # Page 3
# ├── components/
# │   ├── navbar.py       # Reusable navigation bar
# │   └── cards.py        # Reusable card components
# ├── data/
# │   └── dataset.csv     # Data files
# └── assets/
#     └── style.css        # Custom CSS

# app.py (multi-page)
from dash import Dash, html, dcc, page_container

app = Dash(__name__, use_pages=True)

app.layout = html.Div([
    html.H1('My Dashboard'),
    html.Div([
        dcc.Link('Overview', href='/'),
        dcc.Link('Analysis', href='/analysis'),
    ]),
    page_container  # Pages render here
])

if __name__ == '__main__':
    app.run(debug=True)
```

> **Pro Tip**: For quick prototypes, use **Streamlit** (less code, less control). For production dashboards with complex interactivity, use **Dash** (more code, full control). For static reports, use **Plotly + HTML export**.

---

## 4.10 Exporting & Sharing

### Export Formats

```python
import plotly.express as px
import plotly.io as pio

df = px.data.iris()
fig = px.scatter(df, x='sepal_width', y='sepal_length', color='species')

# 1. Interactive HTML (self-contained, can email/share)
fig.write_html('chart.html', include_plotlyjs='cdn')  # Smaller file (uses CDN)
fig.write_html('chart_standalone.html')  # Larger but works offline

# 2. Static images (requires kaleido: pip install -U kaleido)
fig.write_image('chart.png', width=800, height=500, scale=2)  # High-res PNG
fig.write_image('chart.svg')   # Vector format (for papers/publications)
fig.write_image('chart.pdf')   # PDF format

# 3. JSON (for web apps, reconstructible)
fig.write_json('chart.json')

# 4. Display in different contexts
# fig.show('browser')      # Open in default browser
# fig.show('notebook')     # Jupyter notebook
# fig.show('vscode')       # VS Code
```

### Theming & Templates

```python
import plotly.express as px
import plotly.io as pio
import plotly.graph_objects as go

# Available built-in templates
# pio.templates  → shows all available templates

# Set global default template
pio.templates.default = 'plotly_white'

# Create a custom template
custom_template = go.layout.Template(
    layout=go.Layout(
        font=dict(family='Arial', size=14, color='#333333'),
        title=dict(font=dict(size=20, color='#1a1a2e')),
        colorway=['#e63946', '#457b9d', '#2a9d8f', '#e9c46a', '#264653'],
        plot_bgcolor='#f8f9fa',
        paper_bgcolor='white',
        xaxis=dict(gridcolor='#dee2e6', linecolor='#adb5bd'),
        yaxis=dict(gridcolor='#dee2e6', linecolor='#adb5bd'),
    )
)

# Register and use
pio.templates['my_brand'] = custom_template
pio.templates.default = 'my_brand'

# Now all plots use your brand template
fig = px.bar(x=['Q1','Q2','Q3','Q4'], y=[100, 120, 115, 140])
fig.update_layout(title='Quarterly Sales')
fig.show()
```

---

## 4.11 Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Not fixing axis ranges in animations | Charts "jump" as axes rescale each frame | Set `range_x` and `range_y` explicitly |
| Using 3D when 2D suffices | Harder to read, slower to render | Use heatmaps/contours instead |
| Too many traces on one chart | Cluttered, slow rendering | Use facets (`facet_col`) or subplots |
| Forgetting `<extra></extra>` in hover | Shows ugly trace name in hover box | Add it to `hovertemplate` |
| Not using `template='plotly_white'` | Default gray background looks dated | Always set a clean template |
| Massive datasets without sampling | Browser freezes (>100K points) | Use `datashader` or sample data |
| Pie charts for many categories | Unreadable beyond 5 slices | Use bar charts or treemaps |
| Not setting `fig.update_layout(height=...)` | Charts too short/tall in notebooks | Always specify dimensions |
| Mixing `px` and `go` incorrectly | Confusing code, hard to maintain | Pick one per figure; use `go` for complex |

---

## 4.12 Interview Questions

### Conceptual Questions

**Q1: When would you choose Plotly over Matplotlib?**
> When you need interactivity (zoom, hover, select), web-based dashboards, or need to share charts that non-technical users can explore. Matplotlib is better for publication-quality static figures and when you need pixel-perfect control for papers.

**Q2: What's the difference between Plotly Express and Graph Objects?**
> Plotly Express is a high-level wrapper that creates charts in one function call with automatic defaults. Graph Objects gives full control over every element but requires more code. Express creates Graph Objects figures internally — you can always access/modify the underlying `go.Figure`.

**Q3: How does Plotly render charts in the browser?**
> Plotly.py generates a JSON specification that describes the chart. This JSON is passed to Plotly.js (a JavaScript library), which uses D3.js for SVG rendering and WebGL for large datasets/3D. The rendering happens entirely client-side.

**Q4: How would you handle 1 million data points in Plotly?**
> Options: (1) Use `scattergl` trace type (WebGL, handles ~100K points), (2) Aggregate/bin data before plotting, (3) Use `datashader` for server-side rendering, (4) Sample representative points, (5) Use Plotly's `Scattergl` with `marker.opacity` for overplotting.

**Q5: Explain how Dash callbacks work.**
> Callbacks are Python functions decorated with `@callback`. They take `Input` (triggers) and return values to `Output` (what gets updated). When any Input changes (user selects dropdown, moves slider), the function re-executes and the Output component updates. This creates reactive, interactive dashboards.

### Coding Questions

**Q6: Write code to create a dual-axis chart (bar + line).**
> See Section 4.3 — "Combining Chart Types" example with `yaxis='y2'`.

**Q7: How do you add a play/pause animation to a Plotly chart?**
> Use `animation_frame` in `px.scatter/px.bar` or manually create `frames` in `go.Figure` with `updatemenus` containing play/pause buttons.

---

## 4.13 Quick Reference

### Plotly Express Cheat Sheet

| Chart Type | Function | Key Parameters |
|-----------|----------|----------------|
| Line | `px.line()` | `x, y, color, line_dash` |
| Scatter | `px.scatter()` | `x, y, color, size, symbol, facet_col` |
| Bar | `px.bar()` | `x, y, color, barmode, text, orientation` |
| Histogram | `px.histogram()` | `x, nbins, color, marginal, histnorm` |
| Box | `px.box()` | `x, y, color, points, notched` |
| Violin | `px.violin()` | `x, y, color, box, points` |
| Heatmap | `px.imshow()` | `color_continuous_scale, text_auto` |
| Pie | `px.pie()` | `values, names, hole` (donut) |
| Sunburst | `px.sunburst()` | `path, values, color` |
| Treemap | `px.treemap()` | `path, values, color` |
| Choropleth | `px.choropleth()` | `locations, color, hover_name` |
| 3D Scatter | `px.scatter_3d()` | `x, y, z, color, size` |
| Animation | Any + `animation_frame` | `animation_frame, animation_group` |

### Key Layout Properties

```python
fig.update_layout(
    title='...',                    # Chart title
    template='plotly_white',        # Theme
    width=800, height=500,          # Dimensions
    showlegend=True,                # Toggle legend
    legend=dict(x=0, y=1),         # Legend position
    xaxis_title='...', yaxis_title='...',
    font=dict(family='Arial', size=14),
    margin=dict(l=50, r=50, t=80, b=50),
    hovermode='x unified',         # Unified hover across traces
)
```

### Color Scales

| Type | Examples |
|------|----------|
| Sequential | `'Viridis'`, `'Plasma'`, `'Blues'`, `'Reds'` |
| Diverging | `'RdBu'`, `'RdYlGn'`, `'Picnic'` |
| Qualitative | `px.colors.qualitative.Set2`, `.Plotly`, `.D3` |

---

> **Key Takeaway**: Plotly = interactivity. Use `px` for speed, `go` for control, Dash for dashboards. Always think about whether your audience needs to *explore* the data (→ Plotly) or just *see* a conclusion (→ Matplotlib/Seaborn).
