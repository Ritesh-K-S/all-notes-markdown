# Linear Algebra for Machine Learning

## Overview
Linear Algebra is the **language of data** in machine learning. Every dataset is a matrix, every feature is a vector, and every transformation (rotation, scaling, projection) is a matrix operation. Without linear algebra, you cannot understand how neural networks work, how PCA reduces dimensions, or how recommendation systems find similar users.

---

## Table of Contents
1. [Scalars, Vectors, Matrices, and Tensors](#1-scalars-vectors-matrices-and-tensors)
2. [Vector Operations](#2-vector-operations)
3. [Matrix Operations](#3-matrix-operations)
4. [Special Matrices](#4-special-matrices)
5. [Linear Transformations](#5-linear-transformations)
6. [Eigenvalues and Eigenvectors](#6-eigenvalues-and-eigenvectors)
7. [Singular Value Decomposition (SVD)](#7-singular-value-decomposition-svd)
8. [Matrix Decompositions](#8-matrix-decompositions)
9. [Norms and Distances](#9-norms-and-distances)
10. [Applications in ML](#10-applications-in-ml)

---

## 1. Scalars, Vectors, Matrices, and Tensors

### What It Is
These are the fundamental building blocks — think of them as different "containers" for numbers:

| Object | Dimensions | Example | ML Analogy |
|--------|-----------|---------|------------|
| Scalar | 0D (single number) | Temperature = 72°F | A single prediction score |
| Vector | 1D (list of numbers) | [height, weight, age] | A single data point's features |
| Matrix | 2D (table of numbers) | Spreadsheet of data | Entire dataset |
| Tensor | nD (multi-dimensional) | Color image (H×W×3) | Batch of images in deep learning |

### Why It Matters
- **Every dataset** you work with is a matrix (rows = samples, columns = features)
- **Every image** is a tensor (height × width × color channels)
- **Every neural network weight** is stored in matrices
- **Every word embedding** (Word2Vec, BERT) is a vector

### How It Works

```
Scalar:     5
            (just a number)

Vector:     [3, 7, 2, 1]
            (a row or column of numbers)

Matrix:     [[1, 2, 3],
             [4, 5, 6],
             [7, 8, 9]]
            (a 2D grid of numbers)

Tensor:     A "stack" of matrices
            (think: a Rubik's cube of numbers)
```

**Real-world analogy:** 
- Scalar = a single temperature reading
- Vector = today's weather (temp, humidity, wind speed, pressure)
- Matrix = a week of weather data (7 days × 4 measurements)
- Tensor = a year of weather across 50 cities (365 × 50 × 4)

### Code Examples

```python
import numpy as np

# === SCALAR ===
scalar = 5
print(f"Scalar: {scalar}, Shape: {np.array(scalar).shape}")
# Output: Scalar: 5, Shape: ()

# === VECTOR ===
# A feature vector for a person: [height_cm, weight_kg, age]
person = np.array([175, 70, 25])
print(f"Vector: {person}, Shape: {person.shape}")
# Output: Vector: [175  70  25], Shape: (3,)

# === MATRIX ===
# Dataset: 4 people × 3 features (height, weight, age)
dataset = np.array([
    [175, 70, 25],
    [160, 55, 30],
    [180, 85, 45],
    [165, 60, 22]
])
print(f"Matrix Shape: {dataset.shape}")  # (4, 3) → 4 samples, 3 features
# Output: Matrix Shape: (4, 3)

# === TENSOR ===
# A batch of 32 RGB images, each 224×224 pixels
image_batch = np.random.rand(32, 224, 224, 3)
print(f"Tensor Shape: {image_batch.shape}")
# Output: Tensor Shape: (32, 224, 224, 3)
# Interpretation: 32 images, 224 height, 224 width, 3 color channels (RGB)

# === IMPORTANT: Row vs Column Vectors ===
row_vector = np.array([[1, 2, 3]])       # Shape: (1, 3)
col_vector = np.array([[1], [2], [3]])   # Shape: (3, 1)
print(f"Row vector shape: {row_vector.shape}")    # (1, 3)
print(f"Column vector shape: {col_vector.shape}") # (3, 1)
```

### Common Mistakes
1. **Confusing shape (3,) with (3,1) or (1,3)** — NumPy's 1D arrays are neither row nor column vectors. Use `.reshape(-1, 1)` for column vectors.
2. **Assuming all tensors are 3D** — A tensor is ANY multi-dimensional array (vectors and matrices are technically tensors too).
3. **Forgetting that ML frameworks expect specific shapes** — Most models expect input as (batch_size, features), not just (features,).

---

## 2. Vector Operations

### What It Is
Operations you can perform on vectors — adding them, scaling them, measuring angles between them, and projecting one onto another.

### Why It Matters
- **Dot product** → measures similarity (used in recommendation systems, attention mechanisms)
- **Vector addition** → combining features, residual connections in neural networks
- **Projection** → dimensionality reduction, finding the "shadow" of one vector on another

### How It Works

#### Vector Addition & Subtraction
```
    [1, 2, 3] + [4, 5, 6] = [5, 7, 9]
    
    Geometrically: Place vectors tip-to-tail
    
        /→ b
    a →/
    ────────→ a + b
```

#### Scalar Multiplication
```
    3 × [1, 2, 3] = [3, 6, 9]
    
    Geometrically: Stretches the vector by factor 3
    
    →    becomes    ──────→
```

#### Dot Product (Inner Product)
$$\mathbf{a} \cdot \mathbf{b} = \sum_{i=1}^{n} a_i b_i = |\mathbf{a}||\mathbf{b}|\cos\theta$$

```
    [1, 2, 3] · [4, 5, 6] = 1×4 + 2×5 + 3×6 = 32
    
    Interpretation:
    - Positive → vectors point in similar directions
    - Zero → vectors are perpendicular (orthogonal)
    - Negative → vectors point in opposite directions
```

**Analogy:** The dot product tells you "how much" two vectors agree. Like asking two friends to rate 3 movies (1-5 scale) — a high dot product means they have similar tastes.

#### Cross Product (3D only)
$$\mathbf{a} \times \mathbf{b} = |\mathbf{a}||\mathbf{b}|\sin\theta \cdot \hat{n}$$

Produces a vector **perpendicular** to both inputs. Used in 3D graphics/computer vision.

### Code Examples

```python
import numpy as np

# === BASIC OPERATIONS ===
a = np.array([1, 2, 3])
b = np.array([4, 5, 6])

# Addition and subtraction
print(f"a + b = {a + b}")          # [5, 7, 9]
print(f"a - b = {a - b}")          # [-3, -3, -3]

# Scalar multiplication
print(f"3 * a = {3 * a}")          # [3, 6, 9]

# === DOT PRODUCT — The Most Important Operation in ML ===
dot_product = np.dot(a, b)         # 1*4 + 2*5 + 3*6 = 32
print(f"a · b = {dot_product}")

# Alternative ways to compute dot product
print(f"a @ b = {a @ b}")          # Using @ operator (Python 3.5+)
print(f"sum = {np.sum(a * b)}")    # Element-wise multiply then sum

# === MEASURING SIMILARITY WITH COSINE ===
def cosine_similarity(v1, v2):
    """Measures angle between vectors (ignores magnitude)"""
    dot = np.dot(v1, v2)
    norm_v1 = np.linalg.norm(v1)   # Length of v1
    norm_v2 = np.linalg.norm(v2)   # Length of v2
    return dot / (norm_v1 * norm_v2)

# Similar vectors
movie_lover_1 = np.array([5, 4, 1, 2])  # Ratings for 4 movies
movie_lover_2 = np.array([4, 5, 1, 1])  # Similar taste
random_person = np.array([1, 1, 5, 5])  # Opposite taste

print(f"Similar users: {cosine_similarity(movie_lover_1, movie_lover_2):.4f}")  # ~0.98
print(f"Different users: {cosine_similarity(movie_lover_1, random_person):.4f}")  # ~0.42

# === VECTOR PROJECTION ===
# Project vector a onto vector b
# "How much of a goes in the direction of b?"
def project(a, b):
    """Project vector a onto vector b"""
    return (np.dot(a, b) / np.dot(b, b)) * b

a = np.array([3, 4])
b = np.array([1, 0])  # x-axis
projection = project(a, b)
print(f"Projection of {a} onto {b} = {projection}")  # [3, 0]
# The "shadow" of [3,4] on the x-axis is [3,0]

# === LINEAR INDEPENDENCE ===
# Vectors are linearly independent if no vector can be written as 
# a combination of the others
v1 = np.array([1, 0, 0])
v2 = np.array([0, 1, 0])
v3 = np.array([1, 1, 0])  # v3 = v1 + v2 → NOT independent!

# Check using matrix rank
matrix = np.array([v1, v2, v3])
rank = np.linalg.matrix_rank(matrix)
print(f"Rank: {rank}, Vectors: {len(matrix)}")
print(f"Linearly independent: {rank == len(matrix)}")  # False
```

> **Pro Tip:** In ML, cosine similarity is preferred over dot product for comparing embeddings because it's invariant to vector magnitude. A document with 1000 words shouldn't be considered "more similar" just because it's longer.

### Common Mistakes
1. **Using dot product when you need cosine similarity** — Dot product is affected by vector magnitude; cosine is not.
2. **Forgetting that orthogonal doesn't mean "opposite"** — Perpendicular vectors (dot=0) are simply unrelated, not opposite.
3. **Mixing up element-wise multiplication (`*`) with dot product (`@`)** — `a * b` gives a vector; `a @ b` gives a scalar.

---

## 3. Matrix Operations

### What It Is
Operations on 2D grids of numbers — multiplication, transposition, inversion. These are the core computations in every ML algorithm.

### Why It Matters
- **Matrix multiplication** = applying a neural network layer
- **Matrix inverse** = solving systems of equations (linear regression)
- **Transpose** = swapping rows and columns (reshaping data)

### How It Works

#### Matrix Multiplication
$$C = AB \text{ where } C_{ij} = \sum_{k=1}^{n} A_{ik} B_{kj}$$

```
Rule: (m×n) @ (n×p) = (m×p)
The inner dimensions must match!

[1, 2]   [5, 6]     [1×5+2×7, 1×6+2×8]     [19, 22]
[3, 4] × [7, 8]  =  [3×5+4×7, 3×6+4×8]  =  [43, 50]

(2×2)  @  (2×2)  =  (2×2)
```

**Analogy:** Matrix multiplication is like a factory assembly line. Each row of A is a "recipe" that combines columns of B into a final product.

#### Matrix Transpose
$$[A^T]_{ij} = A_{ji}$$

```
Original:       Transposed:
[1, 2, 3]      [1, 4]
[4, 5, 6]      [2, 5]
                [3, 6]
(2×3)     →    (3×2)
```

#### Matrix Inverse
$$AA^{-1} = A^{-1}A = I$$

Only square matrices with non-zero determinant have inverses.

**Analogy:** If a matrix is a "lock" that transforms data, the inverse is the "key" that undoes the transformation.

### Code Examples

```python
import numpy as np

# === MATRIX MULTIPLICATION ===
A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])

# Three equivalent ways
C = A @ B                    # Preferred (Python 3.5+)
C = np.matmul(A, B)         # Explicit function
C = np.dot(A, B)            # Works for 2D arrays
print(f"A @ B =\n{C}")
# [[19, 22],
#  [43, 50]]

# === SHAPE COMPATIBILITY ===
# Neural network layer: 100 samples, 784 features, 256 neurons
X = np.random.rand(100, 784)    # Input: 100 images, 784 pixels each
W = np.random.rand(784, 256)    # Weights: connecting 784 inputs to 256 neurons
output = X @ W                   # Output: 100 images × 256 neuron activations
print(f"Input: {X.shape}, Weights: {W.shape}, Output: {output.shape}")
# Input: (100, 784), Weights: (784, 256), Output: (100, 256)

# === TRANSPOSE ===
A = np.array([[1, 2, 3], [4, 5, 6]])
print(f"Original shape: {A.shape}")       # (2, 3)
print(f"Transposed shape: {A.T.shape}")   # (3, 2)

# Important identity: (AB)^T = B^T @ A^T
A = np.random.rand(3, 4)
B = np.random.rand(4, 5)
assert np.allclose((A @ B).T, B.T @ A.T)  # Always true!

# === MATRIX INVERSE ===
A = np.array([[4, 7], [2, 6]])
A_inv = np.linalg.inv(A)
print(f"A × A_inv =\n{np.round(A @ A_inv, 10)}")
# [[1, 0],
#  [0, 1]]  ← Identity matrix!

# === SOLVING LINEAR SYSTEMS (Linear Regression!) ===
# The Normal Equation: θ = (X^T X)^(-1) X^T y
# This IS linear regression!
np.random.seed(42)
X = np.column_stack([np.ones(100), np.random.rand(100)])  # Add bias column
true_weights = np.array([3, 7])  # y = 3 + 7x
y = X @ true_weights + np.random.randn(100) * 0.5  # Add noise

# Solve using normal equation
theta = np.linalg.inv(X.T @ X) @ X.T @ y
print(f"Recovered weights: {theta}")  # Close to [3, 7]

# Pro tip: Use lstsq instead (numerically stable)
theta_stable, _, _, _ = np.linalg.lstsq(X, y, rcond=None)
print(f"Stable solution: {theta_stable}")

# === DETERMINANT ===
A = np.array([[4, 7], [2, 6]])
det = np.linalg.det(A)
print(f"det(A) = {det}")  # 4*6 - 7*2 = 10
# If det = 0, matrix has no inverse (singular)

# === ELEMENT-WISE vs MATRIX OPERATIONS ===
A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])

print(f"Element-wise multiply:\n{A * B}")   # [[5,12],[21,32]]
print(f"Matrix multiply:\n{A @ B}")          # [[19,22],[43,50]]
# THESE ARE COMPLETELY DIFFERENT!
```

> **Warning:** Never use `np.linalg.inv()` in production for solving linear systems. Use `np.linalg.solve()` or `np.linalg.lstsq()` — they're faster and more numerically stable.

### Common Mistakes
1. **Matrix multiplication is NOT commutative** — `A @ B ≠ B @ A` in general
2. **Confusing `*` with `@`** — `*` is element-wise, `@` is matrix multiplication
3. **Inverting matrices unnecessarily** — Use `np.linalg.solve(A, b)` instead of `inv(A) @ b`
4. **Not checking if a matrix is invertible** — Check `np.linalg.det(A) != 0` or use `try/except`

---

## 4. Special Matrices

### What It Is
Certain matrices with special properties that appear frequently in ML algorithms.

### Why It Matters
- **Identity matrix** → "do nothing" transformation
- **Diagonal matrix** → fast multiplication, feature scaling
- **Symmetric matrix** → covariance matrices, kernel matrices
- **Orthogonal matrix** → rotations that preserve distances
- **Positive definite** → guarantees convexity in optimization

### How It Works

| Matrix Type | Property | ML Usage |
|-------------|----------|----------|
| Identity (I) | AI = IA = A | Regularization baseline |
| Diagonal | Non-zero only on diagonal | Feature scaling, fast inverse |
| Symmetric | $A = A^T$ | Covariance matrix, kernel matrix |
| Orthogonal | $A^T A = I$ | PCA, rotations, QR decomposition |
| Positive Definite | $x^T A x > 0$ for all x≠0 | Guarantees unique minimum |
| Sparse | Mostly zeros | Efficient storage, graph adjacency |

### Code Examples

```python
import numpy as np

# === IDENTITY MATRIX ===
I = np.eye(3)  # 3×3 identity
print(f"Identity:\n{I}")
# [[1, 0, 0],
#  [0, 1, 0],
#  [0, 0, 1]]

A = np.array([[1, 2], [3, 4]])
print(f"A @ I = A? {np.allclose(A @ np.eye(2), A)}")  # True

# === DIAGONAL MATRIX ===
# Creating from a vector
d = np.array([2, 3, 4])
D = np.diag(d)
print(f"Diagonal:\n{D}")
# [[2, 0, 0],
#  [0, 3, 0],
#  [0, 0, 4]]

# Fast inverse of diagonal: just reciprocal of elements
D_inv = np.diag(1.0 / d)
print(f"D @ D_inv = I? {np.allclose(D @ D_inv, np.eye(3))}")  # True

# === SYMMETRIC MATRIX ===
# Covariance matrix is always symmetric
data = np.random.rand(100, 3)  # 100 samples, 3 features
cov_matrix = np.cov(data.T)   # Covariance matrix
print(f"Symmetric? {np.allclose(cov_matrix, cov_matrix.T)}")  # True

# === ORTHOGONAL MATRIX ===
# Rotation matrix (2D rotation by 45 degrees)
theta = np.pi / 4
R = np.array([
    [np.cos(theta), -np.sin(theta)],
    [np.sin(theta),  np.cos(theta)]
])
print(f"R^T @ R = I? {np.allclose(R.T @ R, np.eye(2))}")  # True
print(f"det(R) = {np.linalg.det(R):.4f}")  # 1.0 (preserves volume)

# Orthogonal matrices preserve vector lengths!
v = np.array([3, 4])
v_rotated = R @ v
print(f"Original length: {np.linalg.norm(v):.4f}")   # 5.0
print(f"Rotated length: {np.linalg.norm(v_rotated):.4f}")  # 5.0

# === POSITIVE DEFINITE ===
# A symmetric matrix A is positive definite if all eigenvalues > 0
A = np.array([[2, 1], [1, 3]])  # Symmetric
eigenvalues = np.linalg.eigvals(A)
print(f"Eigenvalues: {eigenvalues}")  # Both positive → positive definite
print(f"Positive definite? {np.all(eigenvalues > 0)}")  # True

# Cholesky decomposition only works on positive definite matrices
L = np.linalg.cholesky(A)  # A = L @ L^T
print(f"A = L @ L^T? {np.allclose(A, L @ L.T)}")  # True

# === SPARSE MATRICES ===
from scipy import sparse

# In real ML: most elements are zero (text data, graphs)
# Dense: stores ALL elements → wastes memory
# Sparse: stores only non-zero elements → efficient

# Example: document-term matrix (10000 docs × 50000 words)
# Most entries are 0 (a doc uses only ~100 unique words)
sparse_matrix = sparse.random(10000, 50000, density=0.002, format='csr')
print(f"Non-zero elements: {sparse_matrix.nnz}")
print(f"Memory (sparse): {sparse_matrix.data.nbytes / 1024:.1f} KB")
print(f"Memory (dense): {10000 * 50000 * 8 / 1024 / 1024:.1f} MB")
```

### Common Mistakes
1. **Assuming covariance matrices are always invertible** — They're singular when features are linearly dependent (add regularization: `cov + ε*I`)
2. **Not exploiting matrix structure** — Diagonal inverse is O(n), general inverse is O(n³)
3. **Storing sparse data as dense matrices** — Use `scipy.sparse` for data with >90% zeros

---

## 5. Linear Transformations

### What It Is
A function that maps vectors to vectors while preserving addition and scalar multiplication. Every matrix multiplication IS a linear transformation.

### Why It Matters
- Neural network layers are linear transformations (+ nonlinear activation)
- PCA finds the "best" linear transformation to reduce dimensions
- Understanding what matrices "do" geometrically helps debug ML models

### How It Works

```
What matrices DO geometrically:

Scaling:        Rotation:        Shearing:        Projection:
[2 0] →         [cos -sin] →     [1 k] →          [1 0] →
[0 2]           [sin  cos]       [0 1]             [0 0]

Doubles size    Rotates          Tilts             Flattens to line
```

$$T(\mathbf{v}) = A\mathbf{v}$$

Every linear transformation can be represented as a matrix, and every matrix represents a linear transformation.

### Code Examples

```python
import numpy as np
import matplotlib.pyplot as plt

# === VISUALIZING LINEAR TRANSFORMATIONS ===
# Original unit square corners
square = np.array([[0,0], [1,0], [1,1], [0,1], [0,0]]).T  # 2×5

# Different transformations
transformations = {
    'Scale 2x': np.array([[2, 0], [0, 2]]),
    'Rotate 45°': np.array([[np.cos(np.pi/4), -np.sin(np.pi/4)],
                            [np.sin(np.pi/4),  np.cos(np.pi/4)]]),
    'Shear': np.array([[1, 1], [0, 1]]),
    'Reflect Y': np.array([[-1, 0], [0, 1]]),
    'Project to X': np.array([[1, 0], [0, 0]]),
}

for name, matrix in transformations.items():
    transformed = matrix @ square
    print(f"{name}: det = {np.linalg.det(matrix):.2f}")
    # det > 1: expansion, det < 1: compression
    # det < 0: orientation flipped, det = 0: dimension lost

# === NEURAL NETWORK AS LINEAR TRANSFORM ===
# A single layer: y = Wx + b
# W is the linear transformation, b is the translation

# Input: 3 features
# Output: 2 features (dimension reduction!)
W = np.random.randn(2, 3)  # 2×3 matrix
b = np.random.randn(2)      # bias vector

x = np.array([1.0, 2.0, 3.0])  # Single sample
y = W @ x + b                   # Linear layer output
print(f"Input shape: {x.shape}, Output shape: {y.shape}")
# 3D space → 2D space (information is compressed)

# === CHANGE OF BASIS ===
# Same point, different coordinate systems
# Standard coordinates
point = np.array([3, 4])

# New basis vectors (rotated 45°)
new_basis = np.array([
    [np.cos(np.pi/4), np.cos(np.pi/4)],
    [-np.sin(np.pi/4), np.sin(np.pi/4)]
])

# Coordinates in new basis
new_coords = np.linalg.inv(new_basis) @ point
print(f"Standard coords: {point}")
print(f"Rotated coords: {new_coords}")

# === RANK — How much "information" a matrix carries ===
full_rank = np.array([[1, 2], [3, 4]])       # Rank 2 (full)
low_rank = np.array([[1, 2], [2, 4]])         # Rank 1 (row2 = 2×row1)
print(f"Full rank: {np.linalg.matrix_rank(full_rank)}")  # 2
print(f"Low rank: {np.linalg.matrix_rank(low_rank)}")    # 1

# In ML: Low-rank matrices mean redundant features
# LoRA (Low-Rank Adaptation) exploits this for efficient fine-tuning!
```

### Common Mistakes
1. **Forgetting that linear transformations can't shift the origin** — That's why neural networks need bias terms (affine transformation = linear + translation)
2. **Confusing "linear" in ML with mathematical linearity** — Linear regression is linear in parameters, not necessarily in features

---

## 6. Eigenvalues and Eigenvectors

### What It Is
**Eigenvectors** are special directions that don't change direction when a matrix is applied — they only get stretched or compressed. The amount of stretching is the **eigenvalue**.

$$A\mathbf{v} = \lambda\mathbf{v}$$

Where:
- $A$ = square matrix (the transformation)
- $\mathbf{v}$ = eigenvector (the special direction)
- $\lambda$ = eigenvalue (how much it's stretched)

### Why It Matters
- **PCA** = finding eigenvectors of the covariance matrix (directions of maximum variance)
- **Google PageRank** = finding the dominant eigenvector of the web link matrix
- **Stability analysis** = eigenvalues tell you if a system will explode or converge
- **Spectral clustering** = uses eigenvectors of the graph Laplacian

### How It Works

**Analogy:** Imagine a stretchy rubber sheet (the matrix transformation). Most points on the sheet move in complicated ways. But there are special directions (eigenvectors) where points just slide along the same line — getting closer or farther from the origin. The eigenvalue tells you how much they slide.

```
Matrix A acts on different vectors:

Regular vector:     A × [1,1] = [3,2]     (direction changed!)
                          ↗ →

Eigenvector:        A × [2,1] = [4,2]     (same direction, just scaled by 2!)
                          → →→

Here, [2,1] is an eigenvector with eigenvalue λ = 2
```

**Finding eigenvalues:** Solve $\det(A - \lambda I) = 0$ (characteristic equation)

### Code Examples

```python
import numpy as np

# === BASIC EIGEN DECOMPOSITION ===
A = np.array([[4, 1], [2, 3]])

eigenvalues, eigenvectors = np.linalg.eig(A)
print(f"Eigenvalues: {eigenvalues}")      # [5, 2]
print(f"Eigenvectors:\n{eigenvectors}")   # Each COLUMN is an eigenvector

# Verify: A @ v = λ * v
for i in range(len(eigenvalues)):
    v = eigenvectors[:, i]  # i-th eigenvector (column!)
    lam = eigenvalues[i]     # i-th eigenvalue
    lhs = A @ v
    rhs = lam * v
    print(f"λ={lam:.1f}: A@v = {lhs}, λ*v = {rhs}, Match: {np.allclose(lhs, rhs)}")

# === PCA FROM SCRATCH (The #1 Use Case) ===
np.random.seed(42)

# Generate correlated 2D data
mean = [0, 0]
cov = [[3, 2], [2, 2]]  # Correlated features
data = np.random.multivariate_normal(mean, cov, 200)

# Step 1: Center the data
data_centered = data - data.mean(axis=0)

# Step 2: Compute covariance matrix
cov_matrix = np.cov(data_centered.T)
print(f"Covariance matrix:\n{cov_matrix}")

# Step 3: Find eigenvectors (principal components)
eigenvalues, eigenvectors = np.linalg.eigh(cov_matrix)  # eigh for symmetric!

# Sort by eigenvalue (largest first)
idx = eigenvalues.argsort()[::-1]
eigenvalues = eigenvalues[idx]
eigenvectors = eigenvectors[:, idx]

print(f"Variance explained: {eigenvalues / eigenvalues.sum() * 100}")
# e.g., [85%, 15%] → first component captures 85% of variance

# Step 4: Project data onto first principal component
pc1 = eigenvectors[:, 0]  # Direction of maximum variance
projected = data_centered @ pc1  # 2D → 1D
print(f"Original shape: {data_centered.shape}, Projected shape: {projected.shape}")

# === EIGENVALUES AND MATRIX PROPERTIES ===
A = np.array([[3, 1], [0, 2]])
eigenvalues = np.linalg.eigvals(A)
print(f"Eigenvalues: {eigenvalues}")

# Important relationships:
print(f"Trace (sum of diag): {np.trace(A)}, Sum of eigenvalues: {sum(eigenvalues)}")
print(f"Determinant: {np.linalg.det(A):.1f}, Product of eigenvalues: {np.prod(eigenvalues):.1f}")
# trace(A) = sum of eigenvalues
# det(A) = product of eigenvalues

# === POWER ITERATION (How Google PageRank works) ===
def power_iteration(A, num_iterations=100):
    """Find the dominant eigenvector (largest eigenvalue)"""
    n = A.shape[0]
    v = np.random.rand(n)  # Random starting vector
    
    for _ in range(num_iterations):
        Av = A @ v           # Apply matrix
        v = Av / np.linalg.norm(Av)  # Normalize
    
    eigenvalue = v @ A @ v   # Rayleigh quotient
    return eigenvalue, v

A = np.array([[4, 1], [2, 3]])
lam, v = power_iteration(A)
print(f"Dominant eigenvalue: {lam:.4f}")  # ≈ 5.0
print(f"Dominant eigenvector: {v}")

# === SPECTRAL DECOMPOSITION ===
# Any symmetric matrix A = Q Λ Q^T
# where Q = eigenvectors, Λ = diagonal of eigenvalues
A_sym = np.array([[4, 2], [2, 3]])
eigenvalues, Q = np.linalg.eigh(A_sym)
Lambda = np.diag(eigenvalues)

# Reconstruct
A_reconstructed = Q @ Lambda @ Q.T
print(f"Reconstruction matches? {np.allclose(A_sym, A_reconstructed)}")  # True
```

> **Pro Tip:** Always use `np.linalg.eigh()` for symmetric matrices (covariance, kernel matrices). It's faster and guarantees real eigenvalues. Use `np.linalg.eig()` for general matrices.

### Common Mistakes
1. **Eigenvectors are COLUMNS of the output matrix**, not rows (in NumPy)
2. **Eigenvectors are only defined up to a sign** — `v` and `-v` are both valid eigenvectors
3. **Not all matrices are diagonalizable** — Defective matrices exist (but are rare in practice)
4. **Using `eig` instead of `eigh` for symmetric matrices** — `eigh` is faster and more numerically stable

### Interview Questions
1. **What's the geometric interpretation of eigenvalues/eigenvectors?**
   → Eigenvectors are invariant directions; eigenvalues are scaling factors along those directions.

2. **How are eigenvalues related to PCA?**
   → The eigenvectors of the covariance matrix are the principal components. Eigenvalues tell you the variance explained by each component.

3. **If all eigenvalues of a matrix are positive, what does it mean?**
   → The matrix is positive definite (if also symmetric). This guarantees a unique minimum in optimization.

4. **What's the relationship between SVD and eigendecomposition?**
   → SVD generalizes eigendecomposition to non-square matrices. For symmetric A: singular values = |eigenvalues|.

---

## 7. Singular Value Decomposition (SVD)

### What It Is
SVD decomposes ANY matrix (even non-square) into three matrices:

$$A = U \Sigma V^T$$

Where:
- $U$ (m×m) = left singular vectors (orthogonal) — "row space patterns"
- $\Sigma$ (m×n) = diagonal matrix of singular values — "importance weights"
- $V^T$ (n×n) = right singular vectors (orthogonal) — "column space patterns"

### Why It Matters
- **Dimensionality reduction** — Truncated SVD / LSA for text
- **Recommendation systems** — Netflix Prize used SVD
- **Image compression** — Keep top-k singular values
- **Pseudo-inverse** — Solving underdetermined/overdetermined systems
- **Noise reduction** — Small singular values = noise
- **Low-rank approximation** — Eckart-Young theorem guarantees SVD gives the best low-rank approximation

### How It Works

```
Original Matrix A (m×n):

A = U    ×    Σ    ×    V^T

[. . .]   [* . .]   [σ₁ 0  0]   [- v₁ -]
[. . .] = [. * .]   [0  σ₂ 0] × [- v₂ -]
[. . .]   [. . *]   [0  0  σ₃]   [- v₃ -]
[. . .]   [. . .]

m×n       m×m        m×n          n×n

σ₁ ≥ σ₂ ≥ σ₃ ≥ 0 (always non-negative, sorted)
```

**Analogy:** Think of SVD as decomposing a complex recipe into:
- What ingredients to use (V)
- How much of each ingredient (Σ)
- What dishes those ingredients make (U)

### Code Examples

```python
import numpy as np

# === BASIC SVD ===
A = np.array([[1, 2, 3], [4, 5, 6], [7, 8, 9], [10, 11, 12]])
print(f"Original shape: {A.shape}")  # (4, 3)

U, sigma, Vt = np.linalg.svd(A, full_matrices=True)
print(f"U shape: {U.shape}")        # (4, 4)
print(f"Sigma: {sigma}")             # [25.4, 1.3, 0.0] (decreasing)
print(f"V^T shape: {Vt.shape}")     # (3, 3)

# Reconstruct (need to create full Sigma matrix)
Sigma_full = np.zeros_like(A, dtype=float)
np.fill_diagonal(Sigma_full, sigma)
A_reconstructed = U @ Sigma_full @ Vt
print(f"Reconstruction error: {np.linalg.norm(A - A_reconstructed):.2e}")  # ≈ 0

# === IMAGE COMPRESSION WITH SVD ===
# Simulate a grayscale image (or use a real one)
np.random.seed(42)
image = np.random.rand(100, 100)  # 100×100 "image"

U, sigma, Vt = np.linalg.svd(image, full_matrices=False)

# Keep only top-k singular values (compression!)
for k in [5, 10, 20, 50]:
    # Truncated SVD: only use first k components
    compressed = U[:, :k] @ np.diag(sigma[:k]) @ Vt[:k, :]
    
    # Compression ratio
    original_size = image.shape[0] * image.shape[1]
    compressed_size = k * (image.shape[0] + image.shape[1] + 1)
    ratio = original_size / compressed_size
    error = np.linalg.norm(image - compressed) / np.linalg.norm(image)
    
    print(f"k={k:2d}: compression={ratio:.1f}x, relative error={error:.4f}")
# k= 5: compression=10.0x, relative error=0.9129
# k=10: compression= 5.0x, relative error=0.8792
# k=20: compression= 2.5x, relative error=0.8399
# k=50: compression= 1.0x, relative error=0.7180

# === RECOMMENDATION SYSTEM (Collaborative Filtering) ===
# User-Item rating matrix (0 = not rated)
ratings = np.array([
    [5, 4, 0, 0, 1],  # User 1 likes action movies (cols 0,1)
    [4, 5, 0, 1, 0],  # User 2 likes action movies
    [0, 0, 5, 4, 0],  # User 3 likes romance (cols 2,3)
    [0, 1, 4, 5, 0],  # User 4 likes romance
    [1, 0, 0, 0, 5],  # User 5 likes documentary (col 4)
])

U, sigma, Vt = np.linalg.svd(ratings, full_matrices=False)

# Use top-2 latent factors
k = 2
U_k = U[:, :k]
sigma_k = np.diag(sigma[:k])
Vt_k = Vt[:k, :]

# Predict missing ratings
predicted_ratings = U_k @ sigma_k @ Vt_k
print(f"Predicted ratings:\n{np.round(predicted_ratings, 1)}")
# The 0s (unrated) now have predicted scores!

# === TRUNCATED SVD FOR TEXT (Latent Semantic Analysis) ===
from sklearn.decomposition import TruncatedSVD
from sklearn.feature_extraction.text import TfidfVectorizer

documents = [
    "machine learning algorithms",
    "deep learning neural networks",
    "supervised learning classification",
    "natural language processing text",
    "computer vision image recognition",
]

# Create TF-IDF matrix
vectorizer = TfidfVectorizer()
tfidf_matrix = vectorizer.fit_transform(documents)
print(f"TF-IDF shape: {tfidf_matrix.shape}")  # (5, n_terms)

# Reduce to 2 dimensions using SVD
svd = TruncatedSVD(n_components=2)
reduced = svd.fit_transform(tfidf_matrix)
print(f"Reduced shape: {reduced.shape}")  # (5, 2)
print(f"Variance explained: {svd.explained_variance_ratio_.sum():.2%}")

# === PSEUDO-INVERSE (Moore-Penrose) ===
# When A is not square or not invertible
A = np.array([[1, 2], [3, 4], [5, 6]])  # 3×2 (overdetermined)
A_pinv = np.linalg.pinv(A)  # Uses SVD internally
print(f"A shape: {A.shape}, Pseudo-inverse shape: {A_pinv.shape}")
# A_pinv @ A ≈ I (for full column rank)
```

> **Pro Tip:** For large sparse matrices, use `scipy.sparse.linalg.svds()` or `sklearn.decomposition.TruncatedSVD` — they compute only the top-k singular values without computing the full decomposition.

### Common Mistakes
1. **Confusing `sigma` vector with `Sigma` matrix** — `np.linalg.svd` returns a 1D array; you need to construct the diagonal matrix
2. **Using full SVD on large matrices** — Use truncated SVD for dimensionality reduction
3. **Not handling missing values** — Standard SVD can't handle missing data; use iterative methods or matrix factorization

---

## 8. Matrix Decompositions

### What It Is
Breaking a matrix into simpler component matrices. Different decompositions are useful for different problems.

### Why It Matters

| Decomposition | Form | When to Use |
|--------------|------|-------------|
| LU | A = LU | Solving systems of equations |
| QR | A = QR | Least squares, numerical stability |
| Cholesky | A = LL^T | Positive definite systems (fast!) |
| Eigen | A = QΛQ^(-1) | PCA, spectral methods |
| SVD | A = UΣV^T | Everything else |

### Code Examples

```python
import numpy as np
from scipy import linalg

# === LU DECOMPOSITION ===
# A = P @ L @ U (P = permutation, L = lower triangular, U = upper triangular)
A = np.array([[2, 1, 1], [4, 3, 3], [8, 7, 9]])
P, L, U = linalg.lu(A)
print(f"P @ L @ U = A? {np.allclose(P @ L @ U, A)}")  # True

# Use case: solve Ax = b for multiple different b vectors efficiently
# Factor once, solve many times

# === QR DECOMPOSITION ===
# A = Q @ R (Q = orthogonal, R = upper triangular)
A = np.array([[1, 2], [3, 4], [5, 6]])
Q, R = np.linalg.qr(A)
print(f"Q shape: {Q.shape}, R shape: {R.shape}")
print(f"Q is orthogonal? {np.allclose(Q.T @ Q, np.eye(Q.shape[1]))}")
print(f"Q @ R = A? {np.allclose(Q @ R, A)}")

# QR is used for numerically stable least squares:
# Ax = b → QRx = b → Rx = Q^T b (easy to solve!)

# === CHOLESKY DECOMPOSITION ===
# A = L @ L^T (only for positive definite matrices)
# 2x faster than LU!
A = np.array([[4, 2], [2, 3]])  # Symmetric positive definite
L = np.linalg.cholesky(A)
print(f"L:\n{L}")
print(f"L @ L^T = A? {np.allclose(L @ L.T, A)}")

# Use case: sampling from multivariate Gaussian
# To sample from N(μ, Σ): x = μ + L @ z, where z ~ N(0, I)
mean = np.array([1, 2])
cov = np.array([[4, 2], [2, 3]])
L = np.linalg.cholesky(cov)

z = np.random.randn(2, 1000)  # Standard normal samples
samples = mean.reshape(-1, 1) + L @ z  # Transform to desired distribution
print(f"Sample mean: {samples.mean(axis=1)}")   # ≈ [1, 2]
print(f"Sample cov:\n{np.cov(samples)}")         # ≈ [[4,2],[2,3]]
```

---

## 9. Norms and Distances

### What It Is
Ways to measure the "size" of a vector or matrix, and the "distance" between two points.

### Why It Matters
- **L1 norm** → Lasso regularization (sparse solutions)
- **L2 norm** → Ridge regularization (small weights)
- **Frobenius norm** → Matrix distance in neural networks
- **Distance metrics** → k-NN, clustering, similarity search

### How It Works

$$\|x\|_p = \left(\sum_i |x_i|^p\right)^{1/p}$$

| Norm | Formula | Effect | ML Usage |
|------|---------|--------|----------|
| L0 | Count of non-zeros | Sparsity | Feature selection (NP-hard) |
| L1 (Manhattan) | $\sum|x_i|$ | Diamond-shaped constraint | Lasso (sparse weights) |
| L2 (Euclidean) | $\sqrt{\sum x_i^2}$ | Circle-shaped constraint | Ridge (small weights) |
| L∞ (Max) | $\max|x_i|$ | Square-shaped constraint | Adversarial robustness |

```
        L1 ball         L2 ball         L∞ ball
        
           *              ***             ****
          * *            *   *            *  *
         *   *          *     *           *  *
          * *            *   *            *  *
           *              ***             ****
        (diamond)       (circle)        (square)
```

### Code Examples

```python
import numpy as np

# === VECTOR NORMS ===
x = np.array([3, -4, 5])

l1_norm = np.linalg.norm(x, ord=1)     # |3| + |-4| + |5| = 12
l2_norm = np.linalg.norm(x, ord=2)     # sqrt(9 + 16 + 25) = sqrt(50)
linf_norm = np.linalg.norm(x, ord=np.inf)  # max(|3|, |-4|, |5|) = 5

print(f"L1: {l1_norm}")     # 12.0
print(f"L2: {l2_norm:.4f}") # 7.0711
print(f"L∞: {linf_norm}")   # 5.0

# === WHY L1 GIVES SPARSITY ===
# Optimization: minimize loss + λ * ||w||
# L1 penalty has "sharp corners" at axes → solutions land ON axes (sparse!)
# L2 penalty is smooth → solutions shrink toward zero but rarely hit exactly zero

# === DISTANCE METRICS ===
a = np.array([1, 2, 3])
b = np.array([4, 5, 6])

# Euclidean distance
euclidean = np.linalg.norm(a - b)
print(f"Euclidean: {euclidean:.4f}")  # 5.1962

# Manhattan distance  
manhattan = np.sum(np.abs(a - b))
print(f"Manhattan: {manhattan}")  # 9

# Cosine distance (1 - cosine_similarity)
cos_sim = np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
cos_dist = 1 - cos_sim
print(f"Cosine distance: {cos_dist:.4f}")  # 0.0254

# === MATRIX NORMS ===
A = np.array([[1, 2], [3, 4]])

frobenius = np.linalg.norm(A, 'fro')    # sqrt(1+4+9+16) = sqrt(30)
spectral = np.linalg.norm(A, 2)          # Largest singular value
nuclear = np.sum(np.linalg.svd(A, compute_uv=False))  # Sum of singular values

print(f"Frobenius: {frobenius:.4f}")
print(f"Spectral (largest σ): {spectral:.4f}")
print(f"Nuclear (sum of σ): {nuclear:.4f}")

# === REGULARIZATION IN PRACTICE ===
# L1 (Lasso) → feature selection (sparse model)
# L2 (Ridge) → weight decay (small weights)
# Elastic Net → both L1 + L2

from sklearn.linear_model import Lasso, Ridge, ElasticNet

# Generate data with redundant features
np.random.seed(42)
X = np.random.randn(100, 20)   # 20 features
true_weights = np.zeros(20)
true_weights[:5] = [3, -2, 5, -1, 4]  # Only 5 features matter
y = X @ true_weights + np.random.randn(100) * 0.5

# L1: Lasso finds the sparse solution
lasso = Lasso(alpha=0.1).fit(X, y)
print(f"Lasso non-zero weights: {np.sum(lasso.coef_ != 0)}")  # ~5-7

# L2: Ridge shrinks all weights but keeps them all
ridge = Ridge(alpha=0.1).fit(X, y)
print(f"Ridge non-zero weights: {np.sum(ridge.coef_ != 0)}")  # 20 (all)
```

### Common Mistakes
1. **Using Euclidean distance on high-dimensional data** — It becomes meaningless ("curse of dimensionality"). Use cosine similarity instead.
2. **Forgetting to normalize before computing distances** — Features on different scales will dominate.
3. **Confusing norm with distance** — Norm = size of one vector; distance = size of the difference between two vectors.

---

## 10. Applications in ML

### Quick Reference: Where Each Concept Appears

| Concept | ML Application |
|---------|---------------|
| Matrix multiply | Neural network forward pass |
| Transpose | Computing gradients, reshaping data |
| Inverse/Pseudo-inverse | Normal equation (linear regression) |
| Eigendecomposition | PCA, spectral clustering, PageRank |
| SVD | Recommender systems, LSA, image compression |
| Dot product | Attention mechanism, similarity |
| Cosine similarity | Document similarity, embedding comparison |
| L1 norm | Lasso regularization, sparse features |
| L2 norm | Ridge regularization, weight decay |
| Positive definite | Kernel matrices, Gaussian processes |
| Cholesky | Sampling multivariate Gaussians |
| Orthogonal matrices | Weight initialization (prevents exploding/vanishing gradients) |
| Matrix rank | Feature redundancy, model capacity |
| Determinant | Volume change, invertibility check |

### Code: Full Pipeline Example

```python
import numpy as np
from sklearn.preprocessing import StandardScaler

# === END-TO-END: PCA for Dimensionality Reduction ===
np.random.seed(42)

# Step 1: Generate high-dimensional data
n_samples, n_features = 200, 50
X = np.random.randn(n_samples, n_features)

# Add correlations (make PCA useful)
X[:, 10:20] = X[:, :10] + np.random.randn(n_samples, 10) * 0.1
X[:, 20:30] = X[:, :10] * 2 + np.random.randn(n_samples, 10) * 0.1

# Step 2: Standardize (zero mean, unit variance)
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# Step 3: Compute covariance matrix
cov_matrix = np.cov(X_scaled.T)  # (50×50)

# Step 4: Eigendecomposition
eigenvalues, eigenvectors = np.linalg.eigh(cov_matrix)

# Sort descending
idx = eigenvalues.argsort()[::-1]
eigenvalues = eigenvalues[idx]
eigenvectors = eigenvectors[:, idx]

# Step 5: Choose number of components (keep 95% variance)
cumulative_variance = np.cumsum(eigenvalues) / eigenvalues.sum()
n_components = np.searchsorted(cumulative_variance, 0.95) + 1
print(f"Components needed for 95% variance: {n_components} (out of {n_features})")

# Step 6: Project data
W = eigenvectors[:, :n_components]  # Top-k eigenvectors
X_reduced = X_scaled @ W            # Project: (200×50) @ (50×k) = (200×k)
print(f"Reduced shape: {X_reduced.shape}")

# This is EXACTLY what sklearn.decomposition.PCA does internally!
```

---

## Interview Questions

### Conceptual
1. **Why is matrix multiplication not commutative? Give an ML example.**
   → Matrix dimensions may not even allow reversal (3×2 @ 2×5 works, but 2×5 @ 3×2 doesn't). In ML: applying layer1 then layer2 gives different results than layer2 then layer1.

2. **Explain the curse of dimensionality using linear algebra.**
   → In high dimensions, all points become roughly equidistant (L2 norm differences converge). Volumes grow exponentially, so data becomes sparse.

3. **What happens to eigenvalues when you add λI to a matrix?**
   → All eigenvalues increase by λ. This is why ridge regression (adding λI to X^TX) makes the matrix invertible.

4. **Why does SVD always exist but eigendecomposition doesn't?**
   → SVD works for ANY matrix (even non-square). Eigendecomposition requires a square matrix and may not exist for defective matrices.

5. **How is the rank of a weight matrix related to model expressiveness?**
   → A rank-r matrix can only represent r independent patterns. Low-rank = limited capacity. LoRA exploits this: fine-tuning only needs a low-rank update.

### Coding
6. **Implement PCA without sklearn.**
   → (See Section 6 code above)

7. **How would you check if a matrix is positive definite?**
   → Check all eigenvalues > 0, or try Cholesky decomposition (fails if not PD).

8. **Implement cosine similarity between two vectors.**
   → `dot(a,b) / (norm(a) * norm(b))`

---

## Quick Reference

| Operation | NumPy Code | Time Complexity |
|-----------|-----------|----------------|
| Dot product | `a @ b` or `np.dot(a, b)` | O(n) |
| Matrix multiply | `A @ B` | O(n³) |
| Transpose | `A.T` | O(1) (view) |
| Inverse | `np.linalg.inv(A)` | O(n³) |
| Determinant | `np.linalg.det(A)` | O(n³) |
| Eigenvalues | `np.linalg.eigh(A)` | O(n³) |
| SVD | `np.linalg.svd(A)` | O(min(mn², m²n)) |
| Solve Ax=b | `np.linalg.solve(A, b)` | O(n³) |
| Pseudo-inverse | `np.linalg.pinv(A)` | O(mn²) |
| Norm | `np.linalg.norm(x, ord=p)` | O(n) |
| Rank | `np.linalg.matrix_rank(A)` | O(mn²) |

> **Golden Rule:** Never invert a matrix if you can solve a linear system instead. `solve(A, b)` is always preferred over `inv(A) @ b`.
