# Chapter 01: Neural Network Fundamentals

> **From a single neuron to multi-layer networks — the building blocks of all deep learning.**

---

## Table of Contents
1. [The Biological Inspiration](#1-the-biological-inspiration)
2. [The Perceptron](#2-the-perceptron)
3. [Activation Functions](#3-activation-functions)
4. [Multi-Layer Perceptron (MLP)](#4-multi-layer-perceptron-mlp)
5. [Forward Propagation](#5-forward-propagation)
6. [Loss Functions (Intro)](#6-loss-functions-intro)
7. [Backpropagation](#7-backpropagation)
8. [Gradient Descent](#8-gradient-descent)
9. [Universal Approximation Theorem](#9-universal-approximation-theorem)
10. [Building Your First Neural Network](#10-building-your-first-neural-network)
11. [Common Mistakes](#11-common-mistakes)
12. [Interview Questions](#12-interview-questions)
13. [Quick Reference](#13-quick-reference)

---

## 1. The Biological Inspiration

### What It Is
A neural network is a computing system loosely inspired by the brain's network of neurons. Just like how your brain has billions of neurons connected by synapses, an artificial neural network has "nodes" connected by "weights."

### The Analogy
Think of it like a **voting committee**:
- Each committee member (neuron) looks at the evidence (inputs)
- Each member has different influence (weights)
- They vote (activate or not)
- The final decision comes from combining all votes

### Biological vs Artificial Neuron

```
BIOLOGICAL NEURON                    ARTIFICIAL NEURON
                                     
  Dendrites ──┐                      Inputs (x₁, x₂, ...) ──┐
  Dendrites ──┤                      Inputs ──────────────────┤
  Dendrites ──┼── Cell Body ── Axon  Inputs ──────────────────┼── Σ(wᵢxᵢ + b) ── f(z) ── Output
  Dendrites ──┤   (Soma)             Inputs ──────────────────┤
  Dendrites ──┘                      Inputs ──────────────────┘
                                     
  Synapse strength = Weight          Weight (wᵢ) = Connection strength
  Threshold = Bias                   Bias (b) = Activation threshold
  Firing = Activation                Activation function = f(z)
```

> **Important:** Artificial neural networks are *inspired* by biology but NOT faithful simulations. Real neurons are far more complex (timing, neurotransmitters, dendritic computation). Don't over-extend the analogy.

---

## 2. The Perceptron

### What It Is
The perceptron is the simplest neural network — a single neuron that makes binary decisions. It's the "Hello World" of deep learning. Invented by Frank Rosenblatt in 1958.

### Why It Matters
- It's the foundation of ALL neural networks
- Understanding it deeply makes everything else click
- It shows both the power and limitations of linear models

### How It Works

**Mathematical Formula:**

$$y = f\left(\sum_{i=1}^{n} w_i x_i + b\right) = f(\mathbf{w}^T \mathbf{x} + b)$$

Where:
- $x_i$ = input features
- $w_i$ = weights (importance of each feature)
- $b$ = bias (threshold adjuster)
- $f$ = activation function (step function for perceptron)
- $y$ = output

**Step Function (Original Perceptron):**

$$f(z) = \begin{cases} 1 & \text{if } z \geq 0 \\ 0 & \text{if } z < 0 \end{cases}$$

### Geometric Intuition

The perceptron draws a **straight line** (hyperplane) to separate two classes:

```
    x₂
    │         Class 1 (y=1)
    │       ○  ○
    │     ○   ○  ○
    │   ─────────────── Decision Boundary: w₁x₁ + w₂x₂ + b = 0
    │     ●  ●
    │   ●   ●  ●
    │         Class 0 (y=0)
    └──────────────────── x₁
```

### The XOR Problem — Perceptron's Fatal Flaw

The perceptron CANNOT solve the XOR problem because XOR is not linearly separable:

```
    x₂
    │
  1 │  ●(0,1)=1      ○(1,1)=0
    │
  0 │  ○(0,0)=0      ●(1,0)=1
    │
    └──────────────────── x₁
       0              1

No single straight line can separate ● from ○!
```

This limitation led to the "AI Winter" of the 1970s. The solution? **Multi-layer networks**.

### Code Example: Perceptron from Scratch

```python
import numpy as np

class Perceptron:
    """Single-layer perceptron for binary classification."""
    
    def __init__(self, n_features, learning_rate=0.01):
        # Initialize weights to small random values
        self.weights = np.zeros(n_features)
        self.bias = 0.0
        self.lr = learning_rate
    
    def step_function(self, z):
        """Binary step activation: returns 1 if z >= 0, else 0."""
        return np.where(z >= 0, 1, 0)
    
    def predict(self, X):
        """Forward pass: compute weighted sum and activate."""
        # z = w·x + b (dot product + bias)
        z = np.dot(X, self.weights) + self.bias
        return self.step_function(z)
    
    def fit(self, X, y, n_epochs=100):
        """Train using perceptron learning rule."""
        errors_per_epoch = []
        
        for epoch in range(n_epochs):
            errors = 0
            for xi, yi in zip(X, y):
                # Predict
                y_pred = self.predict(xi.reshape(1, -1))[0]
                
                # Calculate error
                error = yi - y_pred  # Will be -1, 0, or 1
                
                # Update rule: w = w + lr * error * x
                # Only updates when prediction is wrong (error != 0)
                self.weights += self.lr * error * xi
                self.bias += self.lr * error
                
                if error != 0:
                    errors += 1
            
            errors_per_epoch.append(errors)
            
            # Convergence check
            if errors == 0:
                print(f"Converged at epoch {epoch + 1}")
                break
        
        return errors_per_epoch

# === Demo: AND gate ===
X = np.array([[0, 0], [0, 1], [1, 0], [1, 1]])
y_and = np.array([0, 0, 0, 1])  # AND truth table

perceptron = Perceptron(n_features=2, learning_rate=0.1)
perceptron.fit(X, y_and)

print("AND Gate Predictions:")
for xi in X:
    print(f"  {xi} -> {perceptron.predict(xi.reshape(1, -1))[0]}")
# Output:
# AND Gate Predictions:
#   [0 0] -> 0
#   [0 1] -> 0
#   [1 0] -> 0
#   [1 1] -> 1
```

### Perceptron Learning Rule — Why It Works

The update rule `w = w + lr * error * x` is elegant:
- If prediction is **correct**: error = 0, no update
- If prediction is **too low** (predicted 0, actual 1): error = +1, weights increase toward input
- If prediction is **too high** (predicted 1, actual 0): error = -1, weights decrease away from input

> **Perceptron Convergence Theorem:** If data is linearly separable, the perceptron is guaranteed to converge in a finite number of steps. If NOT linearly separable, it will oscillate forever.

---

## 3. Activation Functions

### What They Are
Activation functions introduce **non-linearity** into neural networks. Without them, stacking layers would be useless — you'd just get another linear function.

### Why They Matter
- Without activation functions: `f(g(x)) = W₂(W₁x) = (W₂W₁)x = Wx` — just linear!
- They allow networks to learn complex, non-linear patterns
- Different activations have different properties that affect training

### The Complete Zoo of Activation Functions

#### Sigmoid (Logistic)

$$\sigma(z) = \frac{1}{1 + e^{-z}}$$

```
Output:  1 |          ___________
           |        /
       0.5 |------/----------------
           |    /
         0 |___/
           +----|----|----|----|--->  z
               -4   -2    2    4
```

**Properties:**
- Range: (0, 1)
- Smooth, differentiable everywhere
- Derivative: $\sigma'(z) = \sigma(z)(1 - \sigma(z))$
- Max derivative = 0.25 (at z=0)

**When to use:** Output layer for binary classification (probability output)

**Problems:**
- **Vanishing gradient:** derivative max is 0.25, so gradients shrink exponentially in deep networks
- **Not zero-centered:** outputs always positive → zig-zag gradient updates
- **Expensive:** `exp()` is computationally costly

#### Tanh (Hyperbolic Tangent)

$$\tanh(z) = \frac{e^z - e^{-z}}{e^z + e^{-z}} = 2\sigma(2z) - 1$$

```
Output:  1 |          ___________
           |        /
         0 |------/----------------
           |    /
        -1 |___/
           +----|----|----|----|--->  z
               -4   -2    2    4
```

**Properties:**
- Range: (-1, 1)
- Zero-centered (better than sigmoid!)
- Derivative: $\tanh'(z) = 1 - \tanh^2(z)$
- Max derivative = 1.0 (at z=0)

**When to use:** Hidden layers in RNNs, when you need zero-centered output

**Problems:** Still suffers from vanishing gradient (saturates at extremes)

#### ReLU (Rectified Linear Unit)

$$\text{ReLU}(z) = \max(0, z)$$

```
Output:    |        /
           |       /
           |      /
           |     /
         0 |____/___________________
           +----|----|----|----|--->  z
               -4   -2    2    4
```

**Properties:**
- Range: [0, ∞)
- Derivative: 1 for z > 0, 0 for z < 0, undefined at z = 0 (use 0 in practice)
- Computationally cheap (just a threshold)
- Doesn't saturate for positive values

**When to use:** Default choice for hidden layers in most networks

**Problems:**
- **Dying ReLU:** If a neuron gets a large negative input, it outputs 0 and gradient is 0. It can never recover — it's "dead."
- Not zero-centered

#### Leaky ReLU

$$\text{LeakyReLU}(z) = \begin{cases} z & \text{if } z > 0 \\ \alpha z & \text{if } z \leq 0 \end{cases}$$

Where $\alpha$ is typically 0.01.

```
Output:    |        /
           |       /
           |      /
           |     /
         0 |_ _/___________________
           |_/   (slight negative slope)
           +----|----|----|----|--->  z
```

**When to use:** When dying ReLU is a problem. Default $\alpha = 0.01$.

#### Parametric ReLU (PReLU)
Same as Leaky ReLU but $\alpha$ is a **learnable parameter**.

#### ELU (Exponential Linear Unit)

$$\text{ELU}(z) = \begin{cases} z & \text{if } z > 0 \\ \alpha(e^z - 1) & \text{if } z \leq 0 \end{cases}$$

**Advantage:** Smooth, pushes mean activations toward zero, more robust to noise.

#### GELU (Gaussian Error Linear Unit)

$$\text{GELU}(z) = z \cdot \Phi(z) \approx 0.5z\left(1 + \tanh\left[\sqrt{\frac{2}{\pi}}(z + 0.044715z^3)\right]\right)$$

**When to use:** Transformers (BERT, GPT use GELU). State-of-the-art for NLP models.

#### Swish / SiLU

$$\text{Swish}(z) = z \cdot \sigma(z) = \frac{z}{1 + e^{-z}}$$

**When to use:** Modern architectures (EfficientNet). Often outperforms ReLU.

#### Softmax (Multi-class Output)

$$\text{Softmax}(z_i) = \frac{e^{z_i}}{\sum_{j=1}^{K} e^{z_j}}$$

**When to use:** Output layer for multi-class classification. Converts logits to probabilities that sum to 1.

### Comparison Table

| Function | Range | Zero-Centered | Gradient | Use Case |
|----------|-------|---------------|----------|----------|
| Sigmoid | (0,1) | No | Vanishes | Binary output |
| Tanh | (-1,1) | Yes | Vanishes | RNN hidden |
| ReLU | [0,∞) | No | Dies | Default hidden |
| Leaky ReLU | (-∞,∞) | No | Alive | When ReLU dies |
| ELU | (-α,∞) | ~Yes | Alive | Noise robustness |
| GELU | (-0.17,∞) | ~Yes | Smooth | Transformers |
| Swish | (-0.28,∞) | ~Yes | Smooth | Modern CNNs |
| Softmax | (0,1) | No | N/A | Multi-class output |

### Code Example: Activation Functions Visualized

```python
import numpy as np
import matplotlib.pyplot as plt

# Define activation functions
def sigmoid(z):
    return 1 / (1 + np.exp(-z))

def tanh(z):
    return np.tanh(z)

def relu(z):
    return np.maximum(0, z)

def leaky_relu(z, alpha=0.01):
    return np.where(z > 0, z, alpha * z)

def elu(z, alpha=1.0):
    return np.where(z > 0, z, alpha * (np.exp(z) - 1))

def swish(z):
    return z * sigmoid(z)

def gelu(z):
    return 0.5 * z * (1 + np.tanh(np.sqrt(2/np.pi) * (z + 0.044715 * z**3)))

# Generate input values
z = np.linspace(-5, 5, 1000)

# Plot all activations
fig, axes = plt.subplots(2, 4, figsize=(16, 8))
activations = [
    (sigmoid, "Sigmoid"), (tanh, "Tanh"),
    (relu, "ReLU"), (leaky_relu, "Leaky ReLU"),
    (elu, "ELU"), (swish, "Swish"),
    (gelu, "GELU"), (None, None)
]

for ax, (func, name) in zip(axes.flat, activations):
    if func is None:
        ax.axis('off')
        continue
    ax.plot(z, func(z), 'b-', linewidth=2)
    ax.axhline(y=0, color='k', linewidth=0.5)
    ax.axvline(x=0, color='k', linewidth=0.5)
    ax.set_title(name, fontsize=14)
    ax.set_xlim(-5, 5)
    ax.grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('activation_functions.png', dpi=100)
plt.show()
```

### Pro Tips

> **Rule of thumb for choosing activation functions:**
> 1. **Hidden layers:** Start with ReLU. If dying neurons → try Leaky ReLU or ELU.
> 2. **Output layer (binary):** Sigmoid
> 3. **Output layer (multi-class):** Softmax
> 4. **Output layer (regression):** Linear (no activation)
> 5. **Transformers/NLP:** GELU
> 6. **Modern CNNs:** Swish or GELU

---

## 4. Multi-Layer Perceptron (MLP)

### What It Is
An MLP is a neural network with one or more hidden layers between input and output. It's also called a "feedforward neural network" or "fully-connected network" or "dense network."

### Why It Matters
- MLPs can approximate ANY continuous function (Universal Approximation Theorem)
- They solve the XOR problem and other non-linear problems
- They're the backbone architecture that all other architectures build upon

### Architecture

```
INPUT LAYER      HIDDEN LAYER 1    HIDDEN LAYER 2    OUTPUT LAYER
(Features)       (Learned repr.)   (Higher-level)    (Prediction)

  x₁ ─────────── h₁⁽¹⁾ ────────── h₁⁽²⁾ ──────────── ŷ₁
      ╲  ╱  ╲  ╱      ╲  ╱  ╲  ╱       ╲  ╱  ╲  ╱
  x₂ ──╳────╳── h₂⁽¹⁾ ──╳────╳── h₂⁽²⁾ ──╳────╳──── ŷ₂
      ╱  ╲  ╱  ╲      ╱  ╲  ╱  ╲       ╱  ╲  ╱  ╲
  x₃ ─────────── h₃⁽¹⁾ ────────── h₃⁽²⁾ ──────────── ŷ₃
      ╲  ╱  ╲  ╱      ╲  ╱  ╲  ╱
  x₄ ─────────── h₄⁽¹⁾ ────────── (fewer neurons)

  [4 neurons]    [4 neurons]       [3 neurons]       [3 neurons]

  Every neuron in layer L is connected to EVERY neuron in layer L+1
  → "Fully Connected" or "Dense"
```

### Key Terminology

| Term | Meaning |
|------|---------|
| **Layer** | A collection of neurons at the same depth |
| **Input Layer** | Receives raw features (NOT counted as a "layer" in depth) |
| **Hidden Layer** | Any layer between input and output |
| **Output Layer** | Produces final prediction |
| **Depth** | Number of hidden + output layers |
| **Width** | Number of neurons in a layer |
| **Parameters** | All weights + biases in the network |
| **Architecture** | The specific configuration of layers/neurons |

### Parameter Count Calculation

For a network with layers of sizes: [input=4, hidden1=8, hidden2=6, output=3]:

$$\text{Parameters} = \sum_{l=1}^{L} (n_{l-1} \times n_l + n_l)$$

- Layer 1: 4×8 + 8 = 40 (32 weights + 8 biases)
- Layer 2: 8×6 + 6 = 54 (48 weights + 6 biases)
- Layer 3: 6×3 + 3 = 21 (18 weights + 3 biases)
- **Total: 115 parameters**

```python
# Quick parameter counting
def count_params(layer_sizes):
    """Count total trainable parameters in a dense network."""
    total = 0
    for i in range(1, len(layer_sizes)):
        weights = layer_sizes[i-1] * layer_sizes[i]
        biases = layer_sizes[i]
        total += weights + biases
        print(f"Layer {i}: {layer_sizes[i-1]}x{layer_sizes[i]} + {layer_sizes[i]} = {weights + biases}")
    print(f"Total parameters: {total}")
    return total

count_params([784, 256, 128, 10])  # MNIST example
# Layer 1: 784x256 + 256 = 200960
# Layer 2: 256x128 + 128 = 32896
# Layer 3: 128x10 + 10 = 1290
# Total parameters: 235146
```

### Code Example: MLP from Scratch (NumPy)

```python
import numpy as np

class MLP:
    """Multi-Layer Perceptron from scratch using NumPy."""
    
    def __init__(self, layer_sizes, activation='relu', seed=42):
        """
        Args:
            layer_sizes: List of ints, e.g., [784, 128, 64, 10]
            activation: 'relu' or 'tanh' for hidden layers
        """
        np.random.seed(seed)
        self.n_layers = len(layer_sizes) - 1
        self.activation_name = activation
        
        # Initialize weights using He initialization (for ReLU)
        # or Xavier initialization (for tanh/sigmoid)
        self.weights = []
        self.biases = []
        
        for i in range(self.n_layers):
            n_in, n_out = layer_sizes[i], layer_sizes[i+1]
            
            if activation == 'relu':
                # He initialization: scale by sqrt(2/n_in)
                w = np.random.randn(n_in, n_out) * np.sqrt(2.0 / n_in)
            else:
                # Xavier initialization: scale by sqrt(1/n_in)
                w = np.random.randn(n_in, n_out) * np.sqrt(1.0 / n_in)
            
            b = np.zeros((1, n_out))
            self.weights.append(w)
            self.biases.append(b)
    
    def relu(self, z):
        return np.maximum(0, z)
    
    def relu_derivative(self, z):
        return (z > 0).astype(float)
    
    def softmax(self, z):
        # Numerical stability: subtract max
        exp_z = np.exp(z - np.max(z, axis=1, keepdims=True))
        return exp_z / np.sum(exp_z, axis=1, keepdims=True)
    
    def forward(self, X):
        """Forward pass — compute all layer outputs."""
        self.activations = [X]  # Store for backprop
        self.z_values = []      # Pre-activation values
        
        current = X
        for i in range(self.n_layers):
            z = current @ self.weights[i] + self.biases[i]
            self.z_values.append(z)
            
            if i == self.n_layers - 1:
                # Output layer: softmax
                current = self.softmax(z)
            else:
                # Hidden layers: ReLU
                current = self.relu(z)
            
            self.activations.append(current)
        
        return current
    
    def compute_loss(self, y_pred, y_true):
        """Cross-entropy loss."""
        # Clip to prevent log(0)
        y_pred = np.clip(y_pred, 1e-15, 1 - 1e-15)
        # One-hot encoded y_true
        loss = -np.sum(y_true * np.log(y_pred)) / y_true.shape[0]
        return loss
    
    def backward(self, y_true, learning_rate=0.001):
        """Backpropagation — compute gradients and update weights."""
        m = y_true.shape[0]  # Batch size
        
        # Output layer gradient (softmax + cross-entropy shortcut)
        delta = self.activations[-1] - y_true  # Shape: (m, n_classes)
        
        for i in range(self.n_layers - 1, -1, -1):
            # Gradient for weights and biases
            dW = self.activations[i].T @ delta / m
            db = np.sum(delta, axis=0, keepdims=True) / m
            
            # Propagate gradient to previous layer (if not input)
            if i > 0:
                delta = (delta @ self.weights[i].T) * self.relu_derivative(self.z_values[i-1])
            
            # Update parameters
            self.weights[i] -= learning_rate * dW
            self.biases[i] -= learning_rate * db
    
    def train(self, X, y, epochs=100, lr=0.001, batch_size=32, verbose=True):
        """Training loop with mini-batches."""
        n_samples = X.shape[0]
        
        for epoch in range(epochs):
            # Shuffle data each epoch
            indices = np.random.permutation(n_samples)
            X_shuffled = X[indices]
            y_shuffled = y[indices]
            
            epoch_loss = 0
            n_batches = 0
            
            for start in range(0, n_samples, batch_size):
                end = min(start + batch_size, n_samples)
                X_batch = X_shuffled[start:end]
                y_batch = y_shuffled[start:end]
                
                # Forward pass
                y_pred = self.forward(X_batch)
                epoch_loss += self.compute_loss(y_pred, y_batch)
                n_batches += 1
                
                # Backward pass
                self.backward(y_batch, learning_rate=lr)
            
            if verbose and (epoch + 1) % 10 == 0:
                avg_loss = epoch_loss / n_batches
                acc = self.accuracy(X, y)
                print(f"Epoch {epoch+1}/{epochs} - Loss: {avg_loss:.4f} - Accuracy: {acc:.4f}")
    
    def predict(self, X):
        """Return class predictions."""
        probs = self.forward(X)
        return np.argmax(probs, axis=1)
    
    def accuracy(self, X, y):
        """Compute accuracy."""
        predictions = self.predict(X)
        true_labels = np.argmax(y, axis=1)
        return np.mean(predictions == true_labels)


# === Demo with synthetic data ===
from sklearn.datasets import make_moons
from sklearn.preprocessing import OneHotEncoder

# Generate non-linear data
X, y = make_moons(n_samples=1000, noise=0.2, random_state=42)

# One-hot encode labels
y_onehot = np.zeros((y.shape[0], 2))
y_onehot[np.arange(y.shape[0]), y] = 1

# Train the MLP
mlp = MLP(layer_sizes=[2, 64, 32, 2], activation='relu')
mlp.train(X, y_onehot, epochs=100, lr=0.01, batch_size=32)
print(f"Final Accuracy: {mlp.accuracy(X, y_onehot):.4f}")
# Output: ~0.97+ accuracy
```

---

## 5. Forward Propagation

### What It Is
Forward propagation is the process of passing input data through the network layer by layer to produce an output. Think of it as water flowing through pipes — input enters from one end and prediction comes out the other.

### How It Works

For each layer $l$:

$$z^{[l]} = W^{[l]} \cdot a^{[l-1]} + b^{[l]}$$
$$a^{[l]} = g^{[l]}(z^{[l]})$$

Where:
- $a^{[0]} = X$ (input)
- $z^{[l]}$ = pre-activation (linear transformation)
- $a^{[l]}$ = post-activation (after applying activation function)
- $g^{[l]}$ = activation function for layer $l$

### Worked Example

```
Network: 2 inputs → 2 hidden (ReLU) → 1 output (sigmoid)

Inputs: x = [1.0, 0.5]

Weights & Biases:
  W¹ = [[0.2, 0.4],    b¹ = [0.1, -0.1]
         [0.6, 0.8]]
  W² = [[0.3],         b² = [0.05]
         [0.7]]

Step 1: Input → Hidden Layer
  z₁⁽¹⁾ = (1.0×0.2) + (0.5×0.6) + 0.1 = 0.2 + 0.3 + 0.1 = 0.6
  z₂⁽¹⁾ = (1.0×0.4) + (0.5×0.8) + (-0.1) = 0.4 + 0.4 - 0.1 = 0.7
  
  a₁⁽¹⁾ = ReLU(0.6) = 0.6
  a₂⁽¹⁾ = ReLU(0.7) = 0.7

Step 2: Hidden → Output Layer
  z⁽²⁾ = (0.6×0.3) + (0.7×0.7) + 0.05 = 0.18 + 0.49 + 0.05 = 0.72
  
  a⁽²⁾ = sigmoid(0.72) = 1/(1+e⁻⁰·⁷²) ≈ 0.672

Output: ŷ = 0.672 (probability of class 1)
```

### Vectorized Forward Pass

```python
import numpy as np

def forward_pass(X, weights, biases, activations):
    """
    Vectorized forward propagation for a batch of inputs.
    
    Args:
        X: Input matrix (batch_size, n_features)
        weights: List of weight matrices
        biases: List of bias vectors
        activations: List of activation function names
    
    Returns:
        Output predictions and cache for backprop
    """
    cache = {'A0': X}
    A = X
    
    for l in range(len(weights)):
        # Linear transformation: Z = A·W + b
        Z = A @ weights[l] + biases[l]
        cache[f'Z{l+1}'] = Z
        
        # Apply activation
        if activations[l] == 'relu':
            A = np.maximum(0, Z)
        elif activations[l] == 'sigmoid':
            A = 1 / (1 + np.exp(-Z))
        elif activations[l] == 'softmax':
            exp_Z = np.exp(Z - np.max(Z, axis=1, keepdims=True))
            A = exp_Z / np.sum(exp_Z, axis=1, keepdims=True)
        
        cache[f'A{l+1}'] = A
    
    return A, cache

# Example usage
X = np.random.randn(32, 784)  # Batch of 32 images (28x28 flattened)
W1 = np.random.randn(784, 128) * 0.01
b1 = np.zeros((1, 128))
W2 = np.random.randn(128, 10) * 0.01
b2 = np.zeros((1, 10))

output, cache = forward_pass(
    X, [W1, W2], [b1, b2], ['relu', 'softmax']
)
print(f"Output shape: {output.shape}")  # (32, 10)
print(f"Sum of probabilities: {output[0].sum():.4f}")  # 1.0000
```

---

## 6. Loss Functions (Intro)

### What They Are
A loss function measures **how wrong** your network's predictions are. It's the "grade" your network gets on each prediction. The goal of training is to minimize this loss.

### Why They Matter
- They define what "good" means for your model
- Different problems need different loss functions
- The choice of loss function affects training dynamics

### Common Loss Functions

#### Mean Squared Error (MSE) — Regression

$$\mathcal{L}_{MSE} = \frac{1}{n}\sum_{i=1}^{n}(y_i - \hat{y}_i)^2$$

```python
def mse_loss(y_true, y_pred):
    """Mean Squared Error — for regression tasks."""
    return np.mean((y_true - y_pred) ** 2)

# Derivative (for backprop):
def mse_gradient(y_true, y_pred):
    return 2 * (y_pred - y_true) / y_true.shape[0]
```

#### Binary Cross-Entropy — Binary Classification

$$\mathcal{L}_{BCE} = -\frac{1}{n}\sum_{i=1}^{n}\left[y_i\log(\hat{y}_i) + (1-y_i)\log(1-\hat{y}_i)\right]$$

```python
def binary_cross_entropy(y_true, y_pred):
    """Binary Cross-Entropy — for binary classification."""
    # Clip to avoid log(0)
    y_pred = np.clip(y_pred, 1e-15, 1 - 1e-15)
    return -np.mean(y_true * np.log(y_pred) + (1 - y_true) * np.log(1 - y_pred))
```

#### Categorical Cross-Entropy — Multi-class Classification

$$\mathcal{L}_{CCE} = -\frac{1}{n}\sum_{i=1}^{n}\sum_{c=1}^{C} y_{i,c} \log(\hat{y}_{i,c})$$

```python
def categorical_cross_entropy(y_true, y_pred):
    """Categorical Cross-Entropy — for multi-class classification."""
    y_pred = np.clip(y_pred, 1e-15, 1 - 1e-15)
    return -np.sum(y_true * np.log(y_pred)) / y_true.shape[0]
```

### Loss Function Selection Guide

| Problem Type | Loss Function | Output Activation | Output Shape |
|---|---|---|---|
| Regression | MSE or MAE | Linear (none) | (n, 1) |
| Binary Classification | Binary Cross-Entropy | Sigmoid | (n, 1) |
| Multi-class (one label) | Categorical Cross-Entropy | Softmax | (n, C) |
| Multi-label | Binary Cross-Entropy (per label) | Sigmoid | (n, C) |

> **Key insight:** Cross-entropy loss paired with softmax/sigmoid has a clean gradient: $\frac{\partial \mathcal{L}}{\partial z} = \hat{y} - y$. This is NOT a coincidence — it's by mathematical design and makes training stable.

---

## 7. Backpropagation

### What It Is
Backpropagation (backprop) is the algorithm that calculates how much each weight contributed to the error, so we know how to adjust it. It's the **chain rule of calculus** applied systematically through the network.

### The Analogy
Imagine a **factory assembly line** that made a defective product:
- The inspector (loss function) finds the defect at the end
- They trace back: "Which machine caused this?"
- Each machine (layer) gets feedback on how to adjust
- Machines closer to the end get clearer feedback (gradients are stronger)
- Machines at the start get diluted feedback (vanishing gradient problem)

### Why It Matters
- It's THE reason deep learning works
- Without backprop, we couldn't train networks with millions of parameters
- Understanding it deeply helps you debug training issues

### The Math — Chain Rule in Action

For a simple network: Input → Hidden → Output

**Forward:**
$$z^{[1]} = W^{[1]}x + b^{[1]}, \quad a^{[1]} = g(z^{[1]})$$
$$z^{[2]} = W^{[2]}a^{[1]} + b^{[2]}, \quad \hat{y} = \sigma(z^{[2]})$$

**Loss:** $\mathcal{L} = -(y\log\hat{y} + (1-y)\log(1-\hat{y}))$

**Backward (Chain Rule):**

$$\frac{\partial \mathcal{L}}{\partial W^{[2]}} = \frac{\partial \mathcal{L}}{\partial \hat{y}} \cdot \frac{\partial \hat{y}}{\partial z^{[2]}} \cdot \frac{\partial z^{[2]}}{\partial W^{[2]}}$$

Let $\delta^{[2]} = \hat{y} - y$ (output error):

$$\frac{\partial \mathcal{L}}{\partial W^{[2]}} = \delta^{[2]} \cdot (a^{[1]})^T$$
$$\frac{\partial \mathcal{L}}{\partial b^{[2]}} = \delta^{[2]}$$

Propagate error to hidden layer:

$$\delta^{[1]} = (W^{[2]})^T \cdot \delta^{[2]} \odot g'(z^{[1]})$$
$$\frac{\partial \mathcal{L}}{\partial W^{[1]}} = \delta^{[1]} \cdot x^T$$
$$\frac{\partial \mathcal{L}}{\partial b^{[1]}} = \delta^{[1]}$$

### Visual Flow of Backpropagation

```
FORWARD PASS (left to right):
═══════════════════════════════════════════════════════════
x ──→ [W¹,b¹] ──→ z¹ ──→ g(z¹) ──→ a¹ ──→ [W²,b²] ──→ z² ──→ σ(z²) ──→ ŷ ──→ L(ŷ,y)


BACKWARD PASS (right to left):
═══════════════════════════════════════════════════════════
                                                              dL/dŷ
                                                         ←─── = ŷ-y ───
                                               dL/dz²                    
                                          ←─── = δ² ──────              
                              dL/dW²                        dL/db²       
                         ←─── = δ²·a¹ᵀ                ←─── = δ² ───     
                  dL/da¹                                                 
             ←─── = W²ᵀ·δ² ──                                          
       dL/dz¹                                                            
  ←─── = W²ᵀ·δ²⊙g'(z¹) = δ¹                                          
dL/dW¹                         dL/db¹                                    
←── = δ¹·xᵀ              ←─── = δ¹ ──                                  
```

### Code Example: Backpropagation Step by Step

```python
import numpy as np

def backprop_example():
    """
    Complete backpropagation example for a 2-layer network.
    Network: 2 inputs → 3 hidden (ReLU) → 1 output (sigmoid)
    """
    np.random.seed(42)
    
    # === SETUP ===
    # Single training example
    x = np.array([[1.0, 0.5]])  # Shape: (1, 2)
    y = np.array([[1.0]])        # True label
    
    # Random weights
    W1 = np.random.randn(2, 3) * 0.5  # (2, 3)
    b1 = np.zeros((1, 3))              # (1, 3)
    W2 = np.random.randn(3, 1) * 0.5  # (3, 1)
    b2 = np.zeros((1, 1))              # (1, 1)
    
    # === FORWARD PASS ===
    print("=== FORWARD PASS ===")
    
    # Layer 1
    z1 = x @ W1 + b1                    # Linear: (1,2)@(2,3) = (1,3)
    a1 = np.maximum(0, z1)              # ReLU activation
    print(f"z1 = {z1}")
    print(f"a1 = {a1}")
    
    # Layer 2 (output)
    z2 = a1 @ W2 + b2                   # Linear: (1,3)@(3,1) = (1,1)
    y_hat = 1 / (1 + np.exp(-z2))      # Sigmoid activation
    print(f"z2 = {z2}")
    print(f"ŷ  = {y_hat}")
    
    # Loss (Binary Cross-Entropy)
    loss = -(y * np.log(y_hat) + (1-y) * np.log(1-y_hat))
    print(f"Loss = {loss[0,0]:.6f}")
    
    # === BACKWARD PASS ===
    print("\n=== BACKWARD PASS ===")
    
    # Output layer gradient
    # For sigmoid + BCE: dL/dz2 = ŷ - y (elegant!)
    dz2 = y_hat - y                      # (1, 1)
    print(f"δ² (output error) = {dz2}")
    
    # Gradients for W2, b2
    dW2 = a1.T @ dz2                     # (3,1)@(1,1) → (3,1)
    db2 = dz2                            # (1, 1)
    print(f"dW2 = {dW2.flatten()}")
    print(f"db2 = {db2.flatten()}")
    
    # Propagate error to hidden layer
    da1 = dz2 @ W2.T                    # (1,1)@(1,3) → (1,3)
    dz1 = da1 * (z1 > 0).astype(float)  # Element-wise × ReLU derivative
    print(f"δ¹ (hidden error) = {dz1}")
    
    # Gradients for W1, b1
    dW1 = x.T @ dz1                      # (2,1)@(1,3) → (2,3)
    db1 = dz1                            # (1, 3)
    print(f"dW1 =\n{dW1}")
    print(f"db1 = {db1}")
    
    # === UPDATE WEIGHTS ===
    lr = 0.1
    W1 -= lr * dW1
    b1 -= lr * db1
    W2 -= lr * dW2
    b2 -= lr * db2
    
    # Verify: loss should decrease after update
    z1_new = x @ W1 + b1
    a1_new = np.maximum(0, z1_new)
    z2_new = a1_new @ W2 + b2
    y_hat_new = 1 / (1 + np.exp(-z2_new))
    loss_new = -(y * np.log(y_hat_new) + (1-y) * np.log(1-y_hat_new))
    
    print(f"\n=== VERIFICATION ===")
    print(f"Loss before: {loss[0,0]:.6f}")
    print(f"Loss after:  {loss_new[0,0]:.6f}")
    print(f"Improvement: {(loss[0,0] - loss_new[0,0]):.6f} ✓")

backprop_example()
```

### Computational Graph Perspective

Modern frameworks (PyTorch, TensorFlow) use **automatic differentiation** through computational graphs:

```
Forward: Build the graph
         x → multiply(W) → add(b) → relu → multiply(W) → add(b) → sigmoid → loss

Backward: Traverse graph in reverse, applying chain rule at each node
         Each node knows its local gradient
         Global gradient = product of local gradients along the path
```

> **Key Insight:** Backprop is just the chain rule applied efficiently. It computes ALL gradients in ONE backward pass (O(n) time), rather than computing each gradient independently (which would be O(n²)).

---

## 8. Gradient Descent

### What It Is
Gradient descent is the optimization algorithm that uses the gradients computed by backprop to update weights. It moves parameters in the direction that reduces the loss.

### The Analogy
Imagine you're **blindfolded on a hilly landscape** trying to reach the lowest valley:
- You feel the slope under your feet (gradient)
- You take a step downhill (weight update)
- Step size = learning rate
- Too big a step → you overshoot the valley
- Too small → takes forever to get there

### The Update Rule

$$W = W - \alpha \cdot \frac{\partial \mathcal{L}}{\partial W}$$

Where $\alpha$ is the learning rate.

### Variants of Gradient Descent

#### 1. Batch Gradient Descent (BGD)
Uses the ENTIRE dataset to compute one gradient update.

```python
# Batch Gradient Descent
for epoch in range(n_epochs):
    gradient = compute_gradient(X_train, y_train)  # ALL data
    weights -= learning_rate * gradient
```

- ✅ Stable convergence, true gradient
- ❌ Extremely slow for large datasets, high memory usage
- ❌ Can get stuck in local minima

#### 2. Stochastic Gradient Descent (SGD)
Uses ONE random sample per update.

```python
# Stochastic Gradient Descent
for epoch in range(n_epochs):
    shuffle(X_train, y_train)
    for xi, yi in zip(X_train, y_train):
        gradient = compute_gradient(xi, yi)  # ONE sample
        weights -= learning_rate * gradient
```

- ✅ Fast, can escape local minima (noise helps)
- ❌ Very noisy, oscillates, doesn't converge smoothly

#### 3. Mini-Batch Gradient Descent (Most Common)
Uses a small batch (32-512 samples) per update. **This is what everyone uses.**

```python
# Mini-Batch Gradient Descent
batch_size = 32
for epoch in range(n_epochs):
    shuffle(X_train, y_train)
    for i in range(0, len(X_train), batch_size):
        X_batch = X_train[i:i+batch_size]
        y_batch = y_train[i:i+batch_size]
        gradient = compute_gradient(X_batch, y_batch)
        weights -= learning_rate * gradient
```

- ✅ Best of both worlds: stable yet efficient
- ✅ Leverages GPU parallelism (matrix operations on batches)
- ✅ Moderate noise helps generalization

### Learning Rate — The Most Important Hyperparameter

```
Loss
  │
  │  ╲     lr too high: diverges!
  │   ╲  /╲  /╲
  │    ╲/  ╲/  ╲  ...
  │
  │  ╲
  │   ╲        lr just right: converges smoothly
  │    ╲___
  │        ╲___________
  │
  │  ╲
  │   |     lr too low: converges very slowly
  │   |
  │    ╲
  │     ╲
  │      ╲
  │       ╲___
  └──────────────────── Epochs
```

**Typical learning rates:**
- SGD: 0.01 - 0.1
- Adam: 0.001 - 0.0001
- Fine-tuning: 1e-5 - 1e-4

### Code Example: Visualizing Gradient Descent

```python
import numpy as np
import matplotlib.pyplot as plt

def gradient_descent_demo():
    """
    Visualize gradient descent on a simple 2D loss surface.
    Loss function: L(w1, w2) = w1² + 3*w2² (elliptical bowl)
    """
    # Define loss function and its gradient
    def loss(w):
        return w[0]**2 + 3 * w[1]**2
    
    def gradient(w):
        return np.array([2 * w[0], 6 * w[1]])
    
    # === Standard SGD ===
    w = np.array([4.0, 3.0])  # Starting point
    lr = 0.1
    path_sgd = [w.copy()]
    
    for _ in range(50):
        w = w - lr * gradient(w)
        path_sgd.append(w.copy())
    
    # === SGD with Momentum ===
    w = np.array([4.0, 3.0])
    velocity = np.array([0.0, 0.0])
    momentum = 0.9
    path_momentum = [w.copy()]
    
    for _ in range(50):
        velocity = momentum * velocity - lr * gradient(w)
        w = w + velocity
        path_momentum.append(w.copy())
    
    # Print results
    path_sgd = np.array(path_sgd)
    path_momentum = np.array(path_momentum)
    
    print(f"SGD final position: {path_sgd[-1]}, loss: {loss(path_sgd[-1]):.6f}")
    print(f"Momentum final: {path_momentum[-1]}, loss: {loss(path_momentum[-1]):.6f}")
    print(f"SGD steps to loss < 0.01: {np.argmax(np.array([loss(p) for p in path_sgd]) < 0.01)}")
    print(f"Momentum steps to loss < 0.01: {np.argmax(np.array([loss(p) for p in path_momentum]) < 0.01)}")

gradient_descent_demo()
# SGD: oscillates along the steep direction (w2)
# Momentum: dampens oscillations, converges faster
```

---

## 9. Universal Approximation Theorem

### What It Is
The **Universal Approximation Theorem** states that a feedforward network with a single hidden layer containing a finite number of neurons can approximate any continuous function on a compact subset of $\mathbb{R}^n$, to any desired degree of accuracy.

### In Plain English
"A neural network with just one hidden layer (if wide enough) can learn to mimic ANY smooth function."

### Why This Matters (And Why It's Misleading)

**What it guarantees:**
- A wide enough single-layer network CAN represent any function
- Neural networks are theoretically as powerful as any model

**What it does NOT guarantee:**
- That gradient descent will FIND that solution
- How many neurons you need (could be exponentially many)
- That training will be efficient
- That the network will generalize to unseen data

> **Critical insight:** The theorem says nothing about **learnability**. In practice, deep (many layers) networks learn better representations with fewer total parameters than wide shallow networks. This is why "deep learning" works — depth gives you compositional representations.

### Depth vs Width

```
SHALLOW & WIDE:                    DEEP & NARROW:
[Input: 2] → [Hidden: 10000] → [Output: 1]    [Input: 2] → [64] → [32] → [16] → [Output: 1]
Parameters: ~20,000+                           Parameters: ~3,500
Memorizes, doesn't generalize                  Learns hierarchical features
                                               Layer 1: edges
                                               Layer 2: shapes
                                               Layer 3: objects
```

**Analogy:** Writing a book as one GIANT sentence vs. organizing it into chapters, paragraphs, and sentences. Same information, but the structured version is easier to write, read, and modify.

---

## 10. Building Your First Neural Network

### With PyTorch

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
import numpy as np

# === 1. Generate Data ===
X, y = make_classification(
    n_samples=2000, n_features=20, n_informative=15,
    n_redundant=5, random_state=42
)

# Split and scale
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
scaler = StandardScaler()
X_train = scaler.fit_transform(X_train)
X_test = scaler.transform(X_test)

# Convert to PyTorch tensors
X_train_t = torch.FloatTensor(X_train)
y_train_t = torch.FloatTensor(y_train).unsqueeze(1)  # Shape: (n, 1)
X_test_t = torch.FloatTensor(X_test)
y_test_t = torch.FloatTensor(y_test).unsqueeze(1)

# Create DataLoader for batching
train_dataset = TensorDataset(X_train_t, y_train_t)
train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True)

# === 2. Define the Network ===
class BinaryClassifier(nn.Module):
    def __init__(self, input_dim):
        super(BinaryClassifier, self).__init__()
        self.network = nn.Sequential(
            nn.Linear(input_dim, 64),     # Layer 1: 20 → 64
            nn.ReLU(),                     # Activation
            nn.Linear(64, 32),            # Layer 2: 64 → 32
            nn.ReLU(),                     # Activation
            nn.Linear(32, 1),             # Output: 32 → 1
            nn.Sigmoid()                   # Probability output
        )
    
    def forward(self, x):
        return self.network(x)

# === 3. Setup Training ===
model = BinaryClassifier(input_dim=20)
criterion = nn.BCELoss()               # Binary Cross-Entropy
optimizer = optim.Adam(model.parameters(), lr=0.001)

# Print model summary
print(model)
total_params = sum(p.numel() for p in model.parameters())
print(f"Total parameters: {total_params}")

# === 4. Training Loop ===
n_epochs = 50
for epoch in range(n_epochs):
    model.train()
    epoch_loss = 0
    
    for X_batch, y_batch in train_loader:
        # Forward pass
        y_pred = model(X_batch)
        loss = criterion(y_pred, y_batch)
        
        # Backward pass
        optimizer.zero_grad()  # Clear old gradients!
        loss.backward()        # Compute gradients
        optimizer.step()       # Update weights
        
        epoch_loss += loss.item()
    
    # Evaluate every 10 epochs
    if (epoch + 1) % 10 == 0:
        model.eval()
        with torch.no_grad():
            y_pred_test = model(X_test_t)
            test_loss = criterion(y_pred_test, y_test_t)
            accuracy = ((y_pred_test > 0.5).float() == y_test_t).float().mean()
            print(f"Epoch {epoch+1}/{n_epochs} | "
                  f"Train Loss: {epoch_loss/len(train_loader):.4f} | "
                  f"Test Loss: {test_loss:.4f} | "
                  f"Test Acc: {accuracy:.4f}")

# === 5. Final Evaluation ===
model.eval()
with torch.no_grad():
    y_pred_final = (model(X_test_t) > 0.5).float()
    final_acc = (y_pred_final == y_test_t).float().mean()
    print(f"\nFinal Test Accuracy: {final_acc:.4f}")
```

### With TensorFlow/Keras

```python
import tensorflow as tf
from tensorflow import keras
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler

# === 1. Data Preparation ===
X, y = make_classification(n_samples=2000, n_features=20, n_informative=15,
                           n_redundant=5, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
scaler = StandardScaler()
X_train = scaler.fit_transform(X_train)
X_test = scaler.transform(X_test)

# === 2. Build Model (Sequential API) ===
model = keras.Sequential([
    keras.layers.Dense(64, activation='relu', input_shape=(20,)),  # Hidden 1
    keras.layers.Dense(32, activation='relu'),                     # Hidden 2
    keras.layers.Dense(1, activation='sigmoid')                    # Output
])

# === 3. Compile ===
model.compile(
    optimizer='adam',
    loss='binary_crossentropy',
    metrics=['accuracy']
)

# Print summary
model.summary()

# === 4. Train ===
history = model.fit(
    X_train, y_train,
    epochs=50,
    batch_size=32,
    validation_split=0.2,  # Use 20% of training data for validation
    verbose=1
)

# === 5. Evaluate ===
test_loss, test_acc = model.evaluate(X_test, y_test)
print(f"\nTest Accuracy: {test_acc:.4f}")
```

---

## 11. Common Mistakes

### Mistake 1: Not Normalizing Input Data
```python
# ❌ BAD: Raw features with different scales
X_train_raw = [[1000, 0.01], [2000, 0.02], ...]  # Feature 1 dominates!

# ✅ GOOD: Standardize all features
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler()
X_train = scaler.fit_transform(X_train_raw)
X_test = scaler.transform(X_test_raw)  # Use SAME scaler on test!
```

### Mistake 2: Wrong Output Activation + Loss Combination
```python
# ❌ BAD: Sigmoid output with MSE loss (gradients vanish at extremes)
model.add(Dense(1, activation='sigmoid'))
model.compile(loss='mse')

# ✅ GOOD: Match activation to loss
# Binary classification:
model.add(Dense(1, activation='sigmoid'))
model.compile(loss='binary_crossentropy')

# Multi-class:
model.add(Dense(10, activation='softmax'))
model.compile(loss='categorical_crossentropy')
```

### Mistake 3: Forgetting `optimizer.zero_grad()` in PyTorch
```python
# ❌ BAD: Gradients accumulate across batches!
for batch in data_loader:
    loss = criterion(model(batch_x), batch_y)
    loss.backward()
    optimizer.step()

# ✅ GOOD: Clear gradients before each backward pass
for batch in data_loader:
    optimizer.zero_grad()  # ALWAYS do this first!
    loss = criterion(model(batch_x), batch_y)
    loss.backward()
    optimizer.step()
```

### Mistake 4: Using Sigmoid for Multi-class Classification
```python
# ❌ BAD: Sigmoid gives independent probabilities (don't sum to 1)
model.add(Dense(10, activation='sigmoid'))

# ✅ GOOD: Softmax ensures probabilities sum to 1
model.add(Dense(10, activation='softmax'))
```

### Mistake 5: Not Setting Model to Eval Mode (PyTorch)
```python
# ❌ BAD: Dropout and BatchNorm still active during inference
predictions = model(X_test)

# ✅ GOOD: Set eval mode and disable gradient computation
model.eval()
with torch.no_grad():
    predictions = model(X_test)
```

### Mistake 6: Learning Rate Too High or Too Low
```python
# ❌ BAD: lr=1.0 → loss explodes to NaN
# ❌ BAD: lr=0.0000001 → trains for days, barely improves

# ✅ GOOD: Start with standard defaults
# Adam: lr=0.001 (most common starting point)
# SGD: lr=0.01 with momentum=0.9
```

### Mistake 7: Too Complex Model for Simple Data
```python
# ❌ BAD: 10 million parameters for 1000 training samples → overfitting
# ✅ GOOD: Parameters should be << training samples
# Rule of thumb: parameters ≤ 10x samples (at minimum)
```

---

## 12. Interview Questions

### Conceptual Questions

**Q1: What is the difference between a perceptron and a neural network?**
> A perceptron is a single neuron with a step activation function that can only learn linearly separable patterns. A neural network has multiple layers with non-linear activations, allowing it to learn complex non-linear decision boundaries.

**Q2: Why do we need non-linear activation functions?**
> Without non-linearity, any number of stacked linear layers collapses into a single linear transformation: $W_n \cdots W_2 W_1 x = Wx$. Non-linear activations allow the network to learn non-linear mappings.

**Q3: What is the vanishing gradient problem?**
> In deep networks with sigmoid/tanh activations, gradients become exponentially smaller as they propagate backward through layers (multiplied by derivatives < 1 repeatedly). Early layers learn extremely slowly or not at all. ReLU largely solves this by having gradient = 1 for positive inputs.

**Q4: Explain backpropagation in simple terms.**
> Backprop computes how much each weight contributed to the error by applying the chain rule of calculus from output to input. It's an efficient algorithm that computes all gradients in a single backward pass.

**Q5: What's the difference between Batch, Mini-batch, and Stochastic gradient descent?**
> - Batch: Uses ALL training data per update (slow but stable)
> - Stochastic: Uses ONE sample per update (fast but noisy)
> - Mini-batch: Uses a small batch (32-512) per update (best balance of speed and stability)

**Q6: Why is weight initialization important?**
> Bad initialization can cause vanishing/exploding gradients before training even begins. If all weights are zero, all neurons learn the same thing (symmetry problem). He initialization (for ReLU) and Xavier (for tanh) set proper variance based on layer sizes.

**Q7: What is the dying ReLU problem and how do you fix it?**
> If a ReLU neuron receives only negative inputs, it always outputs 0 and has 0 gradient — it can never recover. Solutions: Leaky ReLU (small negative slope), PReLU (learned slope), ELU, or careful initialization with smaller learning rates.

**Q8: Can a single hidden layer network approximate any function? What's the catch?**
> Yes (Universal Approximation Theorem), but: (1) it might need exponentially many neurons, (2) gradient descent might not find the right weights, (3) it might not generalize. Deep networks are more parameter-efficient in practice.

### Coding Questions

**Q9: Implement softmax function that is numerically stable.**
```python
def softmax(z):
    # Subtract max for numerical stability (prevents overflow)
    z_shifted = z - np.max(z, axis=-1, keepdims=True)
    exp_z = np.exp(z_shifted)
    return exp_z / np.sum(exp_z, axis=-1, keepdims=True)
```

**Q10: What's wrong with this training loop?**
```python
# Bug: model stays in training mode during evaluation
# Bug: no gradient disabling during evaluation → memory leak
for epoch in range(100):
    for batch in train_loader:
        optimizer.zero_grad()
        loss = criterion(model(batch_x), batch_y)
        loss.backward()
        optimizer.step()
    
    # Evaluation — MISSING model.eval() and torch.no_grad()
    val_loss = criterion(model(X_val), y_val)
```

---

## 13. Quick Reference

### Neural Network Building Blocks

| Component | Purpose | Key Choice |
|-----------|---------|------------|
| Input Layer | Receive features | Size = n_features |
| Hidden Layers | Learn representations | Width, depth, activation |
| Output Layer | Make predictions | Size = n_classes/1, activation matches task |
| Weights | Connection strength | Initialized carefully |
| Biases | Shift activation | Usually init to 0 |
| Activation | Introduce non-linearity | ReLU (hidden), Softmax/Sigmoid (output) |
| Loss Function | Measure error | CE (classification), MSE (regression) |
| Optimizer | Update weights | Adam (default), SGD+momentum (research) |
| Learning Rate | Step size | 0.001 (Adam), 0.01 (SGD) |

### Activation Function Cheat Sheet

| Situation | Use This |
|-----------|----------|
| Hidden layers (default) | ReLU |
| Dying neurons problem | Leaky ReLU / ELU |
| Transformer models | GELU |
| Binary output | Sigmoid |
| Multi-class output | Softmax |
| Regression output | None (Linear) |
| RNN hidden states | Tanh |

### Shapes Cheat Sheet

```
Input:  X       → (batch_size, n_features)
Weight: W[l]    → (n_in, n_out)   [for layer l]
Bias:   b[l]    → (1, n_out)
Output: Z = XW+b → (batch_size, n_out)
```

### Training Checklist

- [ ] Normalize/standardize input features
- [ ] Choose architecture appropriate for data size
- [ ] Use proper weight initialization (He for ReLU, Xavier for tanh)
- [ ] Match output activation to loss function
- [ ] Start with Adam optimizer, lr=0.001
- [ ] Use mini-batches (32-128 typical)
- [ ] Monitor both train AND validation loss
- [ ] Check for overfitting (train loss ↓, val loss ↑)

---

> **Next Chapter:** [02-Training-Deep-Networks.md](02-Training-Deep-Networks.md) — Loss functions in depth, advanced optimizers (Adam, RMSProp), learning rate scheduling, and training diagnostics.
