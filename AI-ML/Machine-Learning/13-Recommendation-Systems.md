# Chapter 13: Recommendation Systems

## Table of Contents
- [What it is](#what-it-is)
- [Why it matters](#why-it-matters)
- [Types of Recommendation Systems](#types-of-recommendation-systems)
- [Content-Based Filtering](#content-based-filtering)
- [Collaborative Filtering](#collaborative-filtering)
- [Matrix Factorization](#matrix-factorization)
- [Hybrid Systems](#hybrid-systems)
- [Deep Learning for Recommendations](#deep-learning-for-recommendations)
- [Evaluation Metrics](#evaluation-metrics)
- [Production Considerations](#production-considerations)
- [Code Examples](#code-examples)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What it is

A **recommendation system** (RecSys) is an algorithm that suggests relevant items to users based on their preferences, behavior, or characteristics of the items themselves.

**Simple Analogy:** Imagine you walk into a bookstore. A great bookstore owner who knows you might say:
- "You liked Harry Potter, so try Percy Jackson" → **Content-Based** (similar items)
- "People who bought Harry Potter also bought Lord of the Rings" → **Collaborative Filtering** (similar users)
- "Based on your reading history AND what similar readers enjoy..." → **Hybrid**

The system learns patterns from data to predict what a user might like next.

---

## Why it matters

### Real-World Impact
| Company | Use Case | Impact |
|---------|----------|--------|
| Netflix | Movie recommendations | 80% of content watched comes from recommendations |
| Amazon | Product suggestions | 35% of revenue from recommendations |
| Spotify | Discover Weekly | 30% of all listening from algorithmic playlists |
| YouTube | Video recommendations | 70% of watch time from recommendations |
| LinkedIn | Job/connection suggestions | Core engagement driver |

### When You'd Use It
- **E-commerce:** "Customers who bought X also bought Y"
- **Streaming:** Personalized content feeds
- **Social media:** Friend/follow suggestions
- **News:** Personalized article feeds
- **Recruiting:** Job-candidate matching
- **Dating apps:** Partner matching
- **Ad targeting:** Showing relevant ads

### Business Value
- Increases user engagement and time-on-platform
- Drives revenue through cross-selling and upselling
- Reduces choice overload (paradox of choice)
- Improves customer retention and satisfaction

---

## Types of Recommendation Systems

```
┌────────────────────────────────────────────────────────────┐
│              RECOMMENDATION SYSTEMS                          │
├──────────────┬──────────────┬──────────────┬───────────────┤
│  Content-    │ Collaborative│   Matrix     │    Hybrid     │
│  Based       │  Filtering   │ Factorization│   Systems     │
├──────────────┼──────────────┼──────────────┼───────────────┤
│ Uses item    │ Uses user    │ Decomposes   │ Combines      │
│ features     │ behavior     │ user-item    │ multiple      │
│ to find      │ patterns to  │ matrix into  │ approaches    │
│ similar      │ find similar │ latent       │ for better    │
│ items        │ users/items  │ factors      │ accuracy      │
└──────────────┴──────────────┴──────────────┴───────────────┘
```

### Comparison Table

| Approach | Data Needed | Cold Start? | Scalability | Diversity |
|----------|------------|-------------|-------------|-----------|
| Content-Based | Item features | Partial (new users) | Good | Low (filter bubble) |
| User-Based CF | User ratings | Yes (both) | Poor for large users | High |
| Item-Based CF | User ratings | Yes (both) | Better than user-based | Medium |
| Matrix Factorization | User ratings | Yes (both) | Good | Medium |
| Hybrid | Both | Mitigated | Varies | High |

---

## Content-Based Filtering

### What it is
Recommends items similar to what a user has liked before, based on **item features/attributes**.

### How it Works

```
User Profile:           Item Features:
┌─────────────┐        ┌──────────────────────────┐
│ Likes:      │        │ Movie A: Action, Sci-Fi  │
│ - Action    │        │ Movie B: Romance, Drama  │
│ - Sci-Fi    │───────>│ Movie C: Action, Thriller│ ← Recommend!
│ - Thriller  │        │ Movie D: Comedy, Family  │
└─────────────┘        └──────────────────────────┘
```

**Steps:**
1. Build a profile of what the user likes (from past interactions)
2. Represent items as feature vectors
3. Compute similarity between user profile and candidate items
4. Rank and recommend top-N most similar items

### Mathematical Foundation

**TF-IDF for text-based features:**

$$TF(t, d) = \frac{\text{count of term } t \text{ in document } d}{\text{total terms in } d}$$

$$IDF(t) = \log\left(\frac{\text{total documents}}{\text{documents containing } t}\right)$$

$$TF\text{-}IDF(t, d) = TF(t, d) \times IDF(t)$$

**Cosine Similarity:**

$$\text{similarity}(A, B) = \frac{A \cdot B}{\|A\| \times \|B\|} = \frac{\sum_{i=1}^{n} A_i B_i}{\sqrt{\sum_{i=1}^{n} A_i^2} \times \sqrt{\sum_{i=1}^{n} B_i^2}}$$

### Advantages
- No cold-start for new items (if features are known)
- Transparent/explainable: "Recommended because you liked X"
- Works with single user's data (no need for other users)
- No popularity bias

### Disadvantages
- **Filter bubble:** Only recommends similar items, no serendipity
- Requires good feature engineering
- Cannot capture complex user preferences
- Cold-start for new users (no history)

---

## Collaborative Filtering

### What it is
Makes predictions based on the collective behavior of many users. The core idea: **"People who agreed in the past will agree in the future."**

### Two Types

#### 1. User-Based Collaborative Filtering

**Intuition:** Find users similar to you → recommend what they liked.

```
         Movie1  Movie2  Movie3  Movie4  Movie5
User A:    5       3       4       ?       2
User B:    5       2       4       5       1      ← Similar to A
User C:    1       5       2       4       5      ← Different from A
User D:    4       3       5       4       2      ← Somewhat similar

For User A's Movie4 prediction:
→ User B (similar) rated Movie4 = 5
→ User D (somewhat similar) rated Movie4 = 4
→ Predicted rating for A ≈ 4.5
```

**Formula (Weighted Average):**

$$\hat{r}_{u,i} = \bar{r}_u + \frac{\sum_{v \in N(u)} \text{sim}(u, v) \cdot (r_{v,i} - \bar{r}_v)}{\sum_{v \in N(u)} |\text{sim}(u, v)|}$$

Where:
- $\hat{r}_{u,i}$ = predicted rating of user $u$ for item $i$
- $\bar{r}_u$ = average rating of user $u$
- $N(u)$ = set of neighbors (similar users)
- $\text{sim}(u, v)$ = similarity between users $u$ and $v$

#### 2. Item-Based Collaborative Filtering

**Intuition:** Find items similar to what you've liked → recommend those items.

```
         User1  User2  User3  User4  User5
Movie A:   5      4      5      3      4     ← Target liked this
Movie B:   4      5      4      2      5     ← Similar pattern to A
Movie C:   1      2      1      5      2     ← Different pattern
Movie D:   5      3      5      4      4     ← Similar pattern to A

→ Recommend Movie B and D (similar rating patterns to A)
```

**Why Item-Based > User-Based in practice:**
- Items don't change as fast as user preferences
- Item-item similarity is more stable over time
- More scalable (fewer items than users typically)
- Amazon's "Customers who bought this also bought..." uses item-based CF

### Similarity Metrics

#### Pearson Correlation
$$\text{sim}(u, v) = \frac{\sum_{i \in I_{uv}} (r_{u,i} - \bar{r}_u)(r_{v,i} - \bar{r}_v)}{\sqrt{\sum_{i \in I_{uv}} (r_{u,i} - \bar{r}_u)^2} \cdot \sqrt{\sum_{i \in I_{uv}} (r_{v,i} - \bar{r}_v)^2}}$$

#### Cosine Similarity
$$\text{sim}(u, v) = \frac{\sum_{i \in I_{uv}} r_{u,i} \cdot r_{v,i}}{\sqrt{\sum_{i \in I_{uv}} r_{u,i}^2} \cdot \sqrt{\sum_{i \in I_{uv}} r_{v,i}^2}}$$

#### Jaccard Similarity (for implicit feedback)
$$\text{sim}(A, B) = \frac{|A \cap B|}{|A \cup B|}$$

### When to Use Which Similarity

| Metric | Best For | Handles Rating Scale Diff? |
|--------|----------|---------------------------|
| Pearson | Explicit ratings | Yes (mean-centered) |
| Cosine | Sparse data | No |
| Adjusted Cosine | Item-based CF | Yes |
| Jaccard | Binary/implicit data | N/A |

---

## Matrix Factorization

### What it is
Decomposes the large, sparse user-item interaction matrix into two smaller dense matrices that, when multiplied together, approximate the original matrix — revealing **latent factors** (hidden features).

### Intuition

Think of it this way: When you rate movies, you don't think "I rate action movies 4.2" — you have hidden preferences like "I like complex plots," "I enjoy visual effects," "I prefer strong female leads." These are **latent factors**.

```
Original Matrix (sparse):          ≈    User Matrix    ×    Item Matrix
                                         (U)                  (V^T)
     Item1 Item2 Item3 Item4            f1  f2  f3         f1  f2  f3
U1 [  5     ?     3     ?  ]       U1 [0.8 0.2 0.9]   I1 [0.9 0.1 0.8]
U2 [  ?     4     ?     2  ]   ≈   U2 [0.3 0.7 0.4] × I2 [0.2 0.8 0.3]
U3 [  4     ?     5     ?  ]       U3 [0.7 0.1 0.9]   I3 [0.6 0.3 0.9]
U4 [  ?     3     ?     4  ]       U4 [0.4 0.6 0.5]   I4 [0.3 0.7 0.5]

Latent factors might represent:
f1 = "likes action/adventure"
f2 = "likes romance/drama"  
f3 = "likes complex narratives"
```

### Mathematical Formulation

Given rating matrix $R \in \mathbb{R}^{m \times n}$ (m users, n items):

$$R \approx U \times V^T$$

Where:
- $U \in \mathbb{R}^{m \times k}$ (user latent factor matrix)
- $V \in \mathbb{R}^{n \times k}$ (item latent factor matrix)
- $k$ = number of latent factors (hyperparameter, typically 20-200)

**Predicted rating:**
$$\hat{r}_{u,i} = \mu + b_u + b_i + \mathbf{p}_u^T \cdot \mathbf{q}_i$$

Where:
- $\mu$ = global average rating
- $b_u$ = user bias (some users rate everything high)
- $b_i$ = item bias (some items are universally liked)
- $\mathbf{p}_u$ = user latent vector
- $\mathbf{q}_i$ = item latent vector

### Optimization: Minimizing Loss

$$\min_{p, q, b} \sum_{(u,i) \in \text{known}} \left(r_{u,i} - \hat{r}_{u,i}\right)^2 + \lambda\left(\|\mathbf{p}_u\|^2 + \|\mathbf{q}_i\|^2 + b_u^2 + b_i^2\right)$$

The regularization term $\lambda$ prevents overfitting.

### Algorithms

#### 1. SVD (Singular Value Decomposition)
- Classic linear algebra decomposition: $R = U \Sigma V^T$
- Problem: Doesn't handle missing values well
- Used as baseline, not practical for sparse matrices

#### 2. Funk SVD (Simon Funk's approach)
- Only trains on observed ratings
- Uses stochastic gradient descent (SGD)
- Won Netflix Prize (in combination with other methods)

**SGD Update Rules:**
$$p_u \leftarrow p_u + \alpha (e_{ui} \cdot q_i - \lambda \cdot p_u)$$
$$q_i \leftarrow q_i + \alpha (e_{ui} \cdot p_u - \lambda \cdot q_i)$$

Where $e_{ui} = r_{ui} - \hat{r}_{ui}$ (prediction error)

#### 3. ALS (Alternating Least Squares)
- Fixes one matrix, optimizes the other, alternates
- Easily parallelizable (used in Spark MLlib)
- Better for implicit feedback data

#### 4. NMF (Non-negative Matrix Factorization)
- Constrains factors to be non-negative
- More interpretable latent factors
- Good when ratings/interactions are non-negative

### Implicit vs Explicit Feedback

| Aspect | Explicit | Implicit |
|--------|----------|----------|
| Data | Ratings, reviews | Clicks, views, purchases |
| Availability | Sparse | Dense |
| Confidence | High | Variable |
| Negative signal | Low ratings | Absence (ambiguous) |
| Example | "Rated 5 stars" | "Watched 80% of movie" |

**Implicit Feedback Model (Hu, Koren, Volinsky 2008):**

$$\min_{x, y} \sum_{u,i} c_{ui}(p_{ui} - x_u^T y_i)^2 + \lambda(\|x_u\|^2 + \|y_i\|^2)$$

Where:
- $p_{ui} = 1$ if user interacted with item, else 0
- $c_{ui} = 1 + \alpha \cdot r_{ui}$ (confidence grows with more interactions)

---

## Hybrid Systems

### What it is
Combines multiple recommendation approaches to overcome individual weaknesses.

### Hybridization Strategies

```
┌─────────────────────────────────────────────────────────┐
│                   HYBRID STRATEGIES                       │
├──────────────┬──────────────────────────────────────────┤
│ Weighted     │ Combine scores: 0.6×CF + 0.4×Content    │
├──────────────┼──────────────────────────────────────────┤
│ Switching    │ Use Content when cold-start, CF otherwise│
├──────────────┼──────────────────────────────────────────┤
│ Feature      │ CF output becomes feature for another    │
│ Augmentation │ model                                    │
├──────────────┼──────────────────────────────────────────┤
│ Cascade      │ Rough filter (Content) → Fine rank (CF) │
├──────────────┼──────────────────────────────────────────┤
│ Stacking     │ Multiple models → Meta-learner combines  │
└──────────────┴──────────────────────────────────────────┘
```

### Netflix's Approach (Simplified)
1. **Candidate Generation:** Content-based + CF generate candidates
2. **Ranking:** Deep learning model ranks candidates
3. **Re-ranking:** Business rules (diversity, freshness) applied
4. **Serving:** Final personalized list displayed

---

## Deep Learning for Recommendations

### Neural Collaborative Filtering (NCF)

Replaces the dot product in matrix factorization with a neural network:

```
Input:          User ID (one-hot)    Item ID (one-hot)
                     │                      │
Embedding:      User Embedding         Item Embedding
                     │                      │
                     └──────────┬───────────┘
                                │
                         Concatenate
                                │
                         ┌──────┴──────┐
                         │  Dense(256) │
                         │  ReLU       │
                         ├─────────────┤
                         │  Dense(128) │
                         │  ReLU       │
                         ├─────────────┤
                         │  Dense(64)  │
                         │  ReLU       │
                         ├─────────────┤
                         │  Dense(1)   │
                         │  Sigmoid    │
                         └─────────────┘
                                │
Output:              Predicted Rating/Probability
```

### Two-Tower Architecture (Industry Standard)

```
    User Tower                    Item Tower
┌──────────────┐             ┌──────────────┐
│ User Features│             │ Item Features│
│ (age, gender,│             │ (title, genre│
│  history...) │             │  year, tags) │
├──────────────┤             ├──────────────┤
│   Dense      │             │   Dense      │
│   Layers     │             │   Layers     │
├──────────────┤             ├──────────────┤
│ User         │             │ Item         │
│ Embedding    │             │ Embedding    │
│ (128-dim)    │             │ (128-dim)    │
└──────┬───────┘             └──────┬───────┘
       │                            │
       └────── Dot Product ─────────┘
                    │
              Similarity Score
```

**Why Two-Tower?**
- Item embeddings can be pre-computed and cached
- Enables real-time retrieval with approximate nearest neighbors (ANN)
- Scales to millions of items

### Sequence-Based Models

For "what will user watch next?" problems:

- **GRU4Rec:** Uses GRU to model session sequences
- **SASRec:** Self-attention over user's interaction history
- **BERT4Rec:** Bidirectional transformer for recommendations

---

## Evaluation Metrics

### For Rating Prediction

$$RMSE = \sqrt{\frac{1}{n}\sum_{i=1}^{n}(r_i - \hat{r}_i)^2}$$

$$MAE = \frac{1}{n}\sum_{i=1}^{n}|r_i - \hat{r}_i|$$

### For Ranking (Top-N Recommendations)

#### Precision@K
$$\text{Precision@K} = \frac{|\text{relevant items in top-K}|}{K}$$

#### Recall@K
$$\text{Recall@K} = \frac{|\text{relevant items in top-K}|}{|\text{total relevant items}|}$$

#### MAP (Mean Average Precision)
$$MAP = \frac{1}{|U|} \sum_{u=1}^{|U|} AP(u)$$

$$AP(u) = \frac{1}{m_u} \sum_{k=1}^{N} P(k) \cdot \text{rel}(k)$$

#### NDCG (Normalized Discounted Cumulative Gain)
$$DCG@K = \sum_{i=1}^{K} \frac{2^{rel_i} - 1}{\log_2(i + 1)}$$

$$NDCG@K = \frac{DCG@K}{IDCG@K}$$

> **Important:** NDCG accounts for the position of relevant items. A relevant item at position 1 is worth more than at position 10.

### Beyond Accuracy Metrics

| Metric | What it Measures | Why it Matters |
|--------|-----------------|----------------|
| Coverage | % of items ever recommended | Avoids popularity bias |
| Diversity | How different recommended items are | Avoids monotony |
| Novelty | How "surprising" recommendations are | Discovery |
| Serendipity | Relevant AND unexpected items | Delight factor |
| Fairness | Equal treatment across groups | Ethics/legal |

---

## Production Considerations

### The Cold Start Problem

```
┌─────────────────────────────────────────────────────┐
│               COLD START SOLUTIONS                    │
├─────────────────────────────────────────────────────┤
│ New User:                                            │
│   • Ask preferences during onboarding               │
│   • Use demographic info for initial recs            │
│   • Show popular/trending items                      │
│   • Content-based until enough interactions          │
│                                                      │
│ New Item:                                            │
│   • Use item features/metadata for content-based     │
│   • Promote to diverse user segments for exploration │
│   • Use item attributes to place in latent space     │
│                                                      │
│ New System:                                          │
│   • Start with popularity-based                      │
│   • Use external data sources                        │
│   • Implement exploration strategies (MAB)           │
└─────────────────────────────────────────────────────┘
```

### Scalability Architecture

```
┌──────────────────────────────────────────────────────────┐
│                PRODUCTION PIPELINE                         │
│                                                           │
│  ┌─────────┐    ┌──────────────┐    ┌───────────────┐   │
│  │ Offline  │    │  Candidate   │    │   Ranking     │   │
│  │ Training │───>│  Generation  │───>│   Model       │   │
│  │ (daily)  │    │  (~1000)     │    │   (top-50)    │   │
│  └─────────┘    └──────────────┘    └───────┬───────┘   │
│                                              │            │
│                                     ┌────────┴────────┐  │
│                                     │  Re-ranking &   │  │
│                                     │  Business Rules │  │
│                                     │  (top-20)       │  │
│                                     └────────┬────────┘  │
│                                              │            │
│                                     ┌────────┴────────┐  │
│                                     │   User Sees     │  │
│                                     │   Final List    │  │
│                                     └─────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Key Production Challenges

1. **Scalability:** Millions of users × millions of items
   - Solution: ANN (Approximate Nearest Neighbors) — FAISS, Annoy, ScaNN
   
2. **Real-time updates:** User just watched something
   - Solution: Online learning, feature stores, streaming updates

3. **Exploration vs Exploitation:**
   - Exploitation: Recommend what we KNOW user likes
   - Exploration: Try new items to learn preferences
   - Solution: Multi-armed bandits (ε-greedy, Thompson Sampling, UCB)

4. **Feedback loops:** Popular items get more exposure → more clicks → more popular
   - Solution: Position bias correction, randomized experiments

5. **Privacy:** GDPR, data minimization
   - Solution: Federated learning, differential privacy

---

## Code Examples

### Example 1: Content-Based Filtering (Movies)

```python
import pandas as pd
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

# Create sample movie dataset
movies = pd.DataFrame({
    'title': ['The Dark Knight', 'Inception', 'Interstellar', 
              'The Notebook', 'Titanic', 'La La Land',
              'Mad Max: Fury Road', 'John Wick', 'The Matrix'],
    'genres': ['Action Crime Thriller', 'Action Sci-Fi Thriller',
               'Adventure Drama Sci-Fi', 'Drama Romance',
               'Drama Romance', 'Comedy Drama Romance',
               'Action Adventure Sci-Fi', 'Action Thriller',
               'Action Sci-Fi'],
    'description': [
        'Batman fights crime in Gotham with the Joker as villain',
        'Thief enters dreams to plant ideas in subconscious mind',
        'Astronauts travel through wormhole to find new home for humanity',
        'Young couple falls in love during summer romance',
        'Love story aboard the ill-fated ship across ocean',
        'Jazz musician and actress fall in love in Los Angeles',
        'Post-apocalyptic warrior rebels against tyrannical ruler',
        'Retired hitman seeks vengeance against those who wronged him',
        'Hacker discovers reality is a computer simulation'
    ]
})

# Combine text features for richer representation
movies['combined_features'] = movies['genres'] + ' ' + movies['description']

# Create TF-IDF matrix from combined features
tfidf = TfidfVectorizer(stop_words='english')  # Remove common words
tfidf_matrix = tfidf.fit_transform(movies['combined_features'])

print(f"TF-IDF Matrix Shape: {tfidf_matrix.shape}")
# Output: TF-IDF Matrix Shape: (9, 56) → 9 movies, 56 unique terms

# Compute cosine similarity between all pairs of movies
cosine_sim = cosine_similarity(tfidf_matrix, tfidf_matrix)

# Function to get recommendations
def get_content_recommendations(title, cosine_sim=cosine_sim, n=5):
    """Get top-N similar movies based on content features."""
    # Get index of the movie
    idx = movies[movies['title'] == title].index[0]
    
    # Get similarity scores for all movies with this movie
    sim_scores = list(enumerate(cosine_sim[idx]))
    
    # Sort by similarity (descending), exclude self
    sim_scores = sorted(sim_scores, key=lambda x: x[1], reverse=True)[1:n+1]
    
    # Get movie indices and scores
    movie_indices = [i[0] for i in sim_scores]
    scores = [i[1] for i in sim_scores]
    
    # Return recommendations with scores
    result = movies.iloc[movie_indices][['title', 'genres']].copy()
    result['similarity_score'] = scores
    return result

# Test: Get recommendations for "Inception"
print("\nMovies similar to 'Inception':")
print(get_content_recommendations('Inception'))
# Output:
# Movies similar to 'Inception':
#                  title               genres  similarity_score
# The Matrix       Action Sci-Fi              0.45
# The Dark Knight  Action Crime Thriller      0.38
# Interstellar     Adventure Drama Sci-Fi     0.35
# Mad Max          Action Adventure Sci-Fi    0.28
# John Wick        Action Thriller            0.22
```

### Example 2: User-Based Collaborative Filtering from Scratch

```python
import numpy as np
import pandas as pd
from scipy.spatial.distance import cosine

# Create user-item rating matrix (0 = not rated)
ratings_data = {
    'User': ['Alice', 'Alice', 'Alice', 'Alice',
             'Bob', 'Bob', 'Bob', 'Bob',
             'Carol', 'Carol', 'Carol',
             'Dave', 'Dave', 'Dave', 'Dave',
             'Eve', 'Eve', 'Eve', 'Eve'],
    'Movie': ['Matrix', 'Inception', 'Interstellar', 'Dark Knight',
              'Matrix', 'Inception', 'John Wick', 'Dark Knight',
              'Inception', 'Interstellar', 'Notebook',
              'Matrix', 'John Wick', 'Dark Knight', 'Notebook',
              'Matrix', 'Inception', 'Interstellar', 'John Wick'],
    'Rating': [5, 4, 5, 4,
               5, 5, 4, 4,
               3, 4, 5,
               4, 5, 5, 2,
               4, 4, 5, 3]
}

df = pd.DataFrame(ratings_data)

# Create user-item matrix
user_item_matrix = df.pivot_table(index='User', columns='Movie', values='Rating')
print("User-Item Matrix:")
print(user_item_matrix)
# Output:
# Movie    Dark Knight  Inception  Interstellar  John Wick  Matrix  Notebook
# User                                                                      
# Alice          4.0        4.0           5.0        NaN     5.0       NaN
# Bob            4.0        5.0           NaN        4.0     5.0       NaN
# Carol          NaN        3.0           4.0        NaN     NaN       5.0
# Dave           5.0        NaN           NaN        5.0     4.0       2.0
# Eve            NaN        4.0           5.0        3.0     4.0       NaN

def pearson_similarity(user1_ratings, user2_ratings):
    """Calculate Pearson correlation between two users on common items."""
    # Find items both users have rated
    common_mask = user1_ratings.notna() & user2_ratings.notna()
    
    if common_mask.sum() < 2:  # Need at least 2 common items
        return 0
    
    u1 = user1_ratings[common_mask]
    u2 = user2_ratings[common_mask]
    
    # Pearson correlation
    u1_centered = u1 - u1.mean()
    u2_centered = u2 - u2.mean()
    
    numerator = (u1_centered * u2_centered).sum()
    denominator = np.sqrt((u1_centered**2).sum()) * np.sqrt((u2_centered**2).sum())
    
    if denominator == 0:
        return 0
    
    return numerator / denominator

def predict_rating(target_user, target_item, user_item_matrix, k=3):
    """Predict rating using k most similar users."""
    if pd.notna(user_item_matrix.loc[target_user, target_item]):
        return user_item_matrix.loc[target_user, target_item]  # Already rated
    
    # Calculate similarity with all other users
    similarities = {}
    for other_user in user_item_matrix.index:
        if other_user == target_user:
            continue
        # Only consider users who rated the target item
        if pd.isna(user_item_matrix.loc[other_user, target_item]):
            continue
        
        sim = pearson_similarity(
            user_item_matrix.loc[target_user],
            user_item_matrix.loc[other_user]
        )
        if sim > 0:  # Only consider positively correlated users
            similarities[other_user] = sim
    
    if not similarities:
        return user_item_matrix.loc[target_user].mean()  # Fallback: user's average
    
    # Select top-k similar users
    top_k = sorted(similarities.items(), key=lambda x: x[1], reverse=True)[:k]
    
    # Weighted average prediction
    target_user_mean = user_item_matrix.loc[target_user].mean()
    numerator = 0
    denominator = 0
    
    for other_user, sim in top_k:
        other_mean = user_item_matrix.loc[other_user].mean()
        other_rating = user_item_matrix.loc[other_user, target_item]
        
        # Deviation from mean (accounts for different rating scales)
        numerator += sim * (other_rating - other_mean)
        denominator += abs(sim)
    
    if denominator == 0:
        return target_user_mean
    
    prediction = target_user_mean + (numerator / denominator)
    return np.clip(prediction, 1, 5)  # Keep within valid range

# Predict Alice's rating for "John Wick"
predicted = predict_rating('Alice', 'John Wick', user_item_matrix)
print(f"\nPredicted rating for Alice on 'John Wick': {predicted:.2f}")

# Get recommendations for Alice (predict all unrated items)
def get_cf_recommendations(user, user_item_matrix, n=3):
    """Get top-N recommendations for a user."""
    predictions = {}
    for item in user_item_matrix.columns:
        if pd.isna(user_item_matrix.loc[user, item]):
            pred = predict_rating(user, item, user_item_matrix)
            predictions[item] = pred
    
    # Sort by predicted rating
    sorted_preds = sorted(predictions.items(), key=lambda x: x[1], reverse=True)
    return sorted_preds[:n]

print(f"\nTop recommendations for Alice:")
for movie, score in get_cf_recommendations('Alice', user_item_matrix):
    print(f"  {movie}: {score:.2f}")
```

### Example 3: Matrix Factorization with Surprise Library

```python
# pip install scikit-surprise
from surprise import Dataset, Reader, SVD, SVDpp, NMF
from surprise.model_selection import cross_validate, GridSearchCV
from surprise import accuracy
import pandas as pd

# Load built-in MovieLens dataset (100K ratings)
data = Dataset.load_builtin('ml-100k')

# --- Basic SVD ---
algo = SVD(
    n_factors=100,       # Number of latent factors
    n_epochs=20,         # Number of SGD iterations
    lr_all=0.005,        # Learning rate
    reg_all=0.02,        # Regularization term
    random_state=42
)

# 5-fold cross-validation
results = cross_validate(algo, data, measures=['RMSE', 'MAE'], cv=5, verbose=True)
print(f"\nAverage RMSE: {results['test_rmse'].mean():.4f}")
print(f"Average MAE: {results['test_mae'].mean():.4f}")

# --- Hyperparameter Tuning ---
param_grid = {
    'n_factors': [50, 100, 150],
    'n_epochs': [20, 30],
    'lr_all': [0.002, 0.005],
    'reg_all': [0.02, 0.1]
}

gs = GridSearchCV(SVD, param_grid, measures=['rmse', 'mae'], cv=3)
gs.fit(data)

print(f"\nBest RMSE: {gs.best_score['rmse']:.4f}")
print(f"Best params: {gs.best_params['rmse']}")

# --- Train final model and make predictions ---
from surprise.model_selection import train_test_split

trainset, testset = train_test_split(data, test_size=0.2, random_state=42)

# Use best parameters
best_algo = SVD(**gs.best_params['rmse'])
best_algo.fit(trainset)

# Predict
predictions = best_algo.test(testset)
print(f"\nTest RMSE: {accuracy.rmse(predictions):.4f}")

# Predict for a specific user-item pair
pred = best_algo.predict(uid='196', iid='302')
print(f"\nPredicted rating for user 196, item 302: {pred.est:.2f}")

# --- Get Top-N Recommendations ---
from collections import defaultdict

def get_top_n(predictions, n=10):
    """Return top-N recommendations for each user."""
    top_n = defaultdict(list)
    for uid, iid, true_r, est, _ in predictions:
        top_n[uid].append((iid, est))
    
    # Sort and keep top N
    for uid, user_ratings in top_n.items():
        user_ratings.sort(key=lambda x: x[1], reverse=True)
        top_n[uid] = user_ratings[:n]
    
    return top_n

# Get predictions for all unrated items
trainset_full = data.build_full_trainset()
best_algo.fit(trainset_full)
testset_anti = trainset_full.build_anti_testset()  # All unrated user-item pairs
predictions_all = best_algo.test(testset_anti)

top_n = get_top_n(predictions_all, n=5)

# Show recommendations for user '196'
print(f"\nTop 5 recommendations for user 196:")
for iid, rating in top_n['196']:
    print(f"  Item {iid}: predicted rating = {rating:.2f}")
```

### Example 4: Item-Based CF with Implicit Feedback

```python
# pip install implicit
import numpy as np
import scipy.sparse as sparse
from implicit.als import AlternatingLeastSquares
from implicit.nearest_neighbours import CosineRecommender

# Create synthetic implicit feedback data (e.g., purchase counts)
np.random.seed(42)
n_users = 1000
n_items = 500

# Sparse interaction matrix (user × item)
# Values represent interaction strength (views, purchases, time spent)
rows = np.random.randint(0, n_users, 5000)
cols = np.random.randint(0, n_items, 5000)
data = np.random.randint(1, 10, 5000)  # Interaction counts

user_item = sparse.csr_matrix((data, (rows, cols)), shape=(n_users, n_items))
print(f"Interaction matrix: {user_item.shape}")
print(f"Sparsity: {1 - user_item.nnz / (n_users * n_items):.2%}")
# Output: Sparsity: 99.01% (typical for real-world data)

# --- ALS for Implicit Feedback ---
model = AlternatingLeastSquares(
    factors=64,              # Latent factor dimensions
    regularization=0.1,      # L2 regularization
    iterations=15,           # Number of ALS iterations
    random_state=42
)

# Confidence matrix: c_ui = 1 + alpha * r_ui
# Higher interaction count → higher confidence
alpha = 40
confidence = (user_item * alpha).astype('double')

# Train the model (expects item-user matrix)
model.fit(confidence.T)  # Transpose: items × users

# Get recommendations for user 0
user_id = 0
recommendations = model.recommend(
    user_id,
    user_item[user_id],      # User's interaction history
    N=10,                     # Number of recommendations
    filter_already_liked_items=True  # Don't recommend already seen items
)

print(f"\nTop 10 recommendations for user {user_id}:")
for item_id, score in zip(recommendations[0], recommendations[1]):
    print(f"  Item {item_id}: score = {score:.4f}")

# Find similar items
item_id = 42
similar = model.similar_items(item_id, N=5)
print(f"\nItems similar to item {item_id}:")
for sim_item, score in zip(similar[0], similar[1]):
    print(f"  Item {sim_item}: similarity = {score:.4f}")

# Find similar users
similar_users = model.similar_users(user_id, N=5)
print(f"\nUsers similar to user {user_id}:")
for sim_user, score in zip(similar_users[0], similar_users[1]):
    print(f"  User {sim_user}: similarity = {score:.4f}")
```

### Example 5: Deep Learning Recommendation (Neural CF with PyTorch)

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import Dataset, DataLoader
import numpy as np
import pandas as pd

class NCFDataset(Dataset):
    """Dataset for Neural Collaborative Filtering."""
    def __init__(self, users, items, ratings):
        self.users = torch.LongTensor(users)
        self.items = torch.LongTensor(items)
        self.ratings = torch.FloatTensor(ratings)
    
    def __len__(self):
        return len(self.users)
    
    def __getitem__(self, idx):
        return self.users[idx], self.items[idx], self.ratings[idx]

class NeuralCollaborativeFiltering(nn.Module):
    """
    Neural Collaborative Filtering combining:
    - GMF (Generalized Matrix Factorization): element-wise product
    - MLP (Multi-Layer Perceptron): learned interactions
    """
    def __init__(self, n_users, n_items, embed_dim=32, mlp_dims=[64, 32, 16]):
        super().__init__()
        
        # GMF embeddings
        self.gmf_user_embed = nn.Embedding(n_users, embed_dim)
        self.gmf_item_embed = nn.Embedding(n_items, embed_dim)
        
        # MLP embeddings (separate from GMF for flexibility)
        self.mlp_user_embed = nn.Embedding(n_users, embed_dim)
        self.mlp_item_embed = nn.Embedding(n_items, embed_dim)
        
        # MLP layers
        mlp_layers = []
        input_dim = embed_dim * 2  # Concatenated user + item embeddings
        for dim in mlp_dims:
            mlp_layers.append(nn.Linear(input_dim, dim))
            mlp_layers.append(nn.ReLU())
            mlp_layers.append(nn.Dropout(0.2))
            input_dim = dim
        self.mlp = nn.Sequential(*mlp_layers)
        
        # Final prediction layer (GMF output + MLP output)
        self.output_layer = nn.Linear(embed_dim + mlp_dims[-1], 1)
        
        # Initialize weights
        self._init_weights()
    
    def _init_weights(self):
        """Xavier initialization for better convergence."""
        for module in self.modules():
            if isinstance(module, nn.Embedding):
                nn.init.normal_(module.weight, std=0.01)
            elif isinstance(module, nn.Linear):
                nn.init.xavier_uniform_(module.weight)
                nn.init.zeros_(module.bias)
    
    def forward(self, user_ids, item_ids):
        # GMF path: element-wise product
        gmf_user = self.gmf_user_embed(user_ids)
        gmf_item = self.gmf_item_embed(item_ids)
        gmf_output = gmf_user * gmf_item  # Element-wise product
        
        # MLP path: concatenation + deep layers
        mlp_user = self.mlp_user_embed(user_ids)
        mlp_item = self.mlp_item_embed(item_ids)
        mlp_input = torch.cat([mlp_user, mlp_item], dim=-1)
        mlp_output = self.mlp(mlp_input)
        
        # Combine GMF and MLP
        combined = torch.cat([gmf_output, mlp_output], dim=-1)
        prediction = self.output_layer(combined)
        
        return prediction.squeeze()

# --- Training Pipeline ---
def train_ncf():
    # Generate synthetic data
    np.random.seed(42)
    n_users, n_items = 1000, 500
    n_interactions = 50000
    
    users = np.random.randint(0, n_users, n_interactions)
    items = np.random.randint(0, n_items, n_interactions)
    # Simulate ratings (1-5) with some pattern
    ratings = np.clip(np.random.normal(3.5, 1.0, n_interactions), 1, 5)
    
    # Train/test split
    split_idx = int(0.8 * n_interactions)
    train_dataset = NCFDataset(users[:split_idx], items[:split_idx], ratings[:split_idx])
    test_dataset = NCFDataset(users[split_idx:], items[split_idx:], ratings[split_idx:])
    
    train_loader = DataLoader(train_dataset, batch_size=256, shuffle=True)
    test_loader = DataLoader(test_dataset, batch_size=256)
    
    # Initialize model
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model = NeuralCollaborativeFiltering(n_users, n_items).to(device)
    
    optimizer = optim.Adam(model.parameters(), lr=0.001, weight_decay=1e-5)
    criterion = nn.MSELoss()
    
    # Training loop
    n_epochs = 10
    for epoch in range(n_epochs):
        model.train()
        total_loss = 0
        
        for users_batch, items_batch, ratings_batch in train_loader:
            users_batch = users_batch.to(device)
            items_batch = items_batch.to(device)
            ratings_batch = ratings_batch.to(device)
            
            optimizer.zero_grad()
            predictions = model(users_batch, items_batch)
            loss = criterion(predictions, ratings_batch)
            loss.backward()
            optimizer.step()
            
            total_loss += loss.item()
        
        # Evaluate
        model.eval()
        test_loss = 0
        with torch.no_grad():
            for users_batch, items_batch, ratings_batch in test_loader:
                users_batch = users_batch.to(device)
                items_batch = items_batch.to(device)
                ratings_batch = ratings_batch.to(device)
                predictions = model(users_batch, items_batch)
                test_loss += criterion(predictions, ratings_batch).item()
        
        if (epoch + 1) % 2 == 0:
            print(f"Epoch {epoch+1}/{n_epochs} | "
                  f"Train Loss: {total_loss/len(train_loader):.4f} | "
                  f"Test RMSE: {np.sqrt(test_loss/len(test_loader)):.4f}")
    
    return model

# Train the model
model = train_ncf()
```

### Example 6: Hybrid System with LightFM

```python
# pip install lightfm
from lightfm import LightFM
from lightfm.datasets import fetch_movielens
from lightfm.evaluation import precision_at_k, recall_at_k, auc_score
import numpy as np

# Load MovieLens data with item features
data = fetch_movielens(min_rating=3.0)  # Treat 3+ as positive

print(f"Training interactions: {data['train'].shape}")
print(f"Test interactions: {data['test'].shape}")
print(f"Item features: {data['item_features'].shape}")

# --- Pure Collaborative Filtering ---
model_cf = LightFM(
    no_components=64,        # Latent factor dimensions
    loss='warp',             # Weighted Approximate-Rank Pairwise loss
    learning_rate=0.05,
    random_state=42
)
model_cf.fit(data['train'], epochs=30, num_threads=4)

# --- Hybrid Model (CF + Content Features) ---
model_hybrid = LightFM(
    no_components=64,
    loss='warp',
    learning_rate=0.05,
    random_state=42
)
model_hybrid.fit(
    data['train'],
    item_features=data['item_features'],  # Genre information
    epochs=30,
    num_threads=4
)

# --- Evaluate Both Models ---
print("\n=== Model Comparison ===")

# Precision@5
p_cf = precision_at_k(model_cf, data['test'], k=5).mean()
p_hybrid = precision_at_k(model_hybrid, data['test'], 
                          item_features=data['item_features'], k=5).mean()
print(f"Precision@5  - CF: {p_cf:.4f} | Hybrid: {p_hybrid:.4f}")

# AUC
auc_cf = auc_score(model_cf, data['test']).mean()
auc_hybrid = auc_score(model_hybrid, data['test'],
                       item_features=data['item_features']).mean()
print(f"AUC Score    - CF: {auc_cf:.4f} | Hybrid: {auc_hybrid:.4f}")

# --- Get Recommendations for a User ---
def get_lightfm_recommendations(model, user_id, item_features=None, n=10):
    """Get top-N recommendations using LightFM."""
    n_items = data['train'].shape[1]
    
    # Predict scores for all items
    scores = model.predict(user_id, np.arange(n_items), 
                          item_features=item_features)
    
    # Get top-N item indices
    top_items = np.argsort(-scores)[:n]
    
    return list(zip(top_items, scores[top_items]))

# Recommendations for user 5
print(f"\nHybrid recommendations for user 5:")
recs = get_lightfm_recommendations(model_hybrid, 5, data['item_features'])
for item_id, score in recs:
    print(f"  Item {item_id}: score = {score:.3f}")
```

### Pro Tips

> **Pro Tip 1:** Always use implicit feedback when possible. Explicit ratings are sparse and biased (people rate things they feel strongly about). Implicit signals (clicks, views, time spent) are abundant and less biased.

> **Pro Tip 2:** For production systems, use a two-stage architecture:
> - Stage 1: Fast candidate retrieval (ANN search with FAISS/ScaNN) → ~1000 candidates
> - Stage 2: Expensive ranking model (deep learning) → final 10-50 items

> **Pro Tip 3:** Always add popularity-based fallback. When your model can't make confident predictions, fall back to "most popular in user's segment."

> **Pro Tip 4:** Monitor for "echo chambers." Track recommendation diversity over time. If users only see similar content, engagement may initially rise but long-term retention drops.

---

## Common Mistakes

### 1. Ignoring the Cold Start Problem
**Mistake:** Building only CF-based system, then wondering why new users/items get bad recommendations.
**Fix:** Always have a fallback strategy. Use content-based for new items, popularity/demographic-based for new users.

### 2. Training on Future Data (Data Leakage)
**Mistake:** Random train/test split where test set contains interactions that happened BEFORE training interactions.
**Fix:** Always split by time. Training data = past, test data = future.

```python
# WRONG: Random split
from sklearn.model_selection import train_test_split
train, test = train_test_split(data, test_size=0.2)  # Temporal leakage!

# RIGHT: Time-based split
data_sorted = data.sort_values('timestamp')
split_point = int(0.8 * len(data_sorted))
train = data_sorted[:split_point]
test = data_sorted[split_point:]
```

### 3. Evaluating with Accuracy Metrics Only
**Mistake:** Only optimizing RMSE/MAE without considering diversity, coverage, or novelty.
**Fix:** Use a suite of metrics. A system that always recommends the top-10 most popular items might have decent RMSE but terrible user experience.

### 4. Not Handling Position Bias
**Mistake:** Items shown at top of list get clicked more → model thinks they're better → shows them more.
**Fix:** Use inverse propensity scoring or randomized experiments to debias.

### 5. Treating Missing Ratings as Zero
**Mistake:** In matrix factorization, filling missing values with 0 (user probably didn't dislike the item — they just haven't seen it).
**Fix:** Only train on observed interactions. Use implicit feedback formulation with confidence weights.

### 6. Not Considering Scalability
**Mistake:** Using user-based CF with cosine similarity computed against ALL users at query time.
**Fix:** Pre-compute similarities, use ANN for retrieval, or switch to model-based approaches.

### 7. Ignoring Temporal Dynamics
**Mistake:** Treating all interactions equally regardless of when they happened.
**Fix:** Apply time decay (recent interactions matter more) or use sequence-aware models.

---

## Interview Questions

### Conceptual Questions

**Q1: Explain the difference between content-based and collaborative filtering.**
> Content-based uses item features to recommend similar items to what user liked. CF uses patterns from all users' behavior — it doesn't need item features, only interaction data. CF can find surprising connections (e.g., users who like X also like Y even if X and Y seem unrelated), while content-based is limited by the features you define.

**Q2: What is the cold start problem and how do you solve it?**
> Cold start occurs when the system can't make good recommendations for new users (no history) or new items (no interactions). Solutions: For new users — use demographic info, ask preferences during onboarding, show popular items. For new items — use content features, promote to random segments for exploration. Hybrid approaches mitigate cold start by combining CF with content-based methods.

**Q3: How does matrix factorization work? Why is it better than raw CF?**
> MF decomposes the sparse user-item matrix into two dense matrices (user factors × item factors). This learns latent representations that capture hidden patterns (genres, quality, complexity). It's better than raw CF because: (1) it handles sparsity through generalization, (2) it's computationally efficient (O(k) per prediction), (3) regularization prevents overfitting.

**Q4: Explain the exploration-exploitation tradeoff in recommendations.**
> Exploitation: recommend items we're confident the user will like (safe, predictable). Exploration: recommend uncertain items to gather information about preferences (risky but informative). Pure exploitation leads to filter bubbles; pure exploration frustrates users. Solutions include epsilon-greedy (random exploration with probability ε), Thompson Sampling (sample from posterior), and Upper Confidence Bound (optimistic estimate).

**Q5: How would you evaluate a recommendation system in production?**
> Offline: NDCG, Precision@K, Recall@K on held-out data. Online: A/B test measuring CTR, conversion rate, session length, revenue. Long-term: user retention, content diversity consumed, creator/seller fairness. Always split data by time (not random) for offline evaluation.

### System Design Questions

**Q6: Design a recommendation system for an e-commerce site with 10M users and 1M products.**
> Architecture: (1) Offline pipeline: Train MF/DL model daily on all interactions. Pre-compute item embeddings. (2) Online candidate generation: User embedding → ANN search (FAISS) → 1000 candidates. (3) Online ranking: Feature-rich model (user features + item features + context) → top 50. (4) Re-ranking: apply business rules (diversity, new item boost, margin). (5) Serving: Cache popular users, real-time for active users. Cold start: content-based + popularity fallback.

**Q7: How do you handle implicit feedback differently from explicit ratings?**
> Implicit feedback (clicks, views, purchases) has no negative signals — absence is ambiguous (user didn't see item vs. didn't like it). Use confidence-weighted formulation where more interactions = higher confidence. Use ranking losses (BPR, WARP) instead of pointwise (MSE). Sample negative items carefully (not all unobserved items are negative).

### Coding Questions

**Q8: Implement cosine similarity between two users given a rating matrix.**
```python
def cosine_sim(user1, user2, matrix):
    # Get common items (both rated)
    common = ~(np.isnan(matrix[user1]) | np.isnan(matrix[user2]))
    if common.sum() == 0:
        return 0
    a = matrix[user1][common]
    b = matrix[user2][common]
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```

**Q9: What loss functions are used in recommendation systems?**
> - **MSE/RMSE:** Rating prediction (explicit feedback)
> - **Binary Cross-Entropy:** Click prediction (implicit, pointwise)
> - **BPR Loss (Bayesian Personalized Ranking):** Pairwise — positive item should score higher than negative
> - **WARP Loss:** Weighted approximate rank — focuses on top of ranking
> - **Contrastive Loss:** For embedding-based models

**Q10: How do you handle the "Harry Potter problem" (universally popular items)?**
> Popular items get recommended to everyone, drowning out personalized suggestions. Solutions: (1) Subtract popularity bias in scoring. (2) Use item bias terms in MF (b_i captures popularity, latent factors capture preference). (3) Apply IDF-like weighting (rare interactions are more informative). (4) Use ranking metrics that reward diversity.

---

## Quick Reference

### Algorithm Selection Guide

| Scenario | Best Approach | Why |
|----------|--------------|-----|
| New system, little data | Popularity-based | No ML needed |
| Rich item metadata | Content-based | Doesn't need user history |
| Lots of explicit ratings | Matrix Factorization (SVD) | Handles sparsity well |
| Implicit feedback (clicks) | ALS / BPR | Designed for confidence-weighted data |
| Need real-time serving | Two-tower + ANN | Pre-computed embeddings |
| Sequence matters | Transformer-based (SASRec) | Captures temporal patterns |
| Cold start critical | Hybrid (LightFM) | Uses both features and interactions |
| Very large scale | Two-stage (retrieval + ranking) | Only feasible approach at scale |

### Key Formulas

| Formula | Use |
|---------|-----|
| $\text{cos}(A,B) = \frac{A \cdot B}{\|A\|\|B\|}$ | Similarity measurement |
| $R \approx U \times V^T$ | Matrix factorization |
| $\hat{r}_{ui} = \mu + b_u + b_i + p_u^T q_i$ | SVD prediction |
| $NDCG@K = \frac{DCG@K}{IDCG@K}$ | Ranking evaluation |

### Python Libraries

| Library | Best For | Scale |
|---------|----------|-------|
| `surprise` | Prototyping, explicit ratings | Small-Medium |
| `implicit` | Implicit feedback, ALS | Medium-Large |
| `lightfm` | Hybrid models | Medium |
| `recbole` | Research, many algorithms | Any |
| `tensorflow-recommenders` | Production DL models | Large |
| `merlin` (NVIDIA) | GPU-accelerated production | Very Large |
| `Spark ALS` | Distributed training | Very Large |

### Key Takeaways

- **Start simple:** Popularity → Content-based → CF → Hybrid → Deep Learning
- **Evaluate properly:** Time-based splits, multiple metrics, A/B tests
- **Cold start is real:** Always have fallback strategies
- **Scale matters:** Two-stage architecture for production
- **Diversity is important:** Optimize beyond accuracy metrics
- **Implicit > Explicit:** More data, less bias, more realistic
- **Monitor continuously:** User preferences drift over time
