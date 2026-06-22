# Chapter 01: Generative AI Fundamentals

## Table of Contents
- [What is Generative AI?](#what-is-generative-ai)
- [History & Evolution](#history--evolution)
- [Types of Generative Models](#types-of-generative-models)
- [Key Concepts](#key-concepts)
- [How Generative AI Works](#how-generative-ai-works)
- [Use Cases & Applications](#use-cases--applications)
- [Code Examples](#code-examples)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Generative AI?

### Simple Explanation
Imagine you've read thousands of books. Now someone asks you to write a new story — you can create something *original* that sounds like it belongs with those books but doesn't copy any single one. That's Generative AI: AI systems that **create new content** (text, images, code, music, video) after learning patterns from massive amounts of existing data.

### Formal Definition
Generative AI refers to artificial intelligence systems that can generate new, previously unseen content by learning the underlying probability distribution of training data. Unlike **discriminative models** (which classify/predict), generative models learn **P(X)** or **P(X|Y)** — the probability of data itself.

### Generative vs Discriminative Models

| Aspect | Discriminative Models | Generative Models |
|--------|----------------------|-------------------|
| **Goal** | Learn decision boundary P(Y\|X) | Learn data distribution P(X) or P(X\|Y) |
| **Output** | Labels, classes, numbers | New data samples (text, images, etc.) |
| **Examples** | Logistic Regression, SVM, BERT (classification) | GPT, Stable Diffusion, GANs |
| **Analogy** | A judge who evaluates | An artist who creates |
| **Training** | Needs labeled data | Can learn from unlabeled data |

> **Key Insight**: A discriminative model learns to tell cats from dogs. A generative model learns what cats and dogs *look like* and can draw new ones.

---

## History & Evolution

### Timeline of Generative AI

```
1950s-60s: Early rule-based text generation (ELIZA chatbot)
     |
1980s: Boltzmann Machines, early neural generative models
     |
2013: Variational Autoencoders (VAEs) — Kingma & Welling
     |
2014: GANs invented — Ian Goodfellow (landmark paper)
     |
2017: ████ TRANSFORMER ARCHITECTURE ████ — "Attention Is All You Need"
     |     (This changed EVERYTHING)
     |
2018: GPT-1 (117M params) — OpenAI shows language generation works
     |
2019: GPT-2 (1.5B params) — "Too dangerous to release"
     |
2020: GPT-3 (175B params) — Few-shot learning emerges
     |
2021: DALL-E, Codex — Multi-modal generation
     |
2022: ChatGPT, Stable Diffusion — GenAI goes mainstream
     |
2023: GPT-4, LLaMA, Mistral — Open-source revolution
     |
2024: GPT-4o, Claude 3.5, Gemini — Multi-modal, reasoning
     |
2025-26: Agents, tool-use, reasoning models (o1, o3), open-weight dominance
```

### Why 2017 Was the Inflection Point

The Transformer paper by Vaswani et al. introduced **self-attention**, which:
1. Allowed parallel processing (unlike sequential RNNs)
2. Could handle long-range dependencies
3. Scaled efficiently with more data and compute
4. Became the backbone of ALL modern GenAI

---

## Types of Generative Models

### 1. Autoregressive Models (e.g., GPT)
- Generate one token at a time, left-to-right
- Each token is conditioned on all previous tokens
- $P(x_1, x_2, ..., x_n) = \prod_{i=1}^{n} P(x_i | x_1, ..., x_{i-1})$

```
Input:  "The cat sat on the"
Output: "The cat sat on the" → "mat"  (one token at a time)
```

### 2. Variational Autoencoders (VAEs)
- Encode input to a latent space, then decode
- Learn a compressed, continuous representation
- Good for smooth interpolation between outputs

```
Input Image → [Encoder] → Latent Vector z → [Decoder] → Reconstructed/New Image
```

### 3. Generative Adversarial Networks (GANs)
- Two networks compete: Generator vs Discriminator
- Generator creates fakes; Discriminator detects fakes
- Both improve through adversarial training

```
Random Noise → [Generator] → Fake Image
                                  ↓
Real Image → [Discriminator] → Real or Fake?
```

### 4. Diffusion Models (e.g., Stable Diffusion, DALL-E 3)
- Start with noise, progressively denoise to create content
- Learn to reverse a gradual noising process
- Currently state-of-the-art for image generation

```
Pure Noise → [Denoise Step 1] → [Step 2] → ... → [Step T] → Clean Image
```

### 5. Flow-Based Models (e.g., Glow)
- Learn invertible transformations between data and noise
- Exact likelihood computation (unlike GANs/VAEs)
- Less common in practice

### Comparison Table

| Model Type | Strengths | Weaknesses | Best For |
|-----------|-----------|------------|----------|
| Autoregressive | High quality text, flexible | Slow generation (sequential) | Text, code |
| VAE | Fast sampling, smooth latent space | Blurry outputs | Data compression, interpolation |
| GAN | Sharp, realistic images | Training instability, mode collapse | Image generation, style transfer |
| Diffusion | Best image quality, stable training | Slow inference | Images, video, audio |
| Flow-Based | Exact likelihood, invertible | Computationally expensive | Density estimation |

---

## Key Concepts

### 1. Tokens and Tokenization

Generative models don't see words — they see **tokens** (subword pieces).

```
"Understanding" → ["Under", "stand", "ing"]  (3 tokens)
"AI"            → ["AI"]                       (1 token)
"ChatGPT"       → ["Chat", "G", "PT"]         (3 tokens)
```

**Why subword tokenization?**
- Handles unknown words (out-of-vocabulary)
- Balances vocabulary size vs sequence length
- Common methods: BPE (Byte-Pair Encoding), WordPiece, SentencePiece

### 2. Temperature and Sampling

Temperature controls **randomness** in generation:

$$P(x_i) = \frac{\exp(z_i / T)}{\sum_j \exp(z_j / T)}$$

| Temperature | Behavior | Use Case |
|------------|----------|----------|
| T → 0 | Always picks highest probability token (deterministic) | Factual answers, code |
| T = 1.0 | Standard sampling (default) | General use |
| T > 1.0 | More random, creative, but risky | Creative writing, brainstorming |

**Analogy**: Temperature is like a "creativity dial" — turn it down for math homework, turn it up for poetry.

### 3. Top-k and Top-p (Nucleus) Sampling

**Top-k**: Only consider the top k most likely next tokens.
- k=1: Greedy decoding (always pick the best)
- k=50: Consider top 50 options

**Top-p (Nucleus)**: Consider the smallest set of tokens whose cumulative probability ≥ p.
- p=0.9: Consider tokens until you have 90% of probability mass
- Adapts dynamically (fewer choices when model is confident)

```
Token probabilities: [0.4, 0.25, 0.15, 0.1, 0.05, 0.03, 0.02]

Top-k (k=3):   Consider [0.4, 0.25, 0.15] → renormalize → sample
Top-p (p=0.8): Consider [0.4, 0.25, 0.15] → cumsum = 0.8 → sample from these
```

### 4. Context Window / Context Length

The maximum number of tokens a model can "see" at once.

| Model | Context Window |
|-------|---------------|
| GPT-3 | 4,096 tokens |
| GPT-4 | 8,192 / 128K tokens |
| Claude 3.5 | 200K tokens |
| Gemini 1.5 | 1M+ tokens |
| LLaMA 3 | 128K tokens |

> **Important**: Longer context ≠ better. Models can struggle with information in the "middle" of very long contexts (the "Lost in the Middle" problem).

### 5. Embeddings

Dense vector representations that capture semantic meaning:

```
"king"  → [0.2, 0.8, -0.1, 0.5, ...]   (768 or 1536 dimensions)
"queen" → [0.2, 0.9, -0.1, 0.4, ...]   (similar direction!)
"car"   → [0.9, -0.2, 0.7, -0.3, ...]  (very different direction)
```

**Key property**: Similar meanings → nearby vectors in embedding space.

$$\text{similarity}(\vec{a}, \vec{b}) = \frac{\vec{a} \cdot \vec{b}}{||\vec{a}|| \cdot ||\vec{b}||}$$

### 6. Hallucination

When a model generates confident-sounding but **factually incorrect** information.

**Why it happens**:
- Models learn patterns, not facts
- They maximize P(next_token | context), not P(truth)
- Training data has contradictions
- No built-in mechanism to verify claims

**Mitigation strategies**:
- Retrieval-Augmented Generation (RAG)
- Grounding with external knowledge
- Prompt engineering ("only answer based on provided context")
- Fine-tuning on high-quality data

### 7. Emergent Abilities

Capabilities that appear suddenly at certain model scales:

```
Small model (1B params):  Can complete sentences
Medium model (10B):       Can follow simple instructions
Large model (100B+):      Can reason, do math, write code
                          (These weren't explicitly trained!)
```

> **Key Insight**: You can't predict what a larger model will be able to do just by looking at smaller versions. Abilities "emerge" at scale thresholds.

---

## How Generative AI Works

### The Core Loop (Simplified)

```
┌─────────────────────────────────────────────────┐
│              TRAINING PHASE                       │
│                                                   │
│  Massive Data ──→ Learn Patterns ──→ Model Weights│
│  (Internet,       (Backprop,        (Billions of  │
│   books, code)     gradient descent)  parameters)  │
└─────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│              INFERENCE PHASE                      │
│                                                   │
│  User Prompt ──→ Model Processing ──→ Generated   │
│  "Write a       (Attention,           Output      │
│   poem about     token-by-token       "Roses are  │
│   roses"         prediction)           red..."     │
└─────────────────────────────────────────────────┘
```

### Training Objective for Language Models

**Next Token Prediction** (used by GPT-style models):

$$\mathcal{L} = -\sum_{i=1}^{N} \log P(x_i | x_1, x_2, ..., x_{i-1}; \theta)$$

The model reads text and tries to predict what comes next. By doing this billions of times on internet-scale data, it learns:
- Grammar and syntax
- Facts and knowledge
- Reasoning patterns
- Code structure
- Multiple languages

### The Three Stages of Modern LLM Training

```
Stage 1: PRE-TRAINING
├── Objective: Next-token prediction on massive corpus
├── Data: Trillions of tokens (web, books, code)
├── Compute: Thousands of GPUs for weeks/months
└── Result: Base model (knows language, has knowledge)

Stage 2: SUPERVISED FINE-TUNING (SFT)  
├── Objective: Follow instructions
├── Data: Human-written instruction-response pairs
├── Compute: Much less than pre-training
└── Result: Instruction-following model

Stage 3: ALIGNMENT (RLHF/DPO)
├── Objective: Be helpful, harmless, honest
├── Data: Human preference data (which response is better?)
├── Method: Reinforcement Learning from Human Feedback
└── Result: Aligned, safe model (ChatGPT, Claude)
```

### Scaling Laws

Kaplan et al. (2020) discovered predictable relationships:

$$L(N) \propto N^{-0.076}$$
$$L(D) \propto D^{-0.095}$$
$$L(C) \propto C^{-0.050}$$

Where:
- $L$ = Loss (lower is better)
- $N$ = Number of parameters
- $D$ = Dataset size
- $C$ = Compute budget

**Chinchilla Scaling** (Hoffmann et al., 2022): For optimal performance, scale data and parameters equally. A 70B model trained on 1.4T tokens beats a 280B model on 300B tokens.

---

## Use Cases & Applications

### Text Generation
- Chatbots and virtual assistants
- Content writing (articles, emails, marketing copy)
- Code generation (GitHub Copilot, Cursor)
- Translation and summarization

### Image Generation
- Art creation (Midjourney, DALL-E)
- Product design and prototyping
- Marketing materials
- Photo editing and enhancement

### Code & Software
- Code completion and generation
- Bug detection and fixing
- Documentation generation
- Test case generation

### Audio & Music
- Text-to-speech (ElevenLabs)
- Music composition (Suno, Udio)
- Voice cloning
- Audio transcription (Whisper)

### Video
- Text-to-video (Sora, Runway)
- Video editing and effects
- Animation generation

### Enterprise Applications
- Document processing and analysis
- Customer support automation
- Knowledge base Q&A (RAG)
- Data analysis and reporting

---

## Code Examples

### Example 1: Using OpenAI API (Text Generation)

```python
# Basic text generation with OpenAI's API
# Install: pip install openai

from openai import OpenAI

# Initialize the client (reads OPENAI_API_KEY from environment)
client = OpenAI()

# Simple completion
response = client.chat.completions.create(
    model="gpt-4o",                    # Model to use
    messages=[
        {
            "role": "system",           # System message sets behavior
            "content": "You are a helpful AI tutor explaining ML concepts."
        },
        {
            "role": "user",             # User's actual question
            "content": "Explain backpropagation in 3 sentences."
        }
    ],
    temperature=0.7,                    # Moderate creativity
    max_tokens=200,                     # Limit response length
    top_p=0.9                           # Nucleus sampling
)

# Extract the generated text
answer = response.choices[0].message.content
print(answer)

# Check token usage (important for cost management!)
print(f"Prompt tokens: {response.usage.prompt_tokens}")
print(f"Completion tokens: {response.usage.completion_tokens}")
print(f"Total tokens: {response.usage.total_tokens}")
```

### Example 2: Understanding Tokenization

```python
# Explore how tokenization works
# Install: pip install tiktoken

import tiktoken

# Load the tokenizer for GPT-4
encoding = tiktoken.encoding_for_model("gpt-4")

# Tokenize a sentence
text = "Generative AI is transforming the world!"
tokens = encoding.encode(text)

print(f"Original text: {text}")
print(f"Token IDs: {tokens}")
print(f"Number of tokens: {len(tokens)}")

# Decode individual tokens to see what they represent
for token_id in tokens:
    token_text = encoding.decode([token_id])
    print(f"  ID {token_id:>6} → '{token_text}'")

# Output:
# Original text: Generative AI is transforming the world!
# Token IDs: [5. .., ...]
# Number of tokens: 7
#   ID  31944 → 'Gener'
#   ID  1413  → 'ative'
#   ID  15592 → ' AI'
#   ID    374 → ' is'
#   ID  38291 → ' transforming'
#   ID    279 → ' the'
#   ID   1917 → ' world'
#   ID      0 → '!'

# Pro Tip: Estimate costs by counting tokens BEFORE making API calls
def estimate_cost(text, model="gpt-4o", cost_per_1k_input=0.0025):
    """Estimate API cost for a given text."""
    enc = tiktoken.encoding_for_model(model)
    num_tokens = len(enc.encode(text))
    cost = (num_tokens / 1000) * cost_per_1k_input
    return num_tokens, cost

tokens, cost = estimate_cost("Your very long prompt here..." * 100)
print(f"Tokens: {tokens}, Estimated cost: ${cost:.4f}")
```

### Example 3: Temperature and Sampling Effects

```python
from openai import OpenAI

client = OpenAI()

prompt = "Write a one-sentence description of the ocean."

# Compare different temperature settings
temperatures = [0.0, 0.5, 1.0, 1.5]

for temp in temperatures:
    responses = []
    # Generate 3 responses at each temperature to see variation
    for _ in range(3):
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=temp,
            max_tokens=50
        )
        responses.append(response.choices[0].message.content)
    
    print(f"\n{'='*60}")
    print(f"Temperature = {temp}")
    print(f"{'='*60}")
    for i, r in enumerate(responses, 1):
        print(f"  {i}. {r}")

# Expected behavior:
# temp=0.0: All 3 responses are IDENTICAL (deterministic)
# temp=0.5: Slight variations, still coherent
# temp=1.0: More variety, occasionally surprising
# temp=1.5: Wild creativity, sometimes nonsensical
```

### Example 4: Working with Embeddings

```python
# Understanding and using embeddings
# Install: pip install openai numpy scikit-learn

import numpy as np
from openai import OpenAI

client = OpenAI()

def get_embedding(text, model="text-embedding-3-small"):
    """Get embedding vector for a piece of text."""
    response = client.embeddings.create(
        input=text,
        model=model
    )
    return np.array(response.data[0].embedding)

def cosine_similarity(a, b):
    """Calculate cosine similarity between two vectors."""
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# Compare semantic similarity of sentences
sentences = [
    "The cat sat on the mat",           # About cats
    "A kitten was resting on the rug",  # Similar meaning!
    "Stock prices rose sharply today",   # Completely different
    "The dog played in the park",       # Animals, but different
]

# Get embeddings for all sentences
embeddings = [get_embedding(s) for s in sentences]

# Calculate pairwise similarities
print("Cosine Similarity Matrix:")
print(f"{'':40} ", end="")
for i in range(len(sentences)):
    print(f"S{i+1:>6}", end="")
print()

for i in range(len(sentences)):
    print(f"{sentences[i]:40} ", end="")
    for j in range(len(sentences)):
        sim = cosine_similarity(embeddings[i], embeddings[j])
        print(f"{sim:6.3f}", end="")
    print()

# Expected output shows:
# "cat on mat" and "kitten on rug" → HIGH similarity (~0.85)
# "cat on mat" and "stock prices" → LOW similarity (~0.15)
```

### Example 5: Simple Text Generation with Hugging Face (Local)

```python
# Run a generative model locally (no API key needed!)
# Install: pip install transformers torch

from transformers import pipeline, set_seed

# Load a text generation pipeline (downloads model on first run)
# Using a small model that runs on CPU
generator = pipeline(
    "text-generation",
    model="gpt2",              # 124M params, runs on any machine
    device=-1                  # -1 = CPU, 0 = first GPU
)

# Set seed for reproducibility
set_seed(42)

# Generate text
outputs = generator(
    "The future of artificial intelligence is",
    max_length=100,            # Maximum tokens to generate
    num_return_sequences=3,    # Generate 3 different completions
    temperature=0.8,           # Slightly creative
    top_k=50,                  # Consider top 50 tokens
    top_p=0.95,                # Nucleus sampling
    do_sample=True             # Enable sampling (vs greedy)
)

# Display results
for i, output in enumerate(outputs, 1):
    print(f"\n--- Completion {i} ---")
    print(output["generated_text"])


# Pro Tip: For better quality with a local model, use:
# model="microsoft/phi-2" (2.7B, good quality/size ratio)
# model="meta-llama/Llama-3.2-1B" (requires HF token)
```

### Example 6: Building a Simple GenAI Application

```python
# A complete mini-application: AI-powered FAQ answerer
# This demonstrates the pattern used in production GenAI apps

from openai import OpenAI

client = OpenAI()

# Knowledge base (in production, this comes from a database/vector store)
FAQ_KNOWLEDGE = """
Company: TechCorp
Products: CloudSync (file storage), DataFlow (analytics), SecureVault (security)
Pricing: Free tier available. Pro starts at $29/month. Enterprise is custom.
Support: 24/7 chat for Pro+, email for Free tier. Response time < 4 hours.
Refund Policy: Full refund within 30 days, no questions asked.
"""

def answer_question(user_question: str) -> str:
    """Answer a user question using the knowledge base."""
    
    # System prompt defines the AI's role and constraints
    system_prompt = f"""You are a helpful customer support agent for TechCorp.
    
RULES:
- Only answer based on the knowledge base provided below
- If the answer is not in the knowledge base, say "I don't have that information"
- Be concise and friendly
- Never make up information

KNOWLEDGE BASE:
{FAQ_KNOWLEDGE}
"""
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",           # Cheaper model for simple Q&A
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_question}
        ],
        temperature=0.3,                # Low temperature for factual answers
        max_tokens=200
    )
    
    return response.choices[0].message.content

# Test the FAQ bot
questions = [
    "What products do you offer?",
    "How much does the Pro plan cost?",
    "Can I get a refund after 60 days?",
    "What's the CEO's email?",         # Not in knowledge base!
]

for q in questions:
    print(f"\nQ: {q}")
    print(f"A: {answer_question(q)}")
```

---

## Common Mistakes

### 1. Treating LLM Output as Ground Truth
❌ **Wrong**: "GPT said X, so X must be true"
✅ **Right**: Always verify factual claims. LLMs are language models, not knowledge databases.

### 2. Ignoring Token Limits
❌ **Wrong**: Stuffing entire documents into a prompt without counting tokens
✅ **Right**: Always check token count. Use `tiktoken` to estimate before API calls.

```python
# Always check if your prompt fits the context window
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4o")
token_count = len(enc.encode(your_prompt))
if token_count > 128000:  # GPT-4o context limit
    # Chunk or summarize your input
    pass
```

### 3. Using High Temperature for Factual Tasks
❌ **Wrong**: `temperature=1.5` for "What is the capital of France?"
✅ **Right**: Use `temperature=0` for factual/deterministic tasks, higher for creative ones.

### 4. Not Setting a System Prompt
❌ **Wrong**: Only sending user messages
✅ **Right**: Always define behavior, constraints, and persona in the system prompt.

### 5. Overpaying by Using Wrong Models
❌ **Wrong**: Using GPT-4o for simple classification tasks
✅ **Right**: Use the cheapest model that achieves acceptable quality. GPT-4o-mini is 10-20x cheaper.

### 6. Expecting Deterministic Output
❌ **Wrong**: "Why does it give different answers each time?"
✅ **Right**: Set `temperature=0` and `seed=42` if you need reproducibility. Even then, minor variations can occur.

### 7. Not Handling Rate Limits and Errors
❌ **Wrong**: Making API calls without error handling
✅ **Right**: Always implement retries with exponential backoff.

```python
import time
from openai import OpenAI, RateLimitError

client = OpenAI()

def robust_completion(messages, max_retries=3):
    """API call with retry logic."""
    for attempt in range(max_retries):
        try:
            return client.chat.completions.create(
                model="gpt-4o-mini",
                messages=messages
            )
        except RateLimitError:
            wait_time = 2 ** attempt  # 1, 2, 4 seconds
            time.sleep(wait_time)
    raise Exception("Max retries exceeded")
```

---

## Interview Questions

### Conceptual Questions

**Q1: What is the difference between Generative AI and Traditional AI?**
> Traditional AI is typically discriminative — it classifies, predicts, or optimizes. Generative AI creates new content by learning the underlying data distribution. Traditional AI asks "what is this?" while GenAI asks "what could this look like?"

**Q2: Explain the difference between GPT and BERT at a high level.**
> GPT is autoregressive (generates left-to-right, one token at a time). BERT is bidirectional (sees the entire input at once). GPT excels at generation; BERT excels at understanding/classification. GPT uses causal masking; BERT uses masked language modeling.

**Q3: What are hallucinations and how do you mitigate them?**
> Hallucinations are confident but incorrect outputs. Mitigation: RAG (grounding with external data), lower temperature, explicit instructions to say "I don't know," fine-tuning on high-quality data, and post-generation fact-checking.

**Q4: Explain the concept of "emergent abilities" in LLMs.**
> Abilities that appear only at certain model scales and weren't explicitly trained. Examples: chain-of-thought reasoning, code generation, multi-step math. They're unpredictable — you can't extrapolate from smaller models. This is both exciting (new capabilities) and concerning (unexpected behaviors).

**Q5: What is the difference between fine-tuning and prompt engineering?**
> Prompt engineering modifies the input to get better outputs (no model changes). Fine-tuning modifies the model weights on new data. Use prompt engineering first (cheaper, faster). Fine-tune when you need consistent behavior, specialized knowledge, or specific output formats that prompting can't achieve.

### Technical Questions

**Q6: How does temperature affect token probabilities mathematically?**
> Temperature T divides the logits before softmax: $P(x_i) = \text{softmax}(z_i / T)$. T→0 makes the distribution peaked (argmax), T→∞ makes it uniform (random). It controls the entropy of the output distribution.

**Q7: What is the "Lost in the Middle" problem?**
> Models with long context windows perform worse on information placed in the middle of the context compared to the beginning or end. This is due to attention patterns during training. Mitigation: place important info at start/end, use structured retrieval.

**Q8: Explain scaling laws for LLMs.**
> Performance (loss) improves predictably as a power law of compute, data, and parameters. Chinchilla scaling shows optimal allocation: for a compute budget C, use $N \propto C^{0.5}$ parameters and $D \propto C^{0.5}$ tokens. This means many models are "over-parameterized and under-trained."

**Q9: What's the difference between Top-k and Top-p sampling?**
> Top-k always considers exactly k tokens regardless of probability distribution. Top-p (nucleus) adapts — it considers the minimum set of tokens whose cumulative probability ≥ p. Top-p is generally preferred because it adapts to the model's confidence: fewer choices when certain, more when uncertain.

**Q10: Why can't you just "make the context window infinite"?**
> Self-attention has $O(n^2)$ complexity with sequence length. A 1M token context costs 1000x more compute than 1K tokens. Also, retrieving relevant information from very long contexts degrades. Solutions: sparse attention, retrieval augmentation, hierarchical approaches.

---

## Quick Reference

### GenAI Model Landscape (2024-2026)

| Model | Company | Type | Key Strength |
|-------|---------|------|-------------|
| GPT-4o | OpenAI | Proprietary | Multi-modal, reasoning |
| Claude 3.5/4 | Anthropic | Proprietary | Long context, safety |
| Gemini 2.0 | Google | Proprietary | Multi-modal, speed |
| LLaMA 3 | Meta | Open-weight | Best open model |
| Mistral | Mistral AI | Open-weight | Efficient, European |
| Phi-3/4 | Microsoft | Open-weight | Small but powerful |
| Command R+ | Cohere | Proprietary | Enterprise RAG |

### Key Formulas

| Concept | Formula |
|---------|---------|
| Next-token prediction loss | $\mathcal{L} = -\sum \log P(x_i \| x_{<i})$ |
| Temperature scaling | $P(x_i) = \frac{e^{z_i/T}}{\sum_j e^{z_j/T}}$ |
| Cosine similarity | $\cos(\theta) = \frac{A \cdot B}{\|A\| \|B\|}$ |
| Scaling law | $L(N) \propto N^{-\alpha}$ |
| Perplexity | $PPL = e^{-\frac{1}{N}\sum \log P(x_i)}$ |

### Decision Framework: When to Use GenAI

| Use When... | Don't Use When... |
|-------------|-------------------|
| Task requires creativity/generation | Exact, deterministic answers needed |
| Problem is language/reasoning-heavy | Simple rule-based logic suffices |
| Few-shot examples can guide behavior | Large labeled datasets available for ML |
| Flexibility in output format is OK | Strict format compliance required |
| Human-like interaction is valued | Low-latency, high-throughput needed |

### Cost Optimization Cheat Sheet

| Strategy | Impact |
|----------|--------|
| Use smaller models (GPT-4o-mini vs GPT-4o) | 10-20x savings |
| Cache frequent responses | 50-80% savings for repetitive queries |
| Batch API calls | Lower per-token cost |
| Shorten prompts (remove unnecessary context) | Linear savings |
| Use open-source models locally | No per-token cost (but GPU cost) |

---

## Summary

Generative AI is the field of creating AI systems that produce new content. The key breakthroughs:
1. **Transformers** (2017) enabled parallel processing and scaling
2. **Scaling laws** showed predictable improvement with more compute/data
3. **RLHF** (2022+) made models useful and aligned with human values
4. **Open-source** (2023+) democratized access to powerful models

The field moves fast — what matters is understanding **fundamentals** (attention, tokenization, sampling) because specific models come and go, but the principles remain.
