# Chapter 05: Visualization Best Practices — Chart Selection, Color Theory & Data Storytelling

## Table of Contents
- [5.1 The Philosophy of Good Visualization](#51-the-philosophy-of-good-visualization)
- [5.2 Chart Selection Guide](#52-chart-selection-guide)
- [5.3 Color Theory for Data Visualization](#53-color-theory-for-data-visualization)
- [5.4 Typography & Layout Principles](#54-typography--layout-principles)
- [5.5 Storytelling with Data](#55-storytelling-with-data)
- [5.6 Accessibility in Visualization](#56-accessibility-in-visualization)
- [5.7 Common Visualization Lies & How to Avoid Them](#57-common-visualization-lies--how-to-avoid-them)
- [5.8 Performance & Scalability](#58-performance--scalability)
- [5.9 Production-Ready Visualization Workflow](#59-production-ready-visualization-workflow)
- [5.10 Common Mistakes](#510-common-mistakes)
- [5.11 Interview Questions](#511-interview-questions)
- [5.12 Quick Reference](#512-quick-reference)

---

## 5.1 The Philosophy of Good Visualization

### What It Is
Data visualization isn't about making pretty pictures — it's about **communicating truth clearly**. A great visualization lets the viewer understand the data's story in seconds, while a bad one confuses, misleads, or overwhelms.

### Why It Matters
- 65% of people are visual learners — charts communicate faster than tables
- A misleading chart can cost millions (wrong business decisions) or damage credibility
- The same data can tell completely different stories depending on how you visualize it
- In interviews, *how* you present results matters as much as the analysis itself

### The Data-Ink Ratio (Edward Tufte)

$$\text{Data-Ink Ratio} = \frac{\text{Ink used to display data}}{\text{Total ink used in the graphic}}$$

**Goal**: Maximize this ratio. Every element on your chart should either show data or help interpret it. Remove everything else.

```
BAD (Low data-ink ratio):          GOOD (High data-ink ratio):
┌─────────────────────────┐        
│ ████████████████████████ │        Revenue ($M)
│ ║▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓║ │        ─────────────
│ ║▓▓▓ REVENUE ▓▓▓▓▓▓▓▓║ │        Q1  ████████ 45
│ ║▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓║ │        Q2  ██████████ 52
│ ║  ┌──┐ ┌──┐ ┌──┐    ║ │        Q3  ████████████ 61
│ ║  │45│ │52│ │61│    ║ │        Q4  █████████████ 67
│ ║  └──┘ └──┘ └──┘    ║ │        
│ ║  Q1   Q2   Q3      ║ │        
│ ████████████████████████ │        
└─────────────────────────┘        
  ↑ Borders, gradients,             ↑ Clean, data-focused
    3D effects = noise                 
```

### Five Principles of Effective Visualization

1. **Clarity** → Can the viewer understand the main message in 5 seconds?
2. **Accuracy** → Does the visual encoding faithfully represent the data?
3. **Efficiency** → Is the data-ink ratio high? No chartjunk?
4. **Aesthetics** → Is it pleasant to look at? (But never at the cost of clarity)
5. **Context** → Are axes labeled? Is there a title? Units? Source?

---

## 5.2 Chart Selection Guide

### What It Is
Choosing the right chart type is the **most important** visualization decision. A wrong chart can make clear data confusing, or worse, misleading.

### The Decision Framework

```
What are you trying to show?
│
├── COMPARISON (between items)
│   ├── Few items (< 7) ──────→ Bar Chart (vertical)
│   ├── Many items (7-20) ────→ Bar Chart (horizontal)
│   ├── Over time ────────────→ Line Chart
│   └── Part-to-whole ────────→ Stacked Bar / 100% Stacked Bar
│
├── DISTRIBUTION (of values)
│   ├── Single variable ──────→ Histogram / KDE
│   ├── Compare groups ───────→ Box Plot / Violin Plot
│   ├── Show all points ─────→ Strip Plot / Swarm Plot
│   └── Bivariate ───────────→ 2D Histogram / Contour
│
├── RELATIONSHIP (between variables)
│   ├── Two variables ────────→ Scatter Plot
│   ├── Three variables ──────→ Bubble Chart (size = 3rd var)
│   ├── Many variables ───────→ Heatmap (correlation matrix)
│   └── Paired variables ────→ Parallel Coordinates
│
├── COMPOSITION (parts of whole)
│   ├── Static, few parts ───→ Pie/Donut (≤ 5 slices ONLY)
│   ├── Static, many parts ──→ Treemap / Stacked Bar
│   ├── Over time ───────────→ Stacked Area Chart
│   └── Hierarchical ────────→ Sunburst / Treemap
│
├── TREND (change over time)
│   ├── Single series ────────→ Line Chart
│   ├── Multiple series ─────→ Multi-line / Small Multiples
│   ├── With confidence ─────→ Line + Shaded Band
│   └── Seasonal patterns ───→ Heatmap (calendar)
│
└── SPATIAL (geographic)
    ├── By region ────────────→ Choropleth Map
    ├── Point locations ──────→ Scatter Map
    └── Flow between ─────────→ Flow / Arc Map
```

### Detailed Chart Comparison

| Chart | Best For | Avoid When | Max Items |
|-------|----------|-----------|-----------|
| **Bar** | Comparing categories | Too many categories (>20) | 5-20 |
| **Line** | Trends over continuous axis | Discrete/unordered categories | 5-7 lines |
| **Scatter** | Relationship between 2 variables | One variable is categorical | 10K points |
| **Histogram** | Distribution shape | Comparing many groups | 1-3 overlays |
| **Box Plot** | Summary statistics + outliers | When shape matters (bimodal) | 3-10 groups |
| **Violin** | Full distribution shape | Small sample sizes | 3-8 groups |
| **Heatmap** | Matrix relationships | Few cells (<9) | Up to 50×50 |
| **Pie** | Part-to-whole (simple) | More than 5 slices, comparison | ≤5 slices |
| **Treemap** | Hierarchical proportions | No hierarchy | 50+ items |
| **Area** | Cumulative trends | Overlapping values | 3-5 series |

### Code: Chart Selection Helper

```python
import plotly.express as px
import plotly.graph_objects as go
from plotly.subplots import make_subplots
import pandas as pd
import numpy as np

# Same data, different questions → different charts
np.random.seed(42)
df = pd.DataFrame({
    'Department': np.repeat(['Engineering', 'Sales', 'Marketing', 'HR'], 50),
    'Salary': np.concatenate([
        np.random.normal(95000, 15000, 50),
        np.random.normal(75000, 12000, 50),
        np.random.normal(70000, 10000, 50),
        np.random.normal(65000, 8000, 50)
    ]),
    'Experience': np.random.uniform(1, 20, 200),
    'Performance': np.random.choice(['A', 'B', 'C'], 200, p=[0.3, 0.5, 0.2])
})

# QUESTION 1: "How do salaries compare across departments?"
# → Bar chart (comparison)
fig1 = px.bar(
    df.groupby('Department')['Salary'].mean().reset_index(),
    x='Department', y='Salary',
    title='Q: How do average salaries compare? → Bar Chart',
    color='Department'
)
fig1.show()

# QUESTION 2: "What's the salary distribution in each department?"
# → Violin plot (distribution comparison)
fig2 = px.violin(
    df, x='Department', y='Salary', box=True, points='outliers',
    title='Q: What does the salary distribution look like? → Violin Plot',
    color='Department'
)
fig2.show()

# QUESTION 3: "Does experience predict salary?"
# → Scatter plot (relationship)
fig3 = px.scatter(
    df, x='Experience', y='Salary', color='Department',
    trendline='ols',  # Add regression line
    title='Q: Does experience predict salary? → Scatter + Trendline'
)
fig3.show()

# QUESTION 4: "What proportion of each department has A/B/C performance?"
# → 100% Stacked bar (composition)
perf_df = df.groupby(['Department', 'Performance']).size().reset_index(name='Count')
fig4 = px.bar(
    perf_df, x='Department', y='Count', color='Performance',
    barmode='stack', text='Count',
    title='Q: Performance distribution per dept? → Stacked Bar'
)
fig4.update_layout(barnorm='percent', yaxis_title='Percentage')
fig4.show()
```

### Small Multiples (Faceting) — The Secret Weapon

```python
import plotly.express as px
import numpy as np
import pandas as pd

# When you have too many categories for one chart, use facets
np.random.seed(42)
df = pd.DataFrame({
    'Month': np.tile(range(1, 13), 4),
    'Sales': np.random.uniform(50, 200, 48),
    'Region': np.repeat(['North', 'South', 'East', 'West'], 12)
})

# Small multiples — one panel per region
fig = px.line(
    df, x='Month', y='Sales',
    facet_col='Region',           # One chart per region
    facet_col_wrap=2,             # 2 columns max
    title='Monthly Sales by Region (Small Multiples)',
    markers=True
)

fig.update_layout(height=500)
fig.show()
```

> **Pro Tip**: Small multiples (faceting) are almost always better than cramming 10+ lines onto one chart. The human brain compares shapes across panels more easily than tangled lines.

---

## 5.3 Color Theory for Data Visualization

### What It Is
Color is one of the most powerful — and most misused — tools in visualization. The right color palette makes data instantly understandable. The wrong one makes it confusing or inaccessible.

### Why It Matters
- ~8% of men and ~0.5% of women have color vision deficiency (color blindness)
- Colors carry cultural meaning (red = danger/loss, green = good/growth)
- Wrong color scales can create visual artifacts that don't exist in data
- Color is pre-attentively processed — the brain notices it before reading text

### Three Types of Color Palettes

```
1. SEQUENTIAL (ordered data: low → high)
   ┌──────────────────────────────────────────────┐
   │ ░░░░ ▒▒▒▒ ▓▓▓▓ ████ ████                    │
   │ light ─────────────────────────→ dark         │
   │ Use for: temperature, revenue, count, density │
   └──────────────────────────────────────────────┘
   Examples: Blues, Viridis, Plasma, Inferno

2. DIVERGING (data with meaningful center point)
   ┌──────────────────────────────────────────────┐
   │ ████ ▓▓▓▓ ░░░░ ▒▒▒▒ ████                    │
   │ negative ──── zero ──── positive              │
   │ Use for: correlation, profit/loss, anomalies  │
   └──────────────────────────────────────────────┘
   Examples: RdBu, RdYlGn, Spectral, coolwarm

3. QUALITATIVE/CATEGORICAL (unordered groups)
   ┌──────────────────────────────────────────────┐
   │ ■ ■ ■ ■ ■  (distinct, equal-weight colors)   │
   │ Use for: categories, labels, group membership │
   └──────────────────────────────────────────────┘
   Examples: Set2, Plotly, D3, Tableau10
```

### Color Do's and Don'ts

| Do ✅ | Don't ❌ |
|-------|---------|
| Use sequential for ordered data | Use rainbow/jet for ordered data |
| Use diverging when zero matters | Use sequential for correlation (-1 to +1) |
| Limit categorical colors to 7-10 | Use 20 different colors (indistinguishable) |
| Test with color blindness simulator | Assume everyone sees colors the same |
| Use colorblind-safe palettes | Use red+green to show good+bad |
| Keep background neutral (white/light gray) | Use colored backgrounds |
| Use color consistently across charts | Change color meaning between charts |

### Colorblind-Safe Palettes

```python
import plotly.express as px
import plotly.graph_objects as go
import numpy as np
import pandas as pd

# Colorblind-safe qualitative palette (Wong, 2011)
COLORBLIND_SAFE = [
    '#E69F00',  # Orange
    '#56B4E9',  # Sky blue
    '#009E73',  # Bluish green
    '#F0E442',  # Yellow
    '#0072B2',  # Blue
    '#D55E00',  # Vermillion
    '#CC79A7',  # Reddish purple
    '#000000',  # Black
]

# Using it in Plotly
df = pd.DataFrame({
    'Category': ['A', 'B', 'C', 'D', 'E'],
    'Value': [23, 45, 67, 34, 56]
})

fig = px.bar(
    df, x='Category', y='Value', color='Category',
    color_discrete_sequence=COLORBLIND_SAFE,
    title='Colorblind-Safe Palette Demo'
)
fig.show()
```

### Choosing the Right Colormap — Decision Tree

```python
import plotly.express as px
import numpy as np
import pandas as pd

# RULE: Match colormap to data type

# 1. Sequential data (all positive, ordered)
# → Use: Viridis, Plasma, Blues, Reds
np.random.seed(42)
matrix = np.random.uniform(0, 100, (8, 8))  # All positive values

fig = px.imshow(matrix, color_continuous_scale='Viridis',
                title='Sequential: Viridis (perceptually uniform)')
fig.show()

# 2. Diverging data (has meaningful midpoint, typically 0)
# → Use: RdBu, RdYlGn, coolwarm
correlation = np.random.uniform(-1, 1, (8, 8))
np.fill_diagonal(correlation, 1)

fig = px.imshow(correlation, color_continuous_scale='RdBu_r',
                zmin=-1, zmax=1,  # ALWAYS center diverging at 0!
                title='Diverging: RdBu (centered at 0)')
fig.show()

# 3. Cyclic data (wraps around: angles, time of day, months)
# → Use: HSV, twilight, phase
angles = np.random.uniform(0, 360, (8, 8))

fig = px.imshow(angles, color_continuous_scale='HSV',
                title='Cyclic: HSV (for angular/periodic data)')
fig.show()
```

### Why Rainbow/Jet is Bad

```
The "Rainbow" (Jet) colormap problem:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. NOT perceptually uniform — equal data steps don't look equal
   
   Data:    [1]  [2]  [3]  [4]  [5]  [6]  [7]
   Jet:      🔵   🔵   🟢   🟡   🟡   🟠   🔴
                      ↑              ↑
              These look like big jumps due to
              color channel boundaries, NOT because
              the data has big jumps!

2. Creates false boundaries — artifacts in the yellow/cyan regions
3. Fails in grayscale (printing) — middle values all look same gray
4. Inaccessible to colorblind users

INSTEAD USE: Viridis, Plasma, Inferno (perceptually uniform)
   Data:    [1]  [2]  [3]  [4]  [5]  [6]  [7]
   Viridis:  ■    ■    ■    ■    ■    ■    ■
             Equal perceptual steps → honest representation
```

### Strategic Use of Color

```python
import plotly.express as px
import plotly.graph_objects as go
import pandas as pd
import numpy as np

# Technique: Use color to highlight, not decorate
np.random.seed(42)
df = pd.DataFrame({
    'Company': ['Apple', 'Google', 'Microsoft', 'Amazon', 'Meta',
                'Netflix', 'Tesla', 'Nvidia', 'Adobe', 'Salesforce'],
    'Growth': [12, 8, 15, 22, -5, -8, 45, 38, 10, 7]
})

# Highlight strategy: gray for context, color for story
colors = ['#E63946' if g < 0 else '#2A9D8F' if g > 20 else '#ADB5BD'
          for g in df['Growth']]

fig = go.Figure(go.Bar(
    x=df['Company'],
    y=df['Growth'],
    marker_color=colors,
    text=[f'{g}%' for g in df['Growth']],
    textposition='outside'
))

fig.update_layout(
    title='Annual Growth: <span style="color:#E63946">Declining</span> vs '
          '<span style="color:#2A9D8F">High Growth</span> vs '
          '<span style="color:#ADB5BD">Moderate</span>',
    yaxis_title='Growth (%)',
    template='plotly_white',
    showlegend=False
)
fig.show()
```

> **Key Insight**: The best use of color is **restraint**. Use gray for most data, and reserve bright color for the specific story you want to tell.

---

## 5.4 Typography & Layout Principles

### What It Is
Typography (fonts, sizes, weights) and layout (spacing, alignment, hierarchy) determine whether your visualization is readable or a wall of noise.

### Typographic Hierarchy

```
┌─────────────────────────────────────────────────┐
│                                                   │
│  TITLE (18-24pt, bold)                           │
│  What's the main takeaway?                       │
│                                                   │
│  Subtitle (14-16pt, regular, gray)               │
│  Context, date range, data source                │
│                                                   │
│  ┌─────────────────────────────────────────┐     │
│  │                                         │     │
│  │   Axis Labels (12-14pt)                 │     │
│  │                                         │     │
│  │   Tick Labels (10-12pt)                 │     │
│  │                                         │     │
│  │   Annotations (10-12pt, contextual)     │     │
│  │                                         │     │
│  └─────────────────────────────────────────┘     │
│                                                   │
│  Source: ... (8-10pt, bottom-left)               │
│                                                   │
└─────────────────────────────────────────────────┘
```

### Layout Code Example

```python
import plotly.graph_objects as go
import numpy as np

# Professional layout with proper hierarchy
months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun']
actual = [120, 135, 142, 155, 148, 167]
target = [130, 130, 140, 150, 160, 170]

fig = go.Figure()

fig.add_trace(go.Bar(
    x=months, y=actual, name='Actual',
    marker_color='#2A9D8F'
))
fig.add_trace(go.Scatter(
    x=months, y=target, name='Target',
    mode='lines+markers',
    line=dict(color='#E76F51', width=2, dash='dash')
))

fig.update_layout(
    # Title hierarchy
    title=dict(
        text='<b>Revenue Exceeds Target in Q2</b>'    # Bold title
             '<br><span style="font-size:13px;color:gray">'
             'Monthly revenue vs target, Jan-Jun 2024</span>',
        x=0.02,                                        # Left-aligned (not centered!)
        font=dict(size=18)
    ),

    # Clean axes
    xaxis=dict(
        title=None,                                    # Remove if obvious
        tickfont=dict(size=12)
    ),
    yaxis=dict(
        title='Revenue ($M)',
        titlefont=dict(size=13),
        tickfont=dict(size=11),
        gridcolor='#EEEEEE',                          # Subtle grid
        zeroline=False
    ),

    # Legend
    legend=dict(
        orientation='h',                               # Horizontal legend
        yanchor='bottom', y=1.02,
        xanchor='left', x=0,
        font=dict(size=11)
    ),

    # Overall
    template='plotly_white',
    font=dict(family='Segoe UI, Arial, sans-serif'),  # Professional font
    margin=dict(l=60, r=30, t=100, b=40),
    width=700, height=400,

    # Add source annotation
    annotations=[dict(
        text='Source: Internal Finance Dashboard',
        x=0, y=-0.15,
        xref='paper', yref='paper',
        showarrow=False,
        font=dict(size=9, color='gray')
    )]
)

fig.show()
```

### Layout Rules

| Rule | Why |
|------|-----|
| Left-align titles | We read left-to-right; centered titles float disconnected |
| Remove axis title if obvious | "Month" label under Jan/Feb/Mar is redundant |
| Use horizontal legends above chart | Doesn't steal space from the plot area |
| Keep grid lines subtle (light gray) | They guide the eye without dominating |
| Consistent margins | Charts in a report should align |
| White/light background | Maximizes contrast for data |

---

## 5.5 Storytelling with Data

### What It Is
Data storytelling is combining **data + visuals + narrative** to drive decisions. It's not about showing all the data — it's about guiding the viewer to an insight.

### Why It Matters
- Data scientists who can tell stories get their recommendations implemented
- Executives don't have time to explore — they need conclusions with evidence
- A story makes data memorable (65% retention vs. 5% for stats alone)

### The Three-Act Structure for Data Stories

```
ACT 1: SETUP (Context)
────────────────────────
"Here's where we are"
- Show the baseline/status quo
- Set expectations
- Define what "normal" looks like

ACT 2: CONFLICT (Insight)
────────────────────────
"Here's what changed / what's surprising"
- Show the deviation, trend, anomaly
- This is your KEY INSIGHT
- Use color/annotation to highlight it

ACT 3: RESOLUTION (Action)
────────────────────────
"Here's what we should do"
- Recommend actions
- Show projected outcomes
- Call to action
```

### Annotation-Driven Storytelling

```python
import plotly.graph_objects as go
import pandas as pd
import numpy as np

# Story: "Our churn rate spiked after the price increase"
months = pd.date_range('2023-01-01', periods=12, freq='MS')
churn = [3.2, 3.1, 3.4, 3.0, 3.3, 3.1,   # Normal period
         5.8, 7.2, 6.5, 5.9, 4.8, 4.2]    # After price increase

fig = go.Figure()

# Plot the line
fig.add_trace(go.Scatter(
    x=months, y=churn,
    mode='lines+markers',
    line=dict(color='#264653', width=3),
    marker=dict(size=8)
))

# ACT 1: Show the normal baseline
fig.add_hrect(y0=2.5, y1=3.8,
              fillcolor='green', opacity=0.08,
              annotation_text='Normal range (3-3.5%)',
              annotation_position='top left')

# ACT 2: Highlight the event
fig.add_vline(x='2023-07-01', line_dash='dash', line_color='red',
              annotation_text='Price increase<br>(+20%)',
              annotation_position='top right',
              annotation_font=dict(color='red', size=12))

# ACT 2: Annotate the peak
fig.add_annotation(
    x='2023-08-01', y=7.2,
    text='<b>Peak: 7.2%</b><br>2.3x normal churn!',
    showarrow=True, arrowhead=2,
    ax=40, ay=-40,
    font=dict(color='#E63946', size=12),
    bordercolor='#E63946', borderwidth=1, bgcolor='#FFF5F5'
)

# ACT 3: Show recovery
fig.add_annotation(
    x='2023-12-01', y=4.2,
    text='Recovery trend<br>(still above baseline)',
    showarrow=True, arrowhead=2,
    ax=-50, ay=30,
    font=dict(color='#457B9D', size=11)
)

fig.update_layout(
    title=dict(
        text='<b>Churn Spiked 2.3x After July Price Increase</b>'
             '<br><span style="color:gray;font-size:12px">'
             'Monthly customer churn rate, 2023</span>',
        x=0.02
    ),
    yaxis=dict(title='Churn Rate (%)', range=[0, 8.5]),
    xaxis=dict(title=None, dtick='M1', tickformat='%b'),
    template='plotly_white',
    height=450, width=750,
    showlegend=False
)
fig.show()
```

### Before & After: Turning Data Dumps into Stories

```python
import plotly.graph_objects as go
import plotly.express as px
import pandas as pd
import numpy as np

# ❌ BAD: Data dump — shows everything, says nothing
np.random.seed(42)
products = [f'Product {i}' for i in range(1, 21)]
revenue = np.random.uniform(10, 200, 20)

# This is what most beginners create — no story, no insight
fig_bad = px.bar(
    x=products, y=revenue,
    title='Product Revenue'  # Vague title
)
# fig_bad.show()  # All bars same color, no ordering, no context

# ✅ GOOD: Insight-driven — highlights the story
df = pd.DataFrame({'Product': products, 'Revenue': revenue})
df = df.sort_values('Revenue', ascending=True)  # Sort for readability!

# Color only the top 3 and bottom 3
colors = ['#E63946' if r < df['Revenue'].quantile(0.15)
          else '#2A9D8F' if r > df['Revenue'].quantile(0.85)
          else '#D3D3D3'
          for r in df['Revenue']]

fig_good = go.Figure(go.Bar(
    x=df['Revenue'],
    y=df['Product'],
    orientation='h',
    marker_color=colors
))

fig_good.update_layout(
    title=dict(
        text='<b>3 Products Drive 45% of Revenue; 3 Need Attention</b>'
             '<br><span style="font-size:12px;color:gray">'
             'Product revenue, sorted. '
             '<span style="color:#2A9D8F">■ Top performers</span> | '
             '<span style="color:#E63946">■ Underperformers</span></span>',
        x=0.02
    ),
    xaxis_title='Revenue ($M)',
    yaxis_title=None,
    template='plotly_white',
    height=600
)
fig_good.show()
```

### Headline-First Approach

> **Rule**: Your chart title should be the **insight**, not the description.

| ❌ Descriptive Title | ✅ Insight Title |
|---------------------|-----------------|
| "Monthly Revenue 2023" | "Revenue Grew 40% After Campaign Launch" |
| "Customer Satisfaction Scores" | "Satisfaction Dropped to 3-Year Low in Q3" |
| "Sales by Region" | "West Region Outperforms All Others by 2x" |
| "User Engagement Metrics" | "Mobile Users Engage 3x More Than Desktop" |

---

## 5.6 Accessibility in Visualization

### What It Is
Making visualizations usable for **everyone** — including people with color vision deficiency, low vision, cognitive differences, or using screen readers.

### Why It Matters
- 300 million people worldwide have color vision deficiency
- Legal requirements (ADA, WCAG) for public-facing content
- Accessible design is usually just *better* design for everyone

### Color Blindness Types

```
Normal Vision:     Red ● Green ● Blue ● Yellow ● Purple ●

Deuteranopia       ●     ●     ●      ●       ●
(~6% of men):    (brown) (brown) (blue) (yellow) (blue)
                  RED AND GREEN LOOK THE SAME!

Protanopia         ●     ●     ●      ●       ●  
(~2% of men):   (dark)  (tan)  (blue) (yellow) (blue)

Tritanopia         ●     ●     ●      ●       ●
(rare):         (red)  (green) (dark) (pink)  (red)
```

### Accessibility Techniques

```python
import plotly.graph_objects as go
import numpy as np

x = np.linspace(0, 10, 50)

fig = go.Figure()

# Technique 1: Use BOTH color AND pattern/dash/marker shape
fig.add_trace(go.Scatter(
    x=x, y=np.sin(x), name='Model A',
    mode='lines+markers',
    line=dict(color='#0072B2', width=3, dash='solid'),   # Blue, solid
    marker=dict(symbol='circle', size=6)
))

fig.add_trace(go.Scatter(
    x=x, y=np.cos(x), name='Model B',
    mode='lines+markers',
    line=dict(color='#D55E00', width=3, dash='dash'),    # Orange, dashed
    marker=dict(symbol='square', size=6)
))

fig.add_trace(go.Scatter(
    x=x, y=np.sin(x + 1), name='Model C',
    mode='lines+markers',
    line=dict(color='#009E73', width=3, dash='dot'),     # Green, dotted
    marker=dict(symbol='diamond', size=6)
))

fig.update_layout(
    title='<b>Accessible Chart: Color + Shape + Dash</b>',
    template='plotly_white',
    font=dict(size=14),          # Technique 2: Larger font
    legend=dict(font=dict(size=13))
)
fig.show()
```

### Accessibility Checklist

| Check | Why |
|-------|-----|
| ✅ Don't rely on color alone | Use shape, pattern, labels, position too |
| ✅ Use high contrast (4.5:1 ratio) | Readable for low-vision users |
| ✅ Minimum font size 11-12pt | Readable without zooming |
| ✅ Add alt text for web charts | Screen reader support |
| ✅ Direct label lines (not just legend) | Reduces cognitive load for everyone |
| ✅ Test with color blindness simulator | Coblis, Viz Palette tools |
| ✅ Provide data table alternative | Some users prefer raw data |

---

## 5.7 Common Visualization Lies & How to Avoid Them

### What It Is
Charts can accidentally (or intentionally) mislead. Understanding these tricks helps you avoid them in your own work and spot them in others'.

### The Classic Lies

#### Lie #1: Truncated Y-Axis

```python
import plotly.graph_objects as go
from plotly.subplots import make_subplots

months = ['Jan', 'Feb', 'Mar', 'Apr']
values = [92, 94, 93, 95]

fig = make_subplots(rows=1, cols=2,
                    subplot_titles=('❌ MISLEADING (truncated axis)',
                                    '✅ HONEST (starts at 0)'))

# Misleading — starts at 90, makes small change look huge
fig.add_trace(go.Bar(x=months, y=values, marker_color='steelblue'),
              row=1, col=1)
fig.update_yaxes(range=[90, 96], row=1, col=1)

# Honest — starts at 0, shows true proportion
fig.add_trace(go.Bar(x=months, y=values, marker_color='steelblue'),
              row=1, col=2)
fig.update_yaxes(range=[0, 100], row=1, col=2)

fig.update_layout(
    title='Lie #1: Truncated Y-Axis Makes 3% Change Look Like 50%',
    showlegend=False, height=400, template='plotly_white'
)
fig.show()
```

> **When truncation is OK**: Line charts showing trends (stock prices, temperature) where relative change matters more than absolute level. NEVER truncate bar charts — bar length encodes absolute value.

#### Lie #2: Cherry-Picked Time Range

```python
import plotly.graph_objects as go
import numpy as np
import pandas as pd

# Full story: growth then decline
np.random.seed(42)
dates = pd.date_range('2022-01-01', periods=24, freq='MS')
values = list(range(50, 62)) + list(range(62, 50, -1))  # Up then down

fig = make_subplots(rows=1, cols=2,
                    subplot_titles=('❌ Cherry-picked: "We\'re growing!"',
                                    '✅ Full picture: Growth reversed'))

# Only show the growth period
fig.add_trace(go.Scatter(x=dates[:12], y=values[:12], mode='lines',
                          line=dict(color='green', width=3)), row=1, col=1)

# Show everything
fig.add_trace(go.Scatter(x=dates, y=values, mode='lines',
                          line=dict(color='#264653', width=3)), row=1, col=2)
fig.add_vline(x='2023-01-01', line_dash='dash', row=1, col=2)

fig.update_layout(title='Lie #2: Cherry-Picked Time Range',
                  height=350, template='plotly_white', showlegend=False)
fig.show()
```

#### Lie #3: Dual Axes Manipulation

```
Dual axes can imply correlation where none exists:

Revenue ($M)          Temperature (°F)
100 ┤                          ┤ 80
 80 ┤    ╱─╲    Revenue       ┤ 70
 60 ┤  ╱     ╲──────────     ┤ 60
 40 ┤╱         ╲   Temp      ┤ 50
 20 ┤             ╲──────    ┤ 40
    └─────────────────────────

"Revenue correlates with temperature!"
Actually: The second axis was scaled to make them align.
You can make ANY two lines look correlated by adjusting axis ranges.

FIX: Normalize both to same scale, or use separate panels.
```

#### Lie #4: Area/Volume Encoding

```
When using circles/icons sized by value:

Value doubles (2x):
Area should double:    ● → ●●  (radius × √2 = 1.41x)
NOT radius doubles:    ● → ⬤    (area is now 4x — exaggerates!)

This is why Plotly uses `sizemode='area'` by default in bubble charts.
Always encode quantity as AREA, not radius/diameter.
```

### Summary of Visualization Ethics

| Principle | Application |
|-----------|-------------|
| Show zero on bar charts | Don't truncate to exaggerate |
| Show full time range | Don't cherry-pick flattering periods |
| Use consistent scales | Don't manipulate dual axes |
| Area = quantity | Don't use radius for proportional symbols |
| Label axes & units | Don't leave viewers guessing |
| Show uncertainty | Don't hide error bars or confidence intervals |
| Cite data source | Don't present data without provenance |

---

## 5.8 Performance & Scalability

### What It Is
When your dataset has thousands to millions of points, naive plotting will freeze your browser or create unreadable blobs. Production visualization requires performance strategies.

### Data Size Recommendations

| Points | Strategy | Tool |
|--------|----------|------|
| < 1,000 | Plot everything | Any (Plotly, Matplotlib) |
| 1K - 50K | Plot everything, use WebGL | `Scattergl` in Plotly |
| 50K - 500K | Aggregate/sample | Binning, random sampling |
| 500K - 5M | Server-side rendering | Datashader, deck.gl |
| > 5M | Pre-aggregate in DB | SQL aggregation → plot summary |

### WebGL for Large Scatter Plots

```python
import plotly.graph_objects as go
import numpy as np

# 100K points — use Scattergl (WebGL) instead of Scatter (SVG)
np.random.seed(42)
n = 100_000
x = np.random.randn(n)
y = x * 0.5 + np.random.randn(n) * 0.8

# ❌ SLOW: go.Scatter (SVG — each point is a DOM element)
# fig = go.Figure(go.Scatter(x=x, y=y, mode='markers'))  # Freezes browser!

# ✅ FAST: go.Scattergl (WebGL — GPU-rendered)
fig = go.Figure(go.Scattergl(
    x=x, y=y,
    mode='markers',
    marker=dict(size=2, color=y, colorscale='Viridis', opacity=0.3)
))

fig.update_layout(
    title=f'100K Points with WebGL (Scattergl)',
    template='plotly_white',
    width=700, height=500
)
fig.show()
```

### Aggregation Strategies

```python
import plotly.express as px
import pandas as pd
import numpy as np

# Scenario: 1M transaction records — can't plot each one
np.random.seed(42)
n = 1_000_000
df = pd.DataFrame({
    'timestamp': pd.date_range('2023-01-01', periods=n, freq='min'),
    'amount': np.random.exponential(50, n),
    'category': np.random.choice(['Food', 'Transport', 'Shopping'], n)
})

# Strategy 1: Temporal aggregation (1M rows → 365 daily points)
daily = df.set_index('timestamp').resample('D')['amount'].agg(['sum', 'count']).reset_index()

fig = px.line(daily, x='timestamp', y='sum',
              title='Daily Transaction Volume (Aggregated from 1M records)',
              labels={'sum': 'Daily Total ($)', 'timestamp': 'Date'})
fig.show()

# Strategy 2: 2D Histogram instead of scatter (shows density)
fig2 = px.density_heatmap(
    df.sample(50000),  # Still sample for speed
    x='timestamp', y='amount',
    nbinsx=100, nbinsy=50,
    color_continuous_scale='Viridis',
    title='Transaction Amount Distribution Over Time (Density)'
)
fig2.show()

# Strategy 3: Statistical summary per category
summary = df.groupby('category')['amount'].describe().reset_index()
fig3 = px.box(df.sample(10000), x='category', y='amount',
              title='Transaction Amount by Category (10K sample)')
fig3.show()
```

### Performance Tips

| Tip | Implementation |
|-----|---------------|
| Use WebGL traces | `go.Scattergl`, `go.Heatmapgl` |
| Reduce point count | Sample, aggregate, or bin data |
| Simplify markers | `marker.size=2`, no borders |
| Limit hover data | Use `hoverinfo='skip'` for background traces |
| Use `fig.write_html(full_html=False)` | Smaller file for embedding |
| Pre-compute layouts | Cache expensive figure objects |
| Use CDN for Plotly.js | `include_plotlyjs='cdn'` in HTML export |

---

## 5.9 Production-Ready Visualization Workflow

### What It Is
A systematic approach to creating visualizations that are consistent, maintainable, and professional — the difference between a Jupyter notebook experiment and a deliverable.

### The Workflow

```
1. UNDERSTAND → Who is the audience? What decision will this inform?
2. EXPLORE    → Quick EDA plots (ugly is fine here)
3. DESIGN     → Sketch the layout. Choose chart type. Plan the story.
4. BUILD      → Create the visualization with clean code
5. REFINE     → Polish: colors, fonts, annotations, accessibility
6. VALIDATE   → Does it accurately represent the data? Peer review.
7. DELIVER    → Export in appropriate format for the audience
```

### Reusable Chart Template

```python
import plotly.graph_objects as go
import plotly.io as pio

def create_branded_figure(title, subtitle='', width=750, height=450):
    """Create a figure with consistent corporate branding."""
    fig = go.Figure()

    fig.update_layout(
        title=dict(
            text=f'<b>{title}</b>'
                 f'<br><span style="font-size:12px;color:#666">{subtitle}</span>',
            x=0.02, xanchor='left',
            font=dict(size=17)
        ),
        font=dict(family='Segoe UI, Arial, sans-serif', size=12, color='#333'),
        template='plotly_white',
        width=width, height=height,
        margin=dict(l=60, r=30, t=90, b=60),
        legend=dict(
            orientation='h',
            yanchor='bottom', y=1.02,
            xanchor='left', x=0
        ),
        colorway=['#2A9D8F', '#E76F51', '#264653', '#E9C46A',
                  '#F4A261', '#606C38', '#BC6C25'],
        xaxis=dict(gridcolor='#F0F0F0', linecolor='#CCC'),
        yaxis=dict(gridcolor='#F0F0F0', linecolor='#CCC', zeroline=False),
    )
    return fig


def add_source(fig, source_text, y_offset=-0.12):
    """Add a source attribution to the bottom of the chart."""
    fig.add_annotation(
        text=f'Source: {source_text}',
        x=0, y=y_offset,
        xref='paper', yref='paper',
        showarrow=False,
        font=dict(size=9, color='#999'),
        xanchor='left'
    )
    return fig


# Usage
fig = create_branded_figure(
    'Q2 Revenue Exceeded Expectations',
    'Quarterly comparison, FY2024 | All figures in $M'
)

fig.add_trace(go.Bar(
    x=['Q1', 'Q2', 'Q3', 'Q4'],
    y=[142, 178, 155, None],
    name='Actual'
))
fig.add_trace(go.Scatter(
    x=['Q1', 'Q2', 'Q3', 'Q4'],
    y=[150, 160, 170, 180],
    name='Target',
    mode='lines+markers',
    line=dict(dash='dash')
))

add_source(fig, 'Finance Team, July 2024')
fig.show()
```

### Export for Different Audiences

```python
import plotly.graph_objects as go

# Same figure, different outputs based on audience:

# For EXECUTIVES (email/presentation):
# fig.write_image('revenue.png', width=1200, height=600, scale=2)
# → High-res static image, no interactivity needed

# For DATA TEAM (exploration):
# fig.write_html('revenue.html')
# → Interactive, can zoom/hover

# For WEB APP (embedding):
# fig.write_html('revenue.html', include_plotlyjs='cdn', full_html=False)
# → Lightweight HTML div for embedding

# For PAPER/PUBLICATION:
# fig.write_image('revenue.svg')
# → Vector format, scales perfectly at any size

# For DASHBOARD:
# → Use Dash, return the figure object directly in callback
```

---

## 5.10 Common Mistakes

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using pie charts for comparison | Humans can't compare angles accurately | Use bar charts |
| Rainbow colormap (jet/spectral) | Creates false patterns, not colorblind-safe | Use Viridis, Plasma |
| No chart title or vague title | Viewer doesn't know the takeaway | Use insight-as-title |
| Too many colors (>7 categories) | Indistinguishable, overwhelming | Use gray + highlight, or facets |
| 3D bar charts | Impossible to read values accurately | Use 2D bars |
| Inconsistent scales across panels | Viewers assume same scale = comparable | Normalize or label clearly |
| Decorative chart elements | Chartjunk reduces data-ink ratio | Remove borders, gradients, shadows |
| Not ordering categories | Random order makes comparison harder | Sort by value |
| Using defaults without thought | Every library has bad defaults | Always customize template |
| Showing too much data | Cognitive overload | Aggregate, sample, or facet |
| Not considering the audience | Technical chart for executives | Adapt complexity to viewer |
| Missing units on axes | "$45" vs "45 thousand" is crucial | Always include units |

---

## 5.11 Interview Questions

### Conceptual Questions

**Q1: How do you choose between a bar chart and a line chart?**
> Bar charts are for **categorical comparisons** (departments, products). Line charts are for **continuous trends** (time series). Using a line chart for categories implies a connection between points that doesn't exist. Using a bar chart for time series makes it hard to see the trend.

**Q2: What makes a visualization "misleading"? Give 3 examples.**
> (1) Truncated y-axis on bar charts exaggerates differences. (2) Cherry-picked time ranges hide unfavorable trends. (3) Dual y-axes with manipulated scales create false correlations. Also: 3D effects that distort proportions, area/volume misuse, and unlabeled axes.

**Q3: How would you visualize data for a colorblind audience?**
> Use colorblind-safe palettes (e.g., Viridis, Wong palette). Never rely on color alone — add shape, pattern, position, or direct labels. Use sufficient contrast. Test with simulation tools (Coblis, Chrome DevTools). Provide alt-text for web.

**Q4: Explain the data-ink ratio. Why does it matter?**
> Coined by Edward Tufte: it's the proportion of ink on a chart that represents actual data vs. total ink (including decoration). Maximizing it means removing chartjunk (unnecessary gridlines, borders, 3D effects, redundant legends). Higher data-ink ratio → clearer, faster comprehension.

**Q5: You have a dataset with 50 categories. How do you visualize it?**
> Options: (1) Show only top/bottom 10 with "Others" grouped. (2) Use a treemap for part-to-whole. (3) Horizontal bar chart sorted by value (allows reading labels). (4) Small multiples if comparing trends. (5) Interactive chart with search/filter. Never use a pie chart with 50 slices.

**Q6: What's wrong with using red/green to indicate good/bad?**
> ~8% of men have red-green color blindness (deuteranopia/protanopia). They can't distinguish between "good" and "bad." Instead: use blue/orange, add icons (✓/✗), or use value + intensity (dark = good, light = bad in same hue).

**Q7: How do you handle visualization of a million data points?**
> (1) Aggregate/bin data before plotting. (2) Use density plots (heatmap, hexbin) instead of scatter. (3) Use WebGL rendering (Scattergl in Plotly). (4) Random sampling with seed for reproducibility. (5) Server-side rendering (Datashader). The goal: show the PATTERN, not every point.

**Q8: When is a table better than a chart?**
> When: (1) Exact values matter (financial reports). (2) Few data points (<5 rows). (3) Audience needs to look up specific values. (4) Multiple units that don't share an axis. Charts are better for: patterns, trends, comparisons, outliers, distributions.

### Scenario Questions

**Q9: A stakeholder says "just make a dashboard with all our KPIs." What do you do?**
> (1) Ask: "What decisions will this inform?" (2) Prioritize — put the most important 3-5 KPIs prominently. (3) Use hierarchy: big number for current value, small sparkline for trend. (4) Group related metrics. (5) Add context (targets, benchmarks, time comparisons). (6) Design for the least technical viewer.

**Q10: Your scatter plot has 10,000 overlapping points. How do you fix it?**
> Options by preference: (1) Reduce opacity (`opacity=0.1`). (2) Use 2D density/heatmap. (3) Use hexbin plot. (4) Add jitter for categorical overlaps. (5) Sample if overplotting is extreme. (6) Use contour plot for bivariate density. Choice depends on what insight you're trying to show.

---

## 5.12 Quick Reference

### Chart Selection Decision Matrix

| Your Question | Chart Type | Example |
|---------------|-----------|---------|
| How much? (compare values) | Bar | Revenue by region |
| How has it changed? (trend) | Line | Monthly active users |
| What's the relationship? | Scatter | Height vs weight |
| How is it distributed? | Histogram/Violin | Income distribution |
| What's the composition? | Stacked Bar/Treemap | Budget allocation |
| Where? (geographic) | Choropleth/Scatter map | Sales by state |
| How do parts relate to whole? | Pie (≤5) / Sunburst | Market share |
| What's the correlation? | Heatmap | Feature correlations |
| How do groups compare on many dimensions? | Radar / Parallel coords | Player stats |

### Color Quick Reference

| Data Type | Palette Type | Examples |
|-----------|-------------|----------|
| Ordered (low→high) | Sequential | Viridis, Blues, Plasma |
| Centered (neg→0→pos) | Diverging | RdBu, coolwarm |
| Categories (unordered) | Qualitative | Set2, Tableau10 |
| Highlight (story) | Gray + 1 accent color | Gray + Red |
| Accessible | Colorblind-safe | Wong palette, Viridis |

### The Golden Rules

| # | Rule |
|---|------|
| 1 | Title = insight, not description |
| 2 | Sort categories by value (not alphabetically) |
| 3 | Start bar chart y-axis at 0 |
| 4 | Use color sparingly and intentionally |
| 5 | Remove chartjunk (borders, 3D, gradients) |
| 6 | Label directly (reduce legend dependence) |
| 7 | One chart = one message |
| 8 | Consider your audience's technical level |
| 9 | Test accessibility (color blind, low vision) |
| 10 | Show uncertainty (error bars, confidence bands) |

### Tool Selection Guide

| Need | Tool | Why |
|------|------|-----|
| Quick EDA | Seaborn / Plotly Express | Fast, good defaults |
| Publication figures | Matplotlib | Pixel-perfect, LaTeX support |
| Interactive exploration | Plotly | Zoom, hover, select |
| Dashboards | Dash / Streamlit | Web-based, callbacks |
| Big data (>1M pts) | Datashader + HoloViews | Server-side aggregation |
| Presentation slides | Plotly (export PNG) | Clean, high-res |
| Web embedding | Plotly (HTML) / D3.js | Self-contained |

---

> **Final Takeaway**: The best visualization is one that makes the viewer say "Oh, I see it!" in under 5 seconds. Everything else — colors, fonts, chart types — is in service of that moment of understanding. Master the "what to show" before obsessing over the "how to show it."
