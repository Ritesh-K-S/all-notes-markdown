# Chapter 10: Data Storytelling and Communication

## Table of Contents
- [What is Data Storytelling?](#what-is-data-storytelling)
- [Why It Matters](#why-it-matters)
- [The Anatomy of a Data Story](#the-anatomy-of-a-data-story)
- [Know Your Audience](#know-your-audience)
- [Visualization Principles](#visualization-principles)
- [Building Effective Dashboards](#building-effective-dashboards)
- [Presenting to Stakeholders](#presenting-to-stakeholders)
- [Python Visualization Code](#python-visualization-code)
- [Writing Data Reports](#writing-data-reports)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Data Storytelling?

### Simple Explanation

You've done the analysis. You found an insight. Now what?

**Data storytelling** is the skill of turning numbers into decisions. It's the bridge between "I found something interesting in the data" and "the company actually does something about it."

Think of it like a courtroom: The data is the evidence, your analysis is the investigation, but your presentation is the closing argument that convinces the jury (stakeholders) to act.

```
┌─────────────────────────────────────────────────────────────┐
│                                                               │
│   RAW DATA          ANALYSIS           DATA STORY            │
│                                                               │
│   "547,231 rows     "Churn increased    "We're losing our    │
│    of customer       12% in Q3 among     best customers       │
│    transactions"     high-value users     because of slow      │
│                      who experienced      support response.    │
│                      >48hr support        Fixing this = $2M    │
│                      response times"      saved annually"      │
│                                                               │
│   Nobody cares ──▶  Interesting ──────▶  ACTIONABLE           │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### The Three Pillars

```
                    DATA STORY
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
      ┌──────────┐ ┌──────────┐ ┌──────────┐
      │   DATA   │ │ VISUALS  │ │NARRATIVE │
      │          │ │          │ │          │
      │ Evidence │ │ Charts,  │ │ Context, │
      │ Numbers  │ │ Graphs,  │ │ Meaning, │
      │ Facts    │ │ Tables   │ │ Action   │
      └──────────┘ └──────────┘ └──────────┘
      
      Alone: boring   Alone: pretty   Alone: opinion
      Together: POWERFUL and PERSUASIVE
```

---

## Why It Matters

### The Hard Truth

> "The best analysis in the world is worthless if nobody understands it or acts on it."

| Scenario | Poor Communication | Great Communication |
|----------|-------------------|---------------------|
| You find a churn signal | "p < 0.05, coefficient = -0.34" | "We'll lose $2M next quarter. Here's the fix." |
| Model improves accuracy | "AUC increased from 0.82 to 0.87" | "We'll catch 500 more fraud cases/month, saving $1.2M" |
| A/B test wins | "Treatment had 3.2% lift, p=0.003" | "New checkout flow adds $4.5M annual revenue. Ship it." |

### Career Impact

- Data scientists who communicate well get promoted 2x faster
- Your analysis quality is judged by how well people understand it
- The best technical work is invisible if poorly presented
- Executives decide budgets based on presentations, not notebooks

---

## The Anatomy of a Data Story

### The SCA Framework

Every data story should follow this structure:

```
┌─────────────────────────────────────────────────────────────┐
│                                                               │
│  S — SITUATION (Context)                                     │
│      "Here's where we are and why this matters"              │
│      • Background and business context                       │
│      • What question we were trying to answer                │
│                                                               │
│  C — COMPLICATION (Insight)                                  │
│      "Here's what we found (the interesting part)"           │
│      • Key finding, supported by evidence                    │
│      • What changed, what's surprising, what's at risk       │
│                                                               │
│  A — ACTION (Recommendation)                                 │
│      "Here's what we should do about it"                     │
│      • Specific, actionable recommendation                   │
│      • Expected impact (quantified)                          │
│      • Timeline and resources needed                         │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Example: Complete Data Story

```
❌ BAD (Data Dump):
"Our logistic regression model on the customer dataset showed that 
tenure (p<0.001, coeff=-0.023), monthly charges (p<0.001, coeff=0.031),
and contract type (p<0.001, coeff=-1.45) are significant predictors of 
churn. The model AUC is 0.84 with precision 0.72 and recall 0.68."

✓ GOOD (Data Story):

SITUATION: "We're losing 26% of customers annually — that's $18M in 
lost revenue. Leadership asked: who's leaving and why?"

COMPLICATION: "Our analysis reveals that 73% of churners share three 
traits: month-to-month contracts, less than 12 months tenure, and 
charges over $70/month. These customers are 5x more likely to leave. 
The #1 trigger? Their first support call going unresolved."

ACTION: "We recommend three actions:
1. Auto-flag at-risk customers (model identifies them with 84% accuracy)
2. Proactive outreach at month 3 with loyalty offer (estimated 15% 
   retention improvement = $2.7M saved)
3. Priority support routing for high-risk customers
Implementation: 3 weeks. Expected ROI: 9x in first year."
```

---

## Know Your Audience

### Audience Matrix

| Audience | What They Care About | How to Present | Avoid |
|----------|---------------------|----------------|-------|
| **C-Suite (CEO, CFO)** | Revenue impact, strategic decisions | 1-3 slides, $$ impact, recommendations | Technical jargon, methodology details |
| **VP/Director** | Department KPIs, resource allocation | 5-10 slides, trends, actionable insights | Code, raw statistics |
| **Product Manager** | User behavior, feature impact, prioritization | Metrics tied to features, trade-offs | Irrelevant technical depth |
| **Engineering** | Implementation feasibility, system impact | Technical specs, data schemas, APIs | Business fluff |
| **Other Data Scientists** | Methodology, reproducibility, validity | Code, statistical details, assumptions | Over-simplification |

### The Pyramid Principle

Structure your communication like an inverted pyramid:

```
         ┌─────────────────────┐
         │   KEY TAKEAWAY      │  ← Lead with this (everyone gets this)
         │   (1 sentence)      │
         └──────────┬──────────┘
                    │
         ┌──────────┴──────────┐
         │  SUPPORTING INSIGHTS │  ← For those who want more
         │  (3-5 key points)    │
         └──────────┬──────────┘
                    │
    ┌───────────────┴───────────────┐
    │     DETAILED ANALYSIS          │  ← For those who want proof
    │  (methodology, edge cases,     │
    │   caveats, appendix)           │
    └────────────────────────────────┘
    
    CEO reads top. Analyst reads everything. Both are satisfied.
```

### Adapting Language

```
Same insight, different audiences:

FOR CEO:
"We're leaving $4M on the table. Customers who receive 
personalized recommendations spend 23% more. 
Recommendation: Invest $500K in personalization engine. 
Expected ROI: 8x in 12 months."

FOR PRODUCT MANAGER:
"Users who see personalized product recommendations have 23% higher 
AOV and 15% higher session duration. The strongest signal comes from 
'recently viewed' items. Recommend adding a 'Continue shopping' 
section on the homepage — estimated 2-sprint implementation."

FOR DATA SCIENCE TEAM:
"Collaborative filtering (ALS, k=50) outperforms content-based 
by 12% on NDCG@10. Cold-start handled via popularity fallback. 
Model retrains nightly on last 90 days of interactions. 
A/B test showed +23% AOV (p<0.001, n=50K per group)."
```

---

## Visualization Principles

### Choose the Right Chart

| Question | Chart Type | Why |
|----------|-----------|-----|
| How much? (comparison) | Bar chart | Easy to compare lengths |
| How has it changed? (trend) | Line chart | Shows time progression |
| What's the breakdown? (composition) | Stacked bar, pie (if ≤5 parts) | Shows parts of whole |
| How are they related? (relationship) | Scatter plot | Shows correlation |
| How is it distributed? (distribution) | Histogram, box plot | Shows spread and shape |
| Where? (geographic) | Map / choropleth | Spatial patterns |

### The Chart Selection Flowchart

```
What are you showing?
│
├─── Comparison ─────── Among items ──── Bar chart (horizontal if many items)
│                   └── Over time ────── Line chart
│
├─── Composition ────── Static ─────────── Stacked bar / Treemap / Pie (≤5)
│                   └── Over time ────── Stacked area chart
│
├─── Distribution ───── Single variable ── Histogram / KDE / Box plot
│                   └── Two variables ─── Scatter / Heatmap
│
├─── Relationship ───── Two variables ─── Scatter plot
│                   └── Three+ ─────────── Bubble chart / Pair plot
│
└─── Trend ──────────── Simple ──────────── Line chart
                    └── With target ───── Line + reference line
```

### Design Principles (Tufte's Rules)

1. **Maximize data-ink ratio** — Remove everything that doesn't convey information
2. **No chartjunk** — No 3D effects, no decorative elements, no unnecessary gridlines
3. **Label directly** — Put labels on the data, not in legends
4. **Use color purposefully** — Highlight what matters, gray out the rest
5. **Show comparisons** — Always provide context (benchmark, target, previous period)

### Before and After

```
❌ BAD CHART:
- 3D pie chart with 12 slices
- Rainbow colors
- Legend on the right (makes you look back and forth)
- Title: "Q3 Revenue Breakdown"

✓ GOOD CHART:
- Horizontal bar chart, sorted by value
- One highlight color for the key insight, gray for everything else
- Labels directly on bars
- Title: "Electronics drives 40% of revenue — 2x any other category"
- Subtitle with context: "Q3 2024, $12.4M total revenue"
```

### Color Guidelines

| Purpose | Approach |
|---------|----------|
| Categorical (different groups) | Use distinct hues (max 5-7 colors) |
| Sequential (low to high) | Single color gradient (light → dark) |
| Diverging (negative vs positive) | Two-color scale (red ← neutral → green) |
| Highlighting | One color + gray for everything else |
| Accessibility | Always check for colorblind-safe palettes |

---

## Building Effective Dashboards

### Dashboard Design Principles

```
┌─────────────────────────────────────────────────────────────────┐
│  DASHBOARD LAYOUT (Z-Pattern Reading)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  KPI CARDS (Top)                                             │ │
│  │  Revenue: $2.4M (+12%)  |  Users: 45K (+5%)  |  NPS: 72    │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌──────────────────────────┐  ┌──────────────────────────────┐ │
│  │  MAIN CHART (Top-Left)   │  │  SECONDARY CHART (Top-Right) │ │
│  │                          │  │                              │ │
│  │  Revenue trend over time │  │  Revenue by channel          │ │
│  │  (the primary story)     │  │  (the breakdown)             │ │
│  └──────────────────────────┘  └──────────────────────────────┘ │
│                                                                   │
│  ┌──────────────────────────┐  ┌──────────────────────────────┐ │
│  │  DETAIL TABLE            │  │  ALERTING / ANOMALIES        │ │
│  │  (Bottom-Left)           │  │  (Bottom-Right)              │ │
│  │  Top products, segments  │  │  What needs attention?       │ │
│  └──────────────────────────┘  └──────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  FILTERS: Date Range | Region | Segment | Product Category  │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Dashboard Dos and Don'ts

| Do | Don't |
|----|-------|
| Answer ONE key question per dashboard | Cram everything onto one page |
| Put most important info top-left | Bury key metrics at the bottom |
| Show trends with context (targets, previous period) | Show numbers without reference points |
| Use consistent colors across the dashboard | Use different color schemes per chart |
| Add clear titles that state the insight | Use generic titles ("Revenue Chart") |
| Allow drill-down for details | Force everyone to see all details |
| Update in real-time or on a clear schedule | Leave stale data without timestamps |
| Test with actual users | Assume you know what they need |

### KPI Cards Best Practices

```
✓ GOOD KPI Card:
┌────────────────────────┐
│  Monthly Revenue       │
│  $2.4M                 │
│  ▲ +12% vs last month  │
│  Target: $2.5M (96%)   │
└────────────────────────┘
Shows: current value, trend direction, comparison to target

❌ BAD KPI Card:
┌────────────────────────┐
│  Revenue               │
│  $2,387,412.67         │
└────────────────────────┘
Missing: context, comparison, trend
```

---

## Presenting to Stakeholders

### The 10-Minute Presentation Framework

```
STRUCTURE (10 minutes total):

Minute 0-1:   HOOK
              "We found something that could save/earn us $X"
              
Minute 1-3:   CONTEXT
              "Here's the situation and what we investigated"
              
Minute 3-7:   EVIDENCE
              "Here's what the data shows" (2-3 key charts)
              
Minute 7-9:   RECOMMENDATION
              "Based on this, we should do X"
              "Expected impact: Y, Timeline: Z"
              
Minute 9-10:  NEXT STEPS
              "Here's what I need from you to move forward"
```

### Slide Design for Data Presentations

```
EACH SLIDE SHOULD HAVE:

┌─────────────────────────────────────────────────────────────┐
│                                                               │
│  HEADLINE: States the insight (not description)              │
│  ─────────────────────────────────────────                   │
│  ❌ "Revenue by Quarter"                                     │
│  ✓  "Q3 revenue grew 23% — fastest since 2021"             │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                                                        │   │
│  │          ONE visualization that proves the headline   │   │
│  │          (annotated with key callouts)                 │   │
│  │                                                        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                               │
│  KEY TAKEAWAY: One sentence summary at the bottom            │
│  "Recommendation: Increase ad spend in channel X by 20%"    │
│                                                               │
│                                                    Slide 3/8 │
└─────────────────────────────────────────────────────────────┘
```

### Handling Questions

| Question Type | How to Handle |
|---------------|--------------|
| "Is this statistically significant?" | Have p-values and CIs ready in appendix |
| "What about segment X?" | Pre-prepare segment breakdowns |
| "How does this compare to..." | Always have benchmarks ready |
| "What's the risk?" | Acknowledge limitations proactively |
| "What if you're wrong?" | Show sensitivity analysis |
| "Can you just show me the data?" | Have detailed tables in backup slides |

### Stakeholder Objections and Responses

```
OBJECTION: "But that's just a correlation, not causation"
RESPONSE: "You're right. That's why we ran an A/B test / controlled for X / 
used causal inference methods. Here's the evidence for causation..."

OBJECTION: "The sample size seems small"
RESPONSE: "We powered this test to detect a X% difference with 80% confidence.
Here's the power calculation. We'd need Y more weeks to detect smaller effects."

OBJECTION: "This doesn't match my experience"
RESPONSE: "I understand that intuition. Let me show you the segment breakdown — 
you may be thinking of [specific segment] where the pattern IS different."

OBJECTION: "What about [thing you didn't analyze]?"
RESPONSE: "Great question. That's worth investigating as a follow-up. 
For now, here's what we can confidently say with the current analysis..."
```

---

## Python Visualization Code

### Publication-Quality Charts with Matplotlib

```python
import matplotlib.pyplot as plt
import matplotlib.ticker as mtick
import numpy as np
import pandas as pd

# Set professional style globally
plt.style.use('seaborn-v0_8-whitegrid')
plt.rcParams.update({
    'font.size': 12,
    'font.family': 'sans-serif',
    'axes.titlesize': 14,
    'axes.titleweight': 'bold',
    'axes.labelsize': 12,
    'figure.figsize': (10, 6),
    'figure.dpi': 100,
})

# ══════════════════════════════════════════════════════════════
# Chart 1: Line Chart with Context (Trend + Target + Annotation)
# ══════════════════════════════════════════════════════════════

def plot_revenue_trend():
    """Revenue trend with target line and key callout."""
    
    months = pd.date_range('2024-01', periods=12, freq='M')
    revenue = [2.1, 2.3, 2.5, 2.4, 2.8, 3.0, 2.9, 3.2, 3.5, 3.4, 3.8, 4.1]
    target = [2.5] * 12  # Monthly target

    fig, ax = plt.subplots(figsize=(12, 6))
    
    # Main line
    ax.plot(months, revenue, color='#2563EB', linewidth=2.5, marker='o', 
            markersize=6, label='Actual Revenue')
    
    # Target line
    ax.axhline(y=2.5, color='#DC2626', linestyle='--', linewidth=1.5, 
               alpha=0.7, label='Target ($2.5M)')
    
    # Shade the area above target (success)
    ax.fill_between(months, revenue, 2.5, 
                    where=[r > 2.5 for r in revenue],
                    alpha=0.1, color='#16A34A')
    
    # Annotate the key insight
    ax.annotate('New pricing\nlaunched here',
                xy=(months[4], revenue[4]),
                xytext=(months[2], 3.3),
                fontsize=11,
                arrowprops=dict(arrowstyle='->', color='gray'),
                bbox=dict(boxstyle='round,pad=0.3', facecolor='lightyellow'))
    
    # Title that states the insight
    ax.set_title("Revenue exceeded target every month since May\n(New pricing strategy working)", 
                 pad=20)
    ax.set_ylabel("Revenue ($M)")
    ax.set_xlabel("")
    ax.legend(loc='upper left', framealpha=0.9)
    
    # Format y-axis as dollars
    ax.yaxis.set_major_formatter(mtick.FormatStrFormatter('$%.1fM'))
    
    # Remove top and right spines
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)
    
    plt.tight_layout()
    plt.savefig('revenue_trend.png', dpi=150, bbox_inches='tight')
    plt.show()

plot_revenue_trend()
```

```python
# ══════════════════════════════════════════════════════════════
# Chart 2: Bar Chart with Highlighting (Comparison)
# ══════════════════════════════════════════════════════════════

def plot_category_comparison():
    """Bar chart highlighting the key category."""
    
    categories = ['Electronics', 'Clothing', 'Home & Garden', 
                  'Food & Beverage', 'Sports', 'Books', 'Beauty']
    revenue = [4.2, 2.8, 2.1, 1.9, 1.5, 0.9, 0.8]
    growth = [23, 5, -3, 12, 8, -15, 45]  # YoY growth %
    
    fig, ax = plt.subplots(figsize=(10, 7))
    
    # Color: highlight top performer and fastest grower
    colors = ['#D1D5DB'] * len(categories)  # Default gray
    colors[0] = '#2563EB'  # Electronics - highest revenue (blue)
    colors[6] = '#16A34A'  # Beauty - fastest growth (green)
    
    # Horizontal bar chart (easier to read labels)
    bars = ax.barh(categories, revenue, color=colors, edgecolor='white', height=0.6)
    
    # Add value labels on bars
    for bar, val, g in zip(bars, revenue, growth):
        # Revenue label
        ax.text(bar.get_width() + 0.05, bar.get_y() + bar.get_height()/2,
                f'${val}M', va='center', fontsize=11, fontweight='bold')
        # Growth label
        color = '#16A34A' if g > 0 else '#DC2626'
        ax.text(bar.get_width() + 0.6, bar.get_y() + bar.get_height()/2,
                f'{"+" if g > 0 else ""}{g}% YoY', va='center', 
                fontsize=10, color=color)
    
    ax.set_title("Electronics dominates revenue, but Beauty is growing fastest (+45% YoY)",
                 pad=20)
    ax.set_xlabel("Revenue ($M)")
    ax.set_xlim(0, 5.5)
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)
    ax.invert_yaxis()  # Highest at top
    
    plt.tight_layout()
    plt.savefig('category_comparison.png', dpi=150, bbox_inches='tight')
    plt.show()

plot_category_comparison()
```

```python
# ══════════════════════════════════════════════════════════════
# Chart 3: Before/After Comparison (A/B Test Results)
# ══════════════════════════════════════════════════════════════

def plot_ab_test_results():
    """Visualize A/B test results with confidence intervals."""
    
    fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 5))
    
    # Left plot: Conversion rates with CI
    groups = ['Control (A)', 'Treatment (B)']
    rates = [0.082, 0.097]  # 8.2% vs 9.7%
    ci_lower = [0.076, 0.091]
    ci_upper = [0.088, 0.103]
    
    colors = ['#6B7280', '#2563EB']
    
    bars = ax1.bar(groups, rates, color=colors, width=0.5, edgecolor='white')
    
    # Error bars (confidence intervals)
    ax1.errorbar(groups, rates, 
                 yerr=[[r-l for r,l in zip(rates, ci_lower)],
                       [u-r for r,u in zip(rates, ci_upper)]],
                 fmt='none', color='black', capsize=8, linewidth=2)
    
    # Value labels
    for bar, rate in zip(bars, rates):
        ax1.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 0.005,
                f'{rate:.1%}', ha='center', fontsize=14, fontweight='bold')
    
    ax1.set_title("Treatment increased conversion by 18%\n(8.2% → 9.7%, p < 0.01)", 
                  fontweight='bold')
    ax1.set_ylabel("Conversion Rate")
    ax1.yaxis.set_major_formatter(mtick.PercentFormatter(1.0))
    ax1.set_ylim(0, 0.13)
    ax1.spines['top'].set_visible(False)
    ax1.spines['right'].set_visible(False)
    
    # Right plot: Lift distribution (Bayesian)
    from scipy import stats
    x = np.linspace(-0.05, 0.25, 1000)
    lift_dist = stats.norm.pdf(x, loc=0.18, scale=0.05)
    
    ax2.plot(x, lift_dist, color='#2563EB', linewidth=2)
    ax2.fill_between(x, lift_dist, where=(x >= 0.08) & (x <= 0.28),
                     alpha=0.3, color='#2563EB', label='95% CI')
    ax2.axvline(x=0.18, color='#2563EB', linestyle='-', linewidth=1.5)
    ax2.axvline(x=0, color='#DC2626', linestyle='--', linewidth=1.5, label='No effect')
    
    ax2.set_title("Relative lift: +18% [+8%, +28%]\n(Entire CI is positive → ship it)", 
                  fontweight='bold')
    ax2.set_xlabel("Relative Lift")
    ax2.xaxis.set_major_formatter(mtick.PercentFormatter(1.0))
    ax2.legend()
    ax2.spines['top'].set_visible(False)
    ax2.spines['right'].set_visible(False)
    
    plt.tight_layout()
    plt.savefig('ab_test_results.png', dpi=150, bbox_inches='tight')
    plt.show()

plot_ab_test_results()
```

### Interactive Dashboards with Plotly

```python
import plotly.express as px
import plotly.graph_objects as go
from plotly.subplots import make_subplots
import pandas as pd
import numpy as np

def create_executive_dashboard():
    """Create an interactive executive dashboard."""
    
    # Sample data
    np.random.seed(42)
    dates = pd.date_range('2024-01-01', periods=365, freq='D')
    
    df = pd.DataFrame({
        'date': dates,
        'revenue': np.cumsum(np.random.normal(50000, 15000, 365)),
        'users': np.cumsum(np.random.normal(500, 100, 365)),
        'conversion': np.random.normal(0.08, 0.01, 365),
    })
    df['revenue'] = df['revenue'].clip(lower=0)
    df['month'] = df['date'].dt.to_period('M').astype(str)
    
    # Create subplot dashboard
    fig = make_subplots(
        rows=2, cols=2,
        subplot_titles=(
            'Cumulative Revenue (Growing 15% MoM)',
            'Daily Conversion Rate',
            'Monthly Revenue Growth',
            'User Acquisition Trend'
        ),
        specs=[[{"type": "scatter"}, {"type": "scatter"}],
               [{"type": "bar"}, {"type": "scatter"}]]
    )
    
    # Revenue trend
    fig.add_trace(
        go.Scatter(x=df['date'], y=df['revenue'], mode='lines',
                   name='Revenue', line=dict(color='#2563EB', width=2)),
        row=1, col=1
    )
    
    # Conversion rate with moving average
    fig.add_trace(
        go.Scatter(x=df['date'], y=df['conversion'].rolling(7).mean(),
                   mode='lines', name='Conv. Rate (7d avg)',
                   line=dict(color='#16A34A', width=2)),
        row=1, col=2
    )
    
    # Monthly revenue bars
    monthly = df.groupby('month')['revenue'].last().diff().dropna()
    fig.add_trace(
        go.Bar(x=monthly.index, y=monthly.values, name='Monthly Growth',
               marker_color='#7C3AED'),
        row=2, col=1
    )
    
    # User growth
    fig.add_trace(
        go.Scatter(x=df['date'], y=df['users'], mode='lines',
                   name='Total Users', line=dict(color='#F59E0B', width=2)),
        row=2, col=2
    )
    
    fig.update_layout(
        height=700,
        title_text="Executive Dashboard — Key Metrics Overview",
        showlegend=False,
        template='plotly_white'
    )
    
    fig.write_html('executive_dashboard.html')
    fig.show()

create_executive_dashboard()
```

### Automated Report Generation

```python
import pandas as pd
import numpy as np
from datetime import datetime

def generate_weekly_report(df: pd.DataFrame) -> str:
    """
    Generate an automated weekly report in Markdown.
    This can be sent via email, Slack, or rendered in a dashboard.
    """
    
    report_date = datetime.now().strftime('%Y-%m-%d')
    
    # Calculate key metrics
    current_week = df[df['date'] >= df['date'].max() - pd.Timedelta(days=7)]
    previous_week = df[(df['date'] >= df['date'].max() - pd.Timedelta(days=14)) & 
                       (df['date'] < df['date'].max() - pd.Timedelta(days=7))]
    
    revenue_current = current_week['revenue'].sum()
    revenue_previous = previous_week['revenue'].sum()
    revenue_change = (revenue_current - revenue_previous) / revenue_previous * 100
    
    orders_current = len(current_week)
    orders_previous = len(previous_week)
    
    avg_order_current = current_week['revenue'].mean()
    avg_order_previous = previous_week['revenue'].mean()
    
    # Generate report
    report = f"""
# Weekly Data Report — {report_date}

## Executive Summary

{'🟢' if revenue_change > 0 else '🔴'} Revenue is **{'up' if revenue_change > 0 else 'down'} {abs(revenue_change):.1f}%** vs last week.

## Key Metrics

| Metric | This Week | Last Week | Change |
|--------|-----------|-----------|--------|
| Revenue | ${revenue_current:,.0f} | ${revenue_previous:,.0f} | {revenue_change:+.1f}% |
| Orders | {orders_current:,} | {orders_previous:,} | {(orders_current-orders_previous)/orders_previous*100:+.1f}% |
| Avg Order Value | ${avg_order_current:,.2f} | ${avg_order_previous:,.2f} | {(avg_order_current-avg_order_previous)/avg_order_previous*100:+.1f}% |

## Key Insights

1. **Top Finding:** {"Revenue growth driven by increased order volume" if revenue_change > 0 else "Revenue decline — investigate conversion drop"}
2. **Action Needed:** {"Continue current strategy" if revenue_change > 0 else "Review funnel for bottlenecks"}

## Recommendations

- {"Scale up marketing spend on performing channels" if revenue_change > 5 else "Investigate root cause of stagnation"}
- Monitor conversion rate closely this week

---
*Generated automatically. Data through {report_date}.*
"""
    
    return report

# Usage
np.random.seed(42)
dates = pd.date_range('2024-01-01', periods=90, freq='D')
sample_df = pd.DataFrame({
    'date': dates,
    'revenue': np.random.lognormal(6, 0.5, 90),
    'customer_id': np.random.randint(1, 1000, 90)
})

report = generate_weekly_report(sample_df)
print(report)

# Save to file
with open('weekly_report.md', 'w') as f:
    f.write(report)
```

---

## Writing Data Reports

### Report Structure Template

```markdown
# [Analysis Title — State the Key Finding]

## TL;DR (Executive Summary)
- One paragraph: What we found, why it matters, what to do
- Keep under 5 sentences

## Background & Motivation
- Why did we do this analysis?
- What business question are we answering?
- What data did we use? (Time period, sources)

## Methodology
- Brief description (non-technical)
- Link to detailed notebook for those who want depth

## Key Findings
### Finding 1: [Insight as a sentence]
[Supporting chart + 2-3 sentences of context]

### Finding 2: [Insight as a sentence]
[Supporting chart + 2-3 sentences of context]

### Finding 3: [Insight as a sentence]
[Supporting chart + 2-3 sentences of context]

## Recommendations
1. [Specific action] — Expected impact: $X / Y% improvement
2. [Specific action] — Expected impact: $X / Y% improvement
3. [Specific action] — Expected impact: $X / Y% improvement

## Limitations & Caveats
- What we DIDN'T account for
- Assumptions that could be wrong
- Data quality issues

## Appendix
- Detailed methodology
- Full statistical outputs
- Raw data tables
```

### The "So What?" Test

For every number or chart you present, ask yourself: **"So what?"**

```
"Revenue was $2.4M last month"
  → So what?
"That's 12% above target"
  → So what?
"It means our new pricing strategy is working and we should expand it"
  → NOW we have something actionable.

Apply this recursively until you reach an ACTION.
```

---

## Common Mistakes

### 1. Presenting Data Dumps Instead of Stories

```
❌ "Here are 47 charts from my analysis"
✓  "Here are 3 charts that answer the business question"
```

### 2. Leading with Methodology

```
❌ "First, I'll explain the Random Forest model I used..."
✓  "We can predict customer churn with 84% accuracy. Here's who's at risk."
   (Details in appendix for those interested)
```

### 3. No Clear Recommendation

```
❌ "The data shows some interesting patterns..."
✓  "Based on this analysis, I recommend we do X. Expected impact: $Y."
```

### 4. Misleading Visualizations

| Trick | Why It's Wrong | Fix |
|-------|---------------|-----|
| Truncated y-axis | Makes small differences look huge | Start at 0 (or clearly mark axis) |
| Cherry-picked time range | Hides unfavorable trends | Show full context |
| Dual y-axes | Implies correlation that may not exist | Use two separate charts |
| Area chart for non-cumulative data | Implies volume that doesn't exist | Use line chart |
| Pie chart with 15 slices | Unreadable | Use horizontal bar |

### 5. Ignoring Audience Context

```
❌ To CEO: "The p-value was 0.003 with a Cohen's d of 0.45"
✓  To CEO: "We're 99.7% confident this change will add $3M annually"

❌ To Engineers: "The vibes are good, we should ship it"
✓  To Engineers: "Here's the API spec, expected QPS increase is 15%, 
   latency impact is <2ms based on load testing"
```

### 6. Not Acknowledging Uncertainty

```
❌ "Revenue WILL grow 15% next quarter"
✓  "Revenue is expected to grow 12-18% next quarter (95% CI), 
   assuming no major market changes"
```

---

## Interview Questions

**Q1: How would you present a complex analysis to a non-technical executive in 5 minutes?**

> **Answer:** Use the Pyramid Principle:
> 1. Lead with the headline: "We're losing $X because of Y"
> 2. Show one chart that proves it (annotated, insight as title)
> 3. State the recommendation: "Do X, expect Y impact in Z timeframe"
> 4. Offer to go deeper if they want details
> 
> Key: Never start with methodology. Start with the business impact.

**Q2: You built a model with 92% accuracy. How do you explain this to a product manager?**

> **Answer:** Don't say "92% accuracy." Instead:
> - "Out of every 100 customers the model flags as likely to churn, 92 actually will churn"
> - "If we intervene on the flagged customers, we can save approximately 500 accounts per month worth $X"
> - Translate technical metrics into business outcomes they care about
> - Show what happens with AND without the model (the counterfactual)

**Q3: What makes a good dashboard vs a bad one?**

> **Answer:** A good dashboard:
> - Answers ONE specific question (not "everything about the business")
> - Is scannable in 5 seconds (key metrics immediately visible)
> - Shows trends and context (not just current numbers)
> - Has clear action triggers ("if this number goes below X, do Y")
> - Updates on a clear schedule
> 
> A bad dashboard: Has 30+ charts, requires scrolling, no clear hierarchy, stale data, no context for what's "good" vs "bad."

**Q4: How do you handle presenting results that contradict what leadership believes?**

> **Answer:** 
> 1. Lead with empathy: "I understand the current assumption is X. I believed the same."
> 2. Show the data clearly and let it speak
> 3. Acknowledge where the intuition IS correct (find common ground)
> 4. Propose a low-risk way to validate: "We could run a small test to verify"
> 5. Never say "you're wrong" — say "the data suggests something different"
> 6. Frame it as an opportunity, not a criticism

**Q5: Describe a situation where you'd choose NOT to share a finding.**

> **Answer:** 
> - When the finding is based on insufficient data (N too small, not statistically significant)
> - When sharing could cause harm and there's no actionable recommendation yet
> - When the data quality is questionable and you haven't validated it
> - However, you should always document it internally and flag for follow-up
> - The decision to withhold should be about readiness, not politics

---

## Quick Reference

### Communication Framework Cheat Sheet

| Framework | Structure | Best For |
|-----------|-----------|----------|
| **SCA** | Situation → Complication → Action | Any analysis presentation |
| **Pyramid** | Conclusion → Supporting → Details | Busy executives |
| **STAR** | Situation → Task → Action → Result | Interview answers |
| **Minto** | Answer first → Group arguments → Detail | Written reports |

### Chart Selection Quick Guide

| Data Relationship | Best Chart | Never Use |
|-------------------|-----------|-----------|
| Comparison (few items) | Horizontal bar | Pie chart with >5 slices |
| Trend over time | Line chart | Area chart (misleading) |
| Part-to-whole (≤5 parts) | Pie or stacked bar | Pie with many slices |
| Distribution | Histogram, box plot | Bar chart for continuous data |
| Correlation | Scatter plot | Dual-axis charts |
| Ranking | Sorted bar chart | Unsorted bar chart |

### Presentation Timing Guide

| Audience | Time Available | Slides | Detail Level |
|----------|---------------|--------|--------------|
| C-Suite | 5 min | 3-5 | Headlines only |
| VP/Director | 15 min | 8-10 | Key findings + recs |
| Team meeting | 30 min | 12-15 | Findings + methodology |
| Technical review | 60 min | 20+ | Full deep dive |

### Data Storytelling Checklist

| Step | Question | Done? |
|------|----------|-------|
| 1 | Who is my audience? What do they care about? | ☐ |
| 2 | What's the ONE key message? | ☐ |
| 3 | What action should they take after seeing this? | ☐ |
| 4 | Does every chart have an insight title (not description)? | ☐ |
| 5 | Have I translated numbers into business impact ($, users, time)? | ☐ |
| 6 | Is the most important info first (not buried)? | ☐ |
| 7 | Have I removed everything that doesn't support the message? | ☐ |
| 8 | Did I acknowledge limitations and uncertainty? | ☐ |
| 9 | Is there a clear next step / call-to-action? | ☐ |
| 10 | Would a smart non-technical person understand this in 60 seconds? | ☐ |

### Tools Reference

| Purpose | Tools |
|---------|-------|
| Static charts | Matplotlib, Seaborn, Plotnine (ggplot for Python) |
| Interactive charts | Plotly, Altair, Bokeh |
| Dashboards | Tableau, Looker, Metabase, Streamlit, Dash |
| Presentations | Google Slides, Reveal.js, Marp (Markdown → slides) |
| Reports | Jupyter + nbconvert, Quarto, R Markdown |
| Automated reports | Python + Jinja2 templates, scheduled notebooks |

---

*This completes the Data Science series. Return to [00-INDEX](00-INDEX.md) for the full chapter list.*
