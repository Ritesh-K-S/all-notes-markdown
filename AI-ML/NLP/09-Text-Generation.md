# Chapter 09: Text Generation

## Table of Contents
- [What is Text Generation](#what-is-text-generation)
- [Why Text Generation Matters](#why-text-generation-matters)
- [How Language Models Generate Text](#how-language-models-generate-text)
- [Decoding Strategies](#decoding-strategies)
- [Beam Search](#beam-search)
- [Sampling Methods](#sampling-methods)
- [Prompt Engineering](#prompt-engineering)
- [Controlling Generation](#controlling-generation)
- [Text Generation with HuggingFace](#text-generation-with-huggingface)
- [Building a Text Generation Pipeline](#building-a-text-generation-pipeline)
- [Evaluation of Generated Text](#evaluation-of-generated-text)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Text Generation

Text generation is the task of producing coherent, natural-sounding text given some input (a prompt, a topic, or a partial sentence). It's the technology behind chatbots, autocomplete, story generators, and coding assistants.

### Simple Analogy

Imagine you're playing a word-prediction game. Someone starts a sentence — "The cat sat on the..." — and you guess the next word: "mat", "roof", "table". A language model does exactly this, but it has read billions of sentences, so it's extremely good at predicting what word comes next. To generate a full paragraph, it just keeps predicting one word at a time, using everything it has generated so far as context.

### The Core Loop

```
Input prompt: "Once upon a time"
                    │
                    ▼
        ┌───────────────────────┐
        │   Language Model       │◄──────────────────┐
        │   P(next_word | context)│                   │
        └───────────┬───────────┘                    │
                    │                                 │
                    ▼                                 │
            Select next word                          │
            (greedy / sample / beam)                  │
                    │                                 │
                    ▼                                 │
        "Once upon a time there"                      │
                    │                                 │
                    └─── Append to context ───────────┘
                    
        Repeat until: max_length OR <EOS> token
```

---

## Why Text Generation Matters

### Real-World Applications

| Application | Description | Example |
|-------------|-------------|---------|
| **Chatbots & Assistants** | Conversational AI | ChatGPT, Claude, Gemini |
| **Content Creation** | Articles, marketing copy | Jasper, Copy.ai |
| **Code Generation** | Writing code from descriptions | GitHub Copilot, Cursor |
| **Creative Writing** | Stories, poetry, scripts | NovelAI, Sudowrite |
| **Translation** | Generate text in target language | Google Translate (neural) |
| **Data Augmentation** | Generate synthetic training data | Paraphrase generation |
| **Email/Reply Suggestions** | Smart compose features | Gmail Smart Compose |
| **Report Generation** | Convert data to narratives | Automated financial reports |
| **Dialogue Systems** | Game NPCs, virtual tutors | Character.ai |

### When to Use Text Generation

- When you need to **create** new text rather than extract from existing text
- When the output format is open-ended (not a fixed classification)
- When you want human-like fluency in output
- When you need flexibility in response length and style

---

## How Language Models Generate Text

### Autoregressive Generation

All modern text generation models are **autoregressive** — they generate one token at a time, left to right, conditioning on everything generated so far.

$$P(w_1, w_2, ..., w_T) = \prod_{t=1}^{T} P(w_t | w_1, w_2, ..., w_{t-1})$$

At each step $t$, the model produces a **probability distribution** over the entire vocabulary:

```
Step 1: Input = "The cat"
        P("sat") = 0.15, P("is") = 0.12, P("ran") = 0.08, P("was") = 0.07, ...

Step 2: Input = "The cat sat" (assuming we picked "sat")
        P("on") = 0.35, P("down") = 0.20, P("in") = 0.08, ...

Step 3: Input = "The cat sat on"
        P("the") = 0.45, P("a") = 0.15, P("my") = 0.08, ...
```

### The Vocabulary Distribution

At each step, the model outputs **logits** (raw scores) for every token in its vocabulary (typically 30K-100K tokens):

```
Vocabulary size: 50,257 (GPT-2)

Logits:  [2.1, -0.3, 5.7, 1.2, -4.5, ...]  ← Raw scores
              │
              ▼ Softmax
Probs:   [0.02, 0.001, 0.42, 0.01, 0.0001, ...]  ← Sum to 1.0
              │
              ▼ Decoding strategy
Next token: Depends on which strategy you use!
```

### Temperature

Temperature ($\tau$) controls the "sharpness" of the probability distribution:

$$P(w_i) = \frac{e^{z_i / \tau}}{\sum_j e^{z_j / \tau}}$$

Where $z_i$ is the logit for token $i$.

```
Temperature = 0.1 (very confident, almost deterministic):
  "the" = 0.95, "a" = 0.04, "my" = 0.009, ...

Temperature = 1.0 (standard):
  "the" = 0.45, "a" = 0.15, "my" = 0.08, ...

Temperature = 2.0 (very random, creative):
  "the" = 0.18, "a" = 0.12, "my" = 0.10, "his" = 0.09, ...
```

```python
import torch
import torch.nn.functional as F

def apply_temperature(logits, temperature):
    """
    Adjust logits by temperature before softmax.
    
    Low temperature → peaked distribution → more deterministic
    High temperature → flat distribution → more random/creative
    """
    if temperature == 0:
        # Greedy: just pick the highest logit
        return torch.zeros_like(logits).scatter_(
            -1, logits.argmax(dim=-1, keepdim=True), 1.0
        )
    
    scaled_logits = logits / temperature
    return F.softmax(scaled_logits, dim=-1)

# Example
logits = torch.tensor([2.0, 1.0, 0.5, 0.1, -1.0])
tokens = ["the", "a", "my", "his", "your"]

for temp in [0.1, 0.5, 1.0, 1.5, 2.0]:
    probs = apply_temperature(logits, temp)
    print(f"Temp={temp}: ", end="")
    for token, p in zip(tokens, probs):
        print(f"{token}={p:.3f}  ", end="")
    print()

# Output:
# Temp=0.1: the=1.000  a=0.000  my=0.000  his=0.000  your=0.000
# Temp=0.5: the=0.836  a=0.114  my=0.038  his=0.017  your=0.002
# Temp=1.0: the=0.466  a=0.171  my=0.104  his=0.070  your=0.023
# Temp=1.5: the=0.349  a=0.178  my=0.127  his=0.097  your=0.048
# Temp=2.0: the=0.296  a=0.182  my=0.143  his=0.118  your=0.072
```

---

## Decoding Strategies

### 1. Greedy Decoding

Always pick the token with the **highest probability** at each step.

```python
def greedy_decode(model, tokenizer, prompt, max_length=50):
    """
    Greedy decoding: always pick the most likely next token.
    
    Pros: Fast, deterministic
    Cons: Repetitive, misses better sequences, no diversity
    """
    input_ids = tokenizer.encode(prompt, return_tensors="pt")
    
    for _ in range(max_length):
        with torch.no_grad():
            outputs = model(input_ids)
            logits = outputs.logits[:, -1, :]  # Last token's logits
        
        # Pick token with highest probability
        next_token_id = logits.argmax(dim=-1, keepdim=True)
        
        # Stop if EOS token
        if next_token_id.item() == tokenizer.eos_token_id:
            break
        
        # Append to sequence
        input_ids = torch.cat([input_ids, next_token_id], dim=-1)
    
    return tokenizer.decode(input_ids[0], skip_special_tokens=True)
```

**Problem with Greedy Decoding:**

```
Greedy path:    "The" → "cat" → "is" → "a" → "cat" → "is" → "a" → ...  (repetitive!)

Better path:    "The" → "old" → "cat" → "sat" → "quietly" → "by" → "the" → "fire"
(not always highest probability at each step, but better overall)
```

### 2. Beam Search

Maintain $k$ best partial sequences (beams) at each step instead of just one.

```
Beam width = 3

Step 0: "The"
        │
Step 1: ├─ "The cat"    (score: -1.2)
        ├─ "The dog"    (score: -1.5)
        └─ "The old"    (score: -1.8)
        │
Step 2: ├─ "The cat sat"     (score: -2.1)  ← Keep
        ├─ "The cat is"      (score: -2.3)  ← Keep
        ├─ "The dog ran"     (score: -2.5)  ← Keep
        ├─ "The dog is"      (score: -2.7)  ← Drop
        ├─ "The old man"     (score: -2.9)  ← Drop
        └─ "The old cat"     (score: -3.1)  ← Drop
```

```python
def beam_search(model, tokenizer, prompt, beam_width=5, max_length=50, 
                length_penalty=1.0, no_repeat_ngram_size=3):
    """
    Beam search: explore multiple hypotheses in parallel.
    
    Pros: Better quality than greedy, finds globally better sequences
    Cons: Slower, still tends toward generic/safe outputs
    """
    input_ids = tokenizer.encode(prompt, return_tensors="pt")
    
    # Each beam: (sequence, cumulative log probability)
    beams = [(input_ids, 0.0)]
    completed = []
    
    for step in range(max_length):
        all_candidates = []
        
        for seq, score in beams:
            if seq[0, -1].item() == tokenizer.eos_token_id:
                # This beam is complete
                # Apply length penalty: score / length^alpha
                length_norm = ((5 + len(seq[0])) / 6) ** length_penalty
                completed.append((seq, score / length_norm))
                continue
            
            with torch.no_grad():
                outputs = model(seq)
                logits = outputs.logits[:, -1, :]
                log_probs = F.log_softmax(logits, dim=-1)
            
            # Get top-k tokens
            top_log_probs, top_indices = log_probs.topk(beam_width)
            
            for i in range(beam_width):
                token_id = top_indices[0, i].unsqueeze(0).unsqueeze(0)
                new_seq = torch.cat([seq, token_id], dim=-1)
                new_score = score + top_log_probs[0, i].item()
                
                # N-gram blocking to prevent repetition
                if no_repeat_ngram_size > 0:
                    tokens = new_seq[0].tolist()
                    ngram = tuple(tokens[-no_repeat_ngram_size:])
                    # Check if this n-gram appeared before
                    ngrams_seen = set()
                    for j in range(len(tokens) - no_repeat_ngram_size):
                        ng = tuple(tokens[j:j + no_repeat_ngram_size])
                        ngrams_seen.add(ng)
                    if ngram in ngrams_seen:
                        continue  # Skip this candidate
                
                all_candidates.append((new_seq, new_score))
        
        if not all_candidates:
            break
        
        # Keep top-k beams
        all_candidates.sort(key=lambda x: x[1], reverse=True)
        beams = all_candidates[:beam_width]
    
    # Return best sequence
    all_results = completed + beams
    all_results.sort(key=lambda x: x[1], reverse=True)
    best_seq = all_results[0][0]
    
    return tokenizer.decode(best_seq[0], skip_special_tokens=True)
```

### Length Penalty in Beam Search

Without length penalty, beam search favors shorter sequences (fewer multiplication of probabilities < 1):

$$\text{score}(Y) = \frac{\log P(Y|X)}{lp(Y)}$$

$$lp(Y) = \frac{(5 + |Y|)^\alpha}{(5 + 1)^\alpha}$$

Where $\alpha > 0$ penalizes short sequences (typical $\alpha = 0.6$-$1.0$).

---

## Sampling Methods

### 1. Pure Random Sampling

```python
def random_sample(logits, temperature=1.0):
    """
    Sample from the full distribution.
    
    Problem: Can sample very unlikely tokens → incoherent text
    """
    probs = F.softmax(logits / temperature, dim=-1)
    next_token = torch.multinomial(probs, num_samples=1)
    return next_token
```

### 2. Top-K Sampling

Only sample from the $k$ most likely tokens:

```python
def top_k_sampling(logits, k=50, temperature=1.0):
    """
    Top-K sampling: restrict to top K most likely tokens.
    
    K=1  → greedy decoding
    K=50 → sample from top 50 tokens (GPT-2 default)
    K=V  → pure random sampling (V = vocab size)
    
    Problem: Fixed K doesn't adapt to the distribution shape.
    When model is confident, K=50 includes garbage tokens.
    When model is uncertain, K=50 might miss good options.
    """
    # Zero out everything outside top-k
    top_k_logits, top_k_indices = logits.topk(k, dim=-1)
    
    # Create a mask: set non-top-k to -infinity
    filtered_logits = torch.full_like(logits, float('-inf'))
    filtered_logits.scatter_(-1, top_k_indices, top_k_logits)
    
    # Apply temperature and sample
    probs = F.softmax(filtered_logits / temperature, dim=-1)
    next_token = torch.multinomial(probs, num_samples=1)
    return next_token
```

**Visual:**
```
Full distribution (sorted):
█████████▇▇▇▆▆▅▅▄▄▃▃▃▂▂▂▂▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁

Top-K=10:
█████████▇                                    ← Only these are candidates
```

### 3. Top-P (Nucleus) Sampling

Sample from the smallest set of tokens whose cumulative probability exceeds $p$:

```python
def top_p_sampling(logits, p=0.9, temperature=1.0):
    """
    Nucleus (Top-P) sampling: dynamic vocabulary size.
    
    Instead of fixed K tokens, include tokens until their
    cumulative probability reaches P.
    
    P=0.9 → include tokens covering 90% of probability mass
    P=1.0 → pure random sampling
    P→0   → greedy decoding
    
    Advantage: Adapts to distribution shape!
    - Confident model → few tokens needed (like small K)
    - Uncertain model → many tokens included (like large K)
    """
    # Apply temperature
    scaled_logits = logits / temperature
    
    # Sort probabilities in descending order
    sorted_logits, sorted_indices = torch.sort(scaled_logits, descending=True)
    cumulative_probs = torch.cumsum(F.softmax(sorted_logits, dim=-1), dim=-1)
    
    # Find cutoff: remove tokens with cumulative prob > p
    # Shift right by 1 so we always keep at least one token
    sorted_indices_to_remove = cumulative_probs - F.softmax(sorted_logits, dim=-1) >= p
    sorted_logits[sorted_indices_to_remove] = float('-inf')
    
    # Unsort and sample
    logits.scatter_(-1, sorted_indices, sorted_logits)
    probs = F.softmax(logits, dim=-1)
    next_token = torch.multinomial(probs, num_samples=1)
    return next_token
```

**Visual — Top-P adapts dynamically:**
```
Confident prediction (Top-P=0.9 selects 3 tokens):
████████▇▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁
^^^^^^^^^^^  90% of mass here

Uncertain prediction (Top-P=0.9 selects 15 tokens):
███▇▇▆▆▅▅▄▄▃▃▃▂▁▁▁▁▁▁▁▁▁
^^^^^^^^^^^^^^^^^^^^^^^^^ 90% of mass needs more tokens
```

### 4. Min-P Sampling

A newer, simpler alternative — keep all tokens with probability ≥ min_p × max_probability:

```python
def min_p_sampling(logits, min_p=0.1, temperature=1.0):
    """
    Min-P sampling: keep tokens with prob >= min_p * max_prob.
    
    Simpler than top-p, adapts naturally.
    min_p=0.1 → keep tokens at least 10% as likely as the best token
    """
    probs = F.softmax(logits / temperature, dim=-1)
    max_prob = probs.max()
    
    # Mask tokens below threshold
    threshold = min_p * max_prob
    mask = probs < threshold
    logits[mask] = float('-inf')
    
    # Re-normalize and sample
    probs = F.softmax(logits / temperature, dim=-1)
    next_token = torch.multinomial(probs, num_samples=1)
    return next_token
```

### 5. Repetition Penalty

Penalize tokens that have already appeared in the generated text:

```python
def apply_repetition_penalty(logits, generated_ids, penalty=1.2):
    """
    Reduce probability of already-generated tokens.
    
    penalty=1.0 → no effect
    penalty>1.0 → discourage repetition (typical: 1.1-1.3)
    penalty<1.0 → encourage repetition (rarely used)
    """
    for token_id in set(generated_ids):
        if logits[0, token_id] > 0:
            logits[0, token_id] /= penalty
        else:
            logits[0, token_id] *= penalty
    return logits
```

### Comparison Table

| Strategy | Diversity | Quality | Speed | Best For |
|----------|----------|---------|-------|----------|
| Greedy | None (deterministic) | Medium | Fastest | Short factual answers |
| Beam Search | Low | High (fluent) | Slow | Translation, summarization |
| Top-K | Medium-High | Good | Fast | General generation |
| Top-P | High | Good | Fast | Creative writing, chatbots |
| Min-P | High | Good | Fast | Alternative to top-p |
| Temp only | Variable | Variable | Fast | Quick tuning |

> **Pro Tip**: In practice, most production systems use **Top-P (0.9) + Temperature (0.7-0.9) + Repetition Penalty (1.1)** as a solid default combination. For factual tasks, lower temperature (0.1-0.3). For creative tasks, higher temperature (0.8-1.2).

---

## Prompt Engineering

### What is Prompt Engineering

Prompt engineering is the art of crafting inputs to get the desired output from a language model. It's the primary way to control text generation without modifying the model.

### Key Techniques

#### 1. Zero-Shot Prompting

```python
# No examples, just instructions
prompt = """Classify the following review as POSITIVE or NEGATIVE.

Review: "The food was terrible and the service was slow."
Classification:"""
# Model outputs: "NEGATIVE"
```

#### 2. Few-Shot Prompting

```python
# Provide examples to establish the pattern
prompt = """Classify reviews as POSITIVE or NEGATIVE.

Review: "Amazing experience, will come back!"
Classification: POSITIVE

Review: "Worst meal I've ever had."
Classification: NEGATIVE

Review: "The ambiance was nice but food was mediocre."
Classification:"""
# Model outputs: "NEGATIVE"
```

#### 3. Chain-of-Thought (CoT) Prompting

```python
# Ask the model to reason step by step
prompt = """Solve this step by step:

If a store has 3 boxes, each containing 12 apples, and 15% of all apples are rotten, 
how many good apples are there?

Let's think step by step:"""
# Model outputs:
# Step 1: Total apples = 3 × 12 = 36
# Step 2: Rotten apples = 36 × 0.15 = 5.4, round to 5
# Step 3: Good apples = 36 - 5 = 31
```

#### 4. System Prompting

```python
# Set the model's persona/behavior
messages = [
    {"role": "system", "content": "You are a concise technical writer. "
     "Always respond in bullet points. Never use more than 50 words."},
    {"role": "user", "content": "Explain what Docker is."}
]
```

#### 5. Structured Output Prompting

```python
prompt = """Extract information from the text and return as JSON.

Text: "John Smith, age 35, works as a software engineer at Google in Mountain View."

Output the following JSON:
{
    "name": "",
    "age": 0,
    "occupation": "",
    "company": "",
    "location": ""
}"""
```

### Prompt Engineering Best Practices

| Practice | Example |
|----------|---------|
| Be specific | "Write a 3-sentence summary" not "Summarize" |
| Set the format | "Reply in JSON format with keys: name, age, role" |
| Give examples | Few-shot > zero-shot for complex tasks |
| Set constraints | "Use only information from the context provided" |
| Define persona | "You are an expert data scientist..." |
| Ask for reasoning | "Explain your reasoning before giving the answer" |
| Iterate | Test, observe failures, refine prompt |

---

## Controlling Generation

### Constrained Generation

```python
from transformers import pipeline, LogitsProcessor, LogitsProcessorList
import torch

class BanTokensProcessor(LogitsProcessor):
    """Ban specific tokens from being generated"""
    def __init__(self, banned_token_ids):
        self.banned_token_ids = banned_token_ids
    
    def __call__(self, input_ids, scores):
        scores[:, self.banned_token_ids] = float('-inf')
        return scores

class ForceTokenProcessor(LogitsProcessor):
    """Force generation of specific tokens at specific positions"""
    def __init__(self, force_map):
        # force_map: {position: token_id}
        self.force_map = force_map
    
    def __call__(self, input_ids, scores):
        cur_len = input_ids.shape[-1]
        if cur_len in self.force_map:
            scores[:, :] = float('-inf')
            scores[:, self.force_map[cur_len]] = 0
        return scores

# Usage
generator = pipeline("text-generation", model="gpt2")
tokenizer = generator.tokenizer

# Ban profanity tokens
banned_words = ["bad_word_1", "bad_word_2"]  # Replace with actual words to ban
banned_ids = [tokenizer.encode(w, add_special_tokens=False)[0] for w in banned_words if tokenizer.encode(w, add_special_tokens=False)]

processor_list = LogitsProcessorList([
    BanTokensProcessor(banned_ids)
])

# Generate with constraints
output = generator(
    "The future of AI is",
    max_length=50,
    logits_processor=processor_list
)
```

### Guided Generation with Grammar/JSON

```python
# Force model output to follow a specific format
# Using outlines library (pip install outlines)

import outlines

model = outlines.models.transformers("gpt2")

# Generate valid JSON only
schema = '''{
    "type": "object",
    "properties": {
        "name": {"type": "string"},
        "sentiment": {"type": "string", "enum": ["positive", "negative", "neutral"]},
        "confidence": {"type": "number", "minimum": 0, "maximum": 1}
    },
    "required": ["name", "sentiment", "confidence"]
}'''

generator = outlines.generate.json(model, schema)
result = generator("Analyze: 'The movie was fantastic!' ->")
print(result)
# {"name": "movie review", "sentiment": "positive", "confidence": 0.95}
```

### Stopping Criteria

```python
from transformers import StoppingCriteria, StoppingCriteriaList

class StopOnKeyword(StoppingCriteria):
    """Stop generation when a specific keyword is produced"""
    def __init__(self, stop_words, tokenizer):
        self.stop_words = stop_words
        self.tokenizer = tokenizer
    
    def __call__(self, input_ids, scores, **kwargs):
        generated_text = self.tokenizer.decode(input_ids[0], skip_special_tokens=True)
        for word in self.stop_words:
            if word in generated_text:
                return True
        return False

class StopAfterNewline(StoppingCriteria):
    """Stop after generating N newlines (useful for single-line outputs)"""
    def __init__(self, tokenizer, max_newlines=1):
        self.tokenizer = tokenizer
        self.max_newlines = max_newlines
    
    def __call__(self, input_ids, scores, **kwargs):
        text = self.tokenizer.decode(input_ids[0], skip_special_tokens=True)
        return text.count('\n') >= self.max_newlines

# Usage
from transformers import AutoModelForCausalLM, AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = AutoModelForCausalLM.from_pretrained("gpt2")

stop_criteria = StoppingCriteriaList([
    StopOnKeyword(["END", "###", "---"], tokenizer)
])

inputs = tokenizer("Write a haiku about coding:\n", return_tensors="pt")
outputs = model.generate(
    **inputs,
    max_length=100,
    stopping_criteria=stop_criteria,
    do_sample=True,
    temperature=0.8
)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

---

## Text Generation with HuggingFace

### Basic Text Generation

```python
from transformers import pipeline

# GPT-2 (small, good for learning)
generator = pipeline("text-generation", model="gpt2")

# Generate text
outputs = generator(
    "Artificial intelligence will",
    max_length=100,
    num_return_sequences=3,  # Generate 3 different completions
    temperature=0.8,
    top_p=0.9,
    do_sample=True,
    no_repeat_ngram_size=3,  # Prevent 3-gram repetition
    pad_token_id=generator.tokenizer.eos_token_id
)

for i, output in enumerate(outputs):
    print(f"\n--- Completion {i+1} ---")
    print(output["generated_text"])
```

### Using Different Models

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

# Load model and tokenizer
model_name = "meta-llama/Llama-2-7b-chat-hf"  # Requires access token
# Alternatives that don't need access:
# model_name = "microsoft/DialoGPT-medium"     # Conversational
# model_name = "EleutherAI/gpt-neo-1.3B"       # Open source GPT
# model_name = "mistralai/Mistral-7B-v0.1"     # Efficient 7B model

tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.float16,  # Half precision to save memory
    device_map="auto"           # Automatically use GPU if available
)

# Generate
prompt = "Explain quantum computing in simple terms:"
inputs = tokenizer(prompt, return_tensors="pt").to(model.device)

with torch.no_grad():
    outputs = model.generate(
        **inputs,
        max_new_tokens=200,        # Max NEW tokens (not total length)
        temperature=0.7,
        top_p=0.9,
        do_sample=True,
        repetition_penalty=1.15,
        no_repeat_ngram_size=3,
    )

# Decode only the new tokens (exclude the prompt)
generated = tokenizer.decode(outputs[0][inputs['input_ids'].shape[1]:], skip_special_tokens=True)
print(generated)
```

### Chat-Style Generation

```python
from transformers import pipeline

# Use a chat model
chatbot = pipeline(
    "text-generation",
    model="microsoft/DialoGPT-medium",
)

# Or with chat template
from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "HuggingFaceH4/zephyr-7b-beta"  # Chat-optimized model
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name, torch_dtype=torch.float16, device_map="auto")

# Format as chat
messages = [
    {"role": "system", "content": "You are a helpful coding assistant."},
    {"role": "user", "content": "Write a Python function to calculate fibonacci numbers."},
]

# Apply chat template
prompt = tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
inputs = tokenizer(prompt, return_tensors="pt").to(model.device)

outputs = model.generate(**inputs, max_new_tokens=300, temperature=0.3, do_sample=True)
response = tokenizer.decode(outputs[0][inputs['input_ids'].shape[1]:], skip_special_tokens=True)
print(response)
```

### Streaming Generation

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, TextIteratorStreamer
from threading import Thread

model_name = "gpt2"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name)

prompt = "The key to successful machine learning is"
inputs = tokenizer(prompt, return_tensors="pt")

# Create streamer
streamer = TextIteratorStreamer(tokenizer, skip_prompt=True, skip_special_tokens=True)

# Generate in a separate thread
generation_kwargs = dict(
    **inputs,
    max_new_tokens=100,
    do_sample=True,
    temperature=0.7,
    streamer=streamer
)

thread = Thread(target=model.generate, kwargs=generation_kwargs)
thread.start()

# Stream output token by token
print(prompt, end="", flush=True)
for new_text in streamer:
    print(new_text, end="", flush=True)
print()

thread.join()
```

### Batch Generation (Efficient)

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("gpt2")
tokenizer.pad_token = tokenizer.eos_token  # GPT-2 has no pad token by default
model = AutoModelForCausalLM.from_pretrained("gpt2")

# Multiple prompts at once
prompts = [
    "The future of healthcare is",
    "Machine learning can help",
    "The most important skill for",
]

# Tokenize with padding (pad on the LEFT for causal models!)
tokenizer.padding_side = "left"
inputs = tokenizer(prompts, return_tensors="pt", padding=True)

# Generate
outputs = model.generate(
    **inputs,
    max_new_tokens=50,
    do_sample=True,
    temperature=0.7,
    pad_token_id=tokenizer.eos_token_id
)

# Decode each
for i, output in enumerate(outputs):
    text = tokenizer.decode(output, skip_special_tokens=True)
    print(f"\nPrompt {i+1}: {text}")
```

> **Pro Tip**: For causal (left-to-right) models like GPT, always pad on the LEFT side. This ensures the actual content ends at the same position, which is what the model expects. Padding on the right would shift meaningful tokens into different positions.

---

## Building a Text Generation Pipeline

### Production-Ready Generator

```python
"""
Production text generation pipeline with safety, quality controls,
and multiple generation strategies.
"""
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch
from typing import List, Optional, Dict
from dataclasses import dataclass, field

@dataclass
class GenerationConfig:
    """Configuration for text generation"""
    max_new_tokens: int = 256
    temperature: float = 0.7
    top_p: float = 0.9
    top_k: int = 50
    repetition_penalty: float = 1.15
    no_repeat_ngram_size: int = 3
    do_sample: bool = True
    num_return_sequences: int = 1
    
    # Presets
    @classmethod
    def creative(cls):
        return cls(temperature=1.0, top_p=0.95, repetition_penalty=1.1)
    
    @classmethod
    def factual(cls):
        return cls(temperature=0.2, top_p=0.8, do_sample=False)
    
    @classmethod
    def balanced(cls):
        return cls(temperature=0.7, top_p=0.9, repetition_penalty=1.15)
    
    @classmethod
    def code(cls):
        return cls(temperature=0.3, top_p=0.85, no_repeat_ngram_size=0)


class TextGenerator:
    def __init__(self, model_name: str, device: str = "auto"):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForCausalLM.from_pretrained(
            model_name,
            torch_dtype=torch.float16 if torch.cuda.is_available() else torch.float32,
            device_map=device if device == "auto" else None
        )
        
        if device != "auto":
            self.model = self.model.to(device)
        
        # Set pad token if not present
        if self.tokenizer.pad_token is None:
            self.tokenizer.pad_token = self.tokenizer.eos_token
        
        self.tokenizer.padding_side = "left"
    
    def generate(
        self,
        prompt: str,
        config: Optional[GenerationConfig] = None,
        stop_strings: Optional[List[str]] = None,
        system_prompt: Optional[str] = None
    ) -> str:
        """Generate text with the given configuration"""
        if config is None:
            config = GenerationConfig.balanced()
        
        # Build full prompt
        full_prompt = prompt
        if system_prompt:
            full_prompt = f"{system_prompt}\n\n{prompt}"
        
        # Tokenize
        inputs = self.tokenizer(
            full_prompt,
            return_tensors="pt",
            truncation=True,
            max_length=2048 - config.max_new_tokens  # Leave room for generation
        ).to(self.model.device)
        
        # Generate
        with torch.no_grad():
            outputs = self.model.generate(
                **inputs,
                max_new_tokens=config.max_new_tokens,
                temperature=config.temperature if config.do_sample else 1.0,
                top_p=config.top_p if config.do_sample else 1.0,
                top_k=config.top_k if config.do_sample else 0,
                repetition_penalty=config.repetition_penalty,
                no_repeat_ngram_size=config.no_repeat_ngram_size,
                do_sample=config.do_sample,
                num_return_sequences=config.num_return_sequences,
                pad_token_id=self.tokenizer.eos_token_id,
            )
        
        # Decode only new tokens
        prompt_length = inputs["input_ids"].shape[1]
        generated_text = self.tokenizer.decode(
            outputs[0][prompt_length:], 
            skip_special_tokens=True
        )
        
        # Apply stop strings
        if stop_strings:
            for stop in stop_strings:
                if stop in generated_text:
                    generated_text = generated_text[:generated_text.index(stop)]
        
        return generated_text.strip()
    
    def generate_batch(
        self,
        prompts: List[str],
        config: Optional[GenerationConfig] = None
    ) -> List[str]:
        """Generate text for multiple prompts efficiently"""
        if config is None:
            config = GenerationConfig.balanced()
        
        inputs = self.tokenizer(
            prompts,
            return_tensors="pt",
            padding=True,
            truncation=True,
            max_length=2048 - config.max_new_tokens
        ).to(self.model.device)
        
        with torch.no_grad():
            outputs = self.model.generate(
                **inputs,
                max_new_tokens=config.max_new_tokens,
                temperature=config.temperature if config.do_sample else 1.0,
                top_p=config.top_p if config.do_sample else 1.0,
                do_sample=config.do_sample,
                repetition_penalty=config.repetition_penalty,
                pad_token_id=self.tokenizer.eos_token_id,
            )
        
        results = []
        for i, output in enumerate(outputs):
            text = self.tokenizer.decode(output, skip_special_tokens=True)
            # Remove the original prompt
            text = text[len(prompts[i]):].strip()
            results.append(text)
        
        return results


# Usage
gen = TextGenerator("gpt2")

# Creative writing
story = gen.generate(
    "Write a short story about a robot learning to paint:",
    config=GenerationConfig.creative()
)
print(f"Story:\n{story}")

# Factual answer
answer = gen.generate(
    "Explain what photosynthesis is in one paragraph:",
    config=GenerationConfig.factual()
)
print(f"\nAnswer:\n{answer}")

# Code generation
code = gen.generate(
    "# Python function to merge two sorted lists\ndef merge_sorted(",
    config=GenerationConfig.code(),
    stop_strings=["\n\n", "# "]
)
print(f"\nCode:\n{code}")
```

---

## Evaluation of Generated Text

### Automatic Metrics

#### Perplexity

How "surprised" the model is by the text — lower is better:

$$PPL(W) = \exp\left(-\frac{1}{N}\sum_{i=1}^{N} \log P(w_i | w_1, ..., w_{i-1})\right)$$

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

def calculate_perplexity(text, model, tokenizer):
    """
    Calculate perplexity of text under the given model.
    Lower = more likely/fluent under this model.
    
    Typical values:
    - Well-written English: 15-30
    - Casual text: 30-80
    - Random/nonsense: 1000+
    """
    inputs = tokenizer(text, return_tensors="pt")
    input_ids = inputs["input_ids"].to(model.device)
    
    with torch.no_grad():
        outputs = model(input_ids, labels=input_ids)
        loss = outputs.loss  # Cross-entropy loss
    
    perplexity = torch.exp(loss).item()
    return perplexity

model = AutoModelForCausalLM.from_pretrained("gpt2")
tokenizer = AutoTokenizer.from_pretrained("gpt2")

texts = [
    "The quick brown fox jumps over the lazy dog.",          # Natural
    "Dog lazy the over jumps fox brown quick the.",          # Shuffled
    "Colorless green ideas sleep furiously.",                 # Grammatical nonsense
]

for text in texts:
    ppl = calculate_perplexity(text, model, tokenizer)
    print(f"PPL={ppl:8.1f} | {text}")

# Output:
# PPL=    45.2 | The quick brown fox jumps over the lazy dog.
# PPL=  1823.5 | Dog lazy the over jumps fox brown quick the.
# PPL=   285.7 | Colorless green ideas sleep furiously.
```

#### BLEU Score (for reference-based tasks)

```python
from nltk.translate.bleu_score import sentence_bleu, corpus_bleu

# Single sentence BLEU
reference = [["the", "cat", "is", "on", "the", "mat"]]
hypothesis = ["the", "cat", "sat", "on", "the", "mat"]

bleu = sentence_bleu(reference, hypothesis)
print(f"BLEU: {bleu:.4f}")  # 0.6687

# BLEU with different n-gram weights
bleu_1 = sentence_bleu(reference, hypothesis, weights=(1, 0, 0, 0))  # Unigram
bleu_2 = sentence_bleu(reference, hypothesis, weights=(0.5, 0.5, 0, 0))  # Up to bigram
print(f"BLEU-1: {bleu_1:.4f}, BLEU-2: {bleu_2:.4f}")
```

#### METEOR, BERTScore, ROUGE

```python
import evaluate

# METEOR (better correlation with human judgment than BLEU)
meteor = evaluate.load("meteor")
result = meteor.compute(
    predictions=["The cat sat on the mat"],
    references=["The cat is on the mat"]
)
print(f"METEOR: {result['meteor']:.4f}")

# BERTScore (semantic similarity using BERT embeddings)
bertscore = evaluate.load("bertscore")
result = bertscore.compute(
    predictions=["The cat sat on the mat"],
    references=["The cat is on the mat"],
    lang="en"
)
print(f"BERTScore F1: {result['f1'][0]:.4f}")
```

### Human Evaluation Criteria

| Criterion | Description | Scale |
|-----------|-------------|-------|
| **Fluency** | Is the text grammatically correct and natural? | 1-5 |
| **Coherence** | Does the text make logical sense? | 1-5 |
| **Relevance** | Is the text on-topic and addressing the prompt? | 1-5 |
| **Informativeness** | Does the text provide useful information? | 1-5 |
| **Harmlessness** | Is the text free from harmful/toxic content? | Binary |
| **Faithfulness** | Is the text factually accurate? | 1-5 |

### LLM-as-Judge

```python
def llm_evaluate(generated_text, criteria, judge_model):
    """
    Use a stronger LLM to evaluate generated text.
    Common in research and production evaluations.
    """
    prompt = f"""Evaluate the following text on a scale of 1-5 for each criterion.
Return only a JSON object.

Text: "{generated_text}"

Criteria:
- Fluency: Is it grammatically correct and natural?
- Coherence: Does it make logical sense?
- Relevance: Is it on-topic?
- Informativeness: Does it provide useful information?

JSON output:"""
    
    # Use judge model to evaluate
    evaluation = judge_model(prompt)
    return evaluation
```

---

## Common Mistakes

### 1. Temperature = 0 vs do_sample = False

```python
# WRONG — temperature=0 can cause division by zero in some implementations
output = model.generate(temperature=0, do_sample=True)

# RIGHT — for deterministic output, disable sampling
output = model.generate(do_sample=False)  # Greedy decoding
# OR
output = model.generate(do_sample=False, num_beams=5)  # Beam search
```

### 2. Forgetting to Set pad_token

```python
# WRONG — GPT-2 and many causal models have no pad token
tokenizer = AutoTokenizer.from_pretrained("gpt2")
inputs = tokenizer(["short", "a much longer prompt"], padding=True)
# Error or silent bug!

# RIGHT — set pad token
tokenizer.pad_token = tokenizer.eos_token
# AND pad on the LEFT for causal models
tokenizer.padding_side = "left"
```

### 3. Using max_length Instead of max_new_tokens

```python
# CONFUSING — max_length includes the prompt
output = model.generate(max_length=100)
# If prompt is 80 tokens, you only get 20 new tokens!

# CLEAR — max_new_tokens is just the generated part
output = model.generate(max_new_tokens=100)
# Always generates up to 100 new tokens regardless of prompt length
```

### 4. Not Handling Repetition

```python
# WRONG — model enters a repetition loop
# "The cat sat on the mat. The cat sat on the mat. The cat sat..."

# RIGHT — use multiple strategies together
output = model.generate(
    repetition_penalty=1.2,          # Penalize repeated tokens
    no_repeat_ngram_size=3,          # Block repeated 3-grams
    encoder_no_repeat_ngram_size=3,  # For encoder-decoder models
)
```

### 5. Ignoring Memory Constraints

```python
# WRONG — loading a 7B model in full precision on a consumer GPU
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-7b-hf")
# OOM error! 7B × 4 bytes = 28GB

# RIGHT — use quantization or half precision
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    torch_dtype=torch.float16,           # 7B × 2 bytes = 14GB
    # OR
    load_in_8bit=True,                   # 7B × 1 byte = 7GB
    # OR
    load_in_4bit=True,                   # 7B × 0.5 bytes = 3.5GB
    device_map="auto"
)
```

### 6. Not Stripping Prompt from Output

```python
# WRONG — returning the full output including the prompt
output = model.generate(**inputs, max_new_tokens=50)
text = tokenizer.decode(output[0])
# Returns: "My prompt text here... PLUS the generated continuation"

# RIGHT — only decode the new tokens
prompt_length = inputs["input_ids"].shape[1]
generated_only = output[0][prompt_length:]
text = tokenizer.decode(generated_only, skip_special_tokens=True)
```

### 7. Applying Temperature AND Top-K/Top-P Without Sampling

```python
# WRONG — temperature/top_p/top_k have no effect without sampling!
output = model.generate(
    do_sample=False,    # Greedy/beam search
    temperature=0.8,    # IGNORED!
    top_p=0.9,          # IGNORED!
)

# RIGHT — enable sampling to use these parameters
output = model.generate(
    do_sample=True,
    temperature=0.8,
    top_p=0.9,
)
```

---

## Interview Questions

### Conceptual

**Q1: Explain the difference between greedy decoding, beam search, and nucleus sampling.**
> **Greedy** picks the highest probability token at each step — fast but repetitive and suboptimal. **Beam search** maintains K best partial sequences, finding globally better results but tending toward generic outputs. **Nucleus (top-p) sampling** randomly samples from tokens covering the top P% of probability mass, producing diverse and natural text. Greedy/beam are deterministic and suit factual tasks; sampling suits creative/conversational tasks.

**Q2: What is temperature in text generation and how does it affect output?**
> Temperature scales the logits before softmax: $P(w) \propto \exp(z / \tau)$. Low temperature ($\tau \to 0$) sharpens the distribution, making the model more confident and deterministic. High temperature ($\tau > 1$) flattens it, giving unlikely tokens more chance and increasing diversity/randomness. Temperature = 1.0 is the model's natural distribution. It doesn't change which token is most likely, only how concentrated the probability is.

**Q3: Why does beam search tend to produce generic/boring text?**
> Beam search optimizes for the highest total probability sequence. Common, safe phrases ("I don't know", "It is important to note that...") have high probability individually at each step, so they dominate. Rare but interesting phrasings get pruned early because they have lower per-step probability. This is called the "bland text" problem. Sampling-based methods avoid this by introducing randomness.

**Q4: What is the difference between top-k and top-p sampling? When would you prefer one over the other?**
> Top-k always considers exactly K tokens regardless of probability distribution. Top-p dynamically selects the smallest set of tokens whose cumulative probability reaches P. Top-p is generally preferred because it adapts: when the model is confident, it considers few tokens (like small K); when uncertain, it considers many (like large K). Top-k can include garbage tokens when the model is confident, or miss good options when it's uncertain.

**Q5: How does repetition penalty work? What are the alternatives?**
> Repetition penalty divides the logit of previously generated tokens by a factor > 1 (if logit is positive) or multiplies (if negative), making them less likely. Alternatives: (1) no_repeat_ngram_size blocks exact n-gram repetitions, (2) frequency penalty that increases with each occurrence, (3) presence penalty that's binary (appeared or not), (4) exponential decay penalty for tokens based on distance from last occurrence.

### Practical

**Q6: You're building a customer-facing chatbot. What generation parameters would you use and why?**
> Temperature 0.3-0.5 (mostly deterministic but slightly varied), top-p 0.85 (safe vocabulary), repetition_penalty 1.15 (avoid loops), max_new_tokens 256 (concise answers). Use do_sample=True for natural variation between identical queries. Add stop strings for turn-taking markers. Use a safety filter on outputs. For factual questions, lower temperature further. Monitor and log generation metrics in production.

**Q7: How would you reduce hallucination in a text generation system?**
> 1. Ground generation in retrieved context (RAG)
> 2. Lower temperature for factual tasks
> 3. Add explicit instructions: "Only state facts from the provided context"
> 4. Post-generation fact-checking with NLI models
> 5. Use constrained decoding to limit outputs to known-valid tokens
> 6. Fine-tune on high-quality, verified data
> 7. Implement confidence scoring and abstain when uncertain
> 8. Use self-consistency: generate multiple times and check agreement

**Q8: How do you serve a large language model (7B+ parameters) efficiently in production?**
> 1. Quantization (4-bit/8-bit) to reduce memory 4-8x
> 2. KV-cache optimization for faster autoregressive generation
> 3. Continuous batching (vLLM, TGI) to maximize GPU utilization
> 4. Speculative decoding with a smaller draft model
> 5. PagedAttention (vLLM) for efficient memory management
> 6. Tensor parallelism across multiple GPUs
> 7. ONNX/TensorRT export for inference optimization
> 8. Streaming responses to reduce perceived latency

---

## Quick Reference

### Parameter Cheat Sheet

| Parameter | Range | Low Value Effect | High Value Effect | Default |
|-----------|-------|-----------------|-------------------|---------|
| `temperature` | 0.0 - 2.0 | Deterministic, focused | Random, creative | 1.0 |
| `top_p` | 0.0 - 1.0 | Very few token candidates | Many candidates | 1.0 |
| `top_k` | 1 - vocab_size | Few candidates (greedy at 1) | Many candidates | 50 |
| `repetition_penalty` | 1.0 - 2.0 | No penalty | Strong anti-repetition | 1.0 |
| `max_new_tokens` | 1 - model_max | Short output | Long output | - |
| `num_beams` | 1 - 10+ | Greedy (1) | Better but slower | 1 |
| `no_repeat_ngram_size` | 0 - 5 | No blocking | Block repeated phrases | 0 |
| `length_penalty` | 0.0 - 2.0 | Shorter outputs | Longer outputs | 1.0 |

### Recommended Presets

| Use Case | Temperature | Top-P | Rep. Penalty | Sampling |
|----------|------------|-------|-------------|----------|
| **Factual QA** | 0.1-0.3 | 0.8 | 1.0 | Off (greedy) |
| **Chatbot** | 0.5-0.7 | 0.9 | 1.15 | On |
| **Creative Writing** | 0.8-1.2 | 0.95 | 1.1 | On |
| **Code Generation** | 0.2-0.4 | 0.85 | 1.0 | On |
| **Translation** | 0.1 | 1.0 | 1.0 | Off (beam=5) |
| **Summarization** | 0.3 | 0.9 | 1.2 | Off (beam=4) |
| **Brainstorming** | 1.0-1.5 | 0.95 | 1.0 | On |

### Model Size Guide

| Model Size | VRAM (FP16) | VRAM (4-bit) | Quality | Speed |
|-----------|------------|-------------|---------|-------|
| 1.5B (GPT-2 XL) | 3 GB | 1 GB | Basic | Very fast |
| 7B (Llama-2, Mistral) | 14 GB | 4 GB | Good | Fast |
| 13B (Llama-2) | 26 GB | 7 GB | Better | Medium |
| 34B (CodeLlama) | 68 GB | 18 GB | Great | Slow |
| 70B (Llama-2) | 140 GB | 35 GB | Excellent | Very slow |

### Serving Frameworks

| Framework | Strengths | Best For |
|-----------|-----------|----------|
| **vLLM** | PagedAttention, continuous batching | High-throughput serving |
| **TGI** (HuggingFace) | Easy deployment, streaming | HuggingFace integration |
| **Ollama** | Local, easy setup | Development, small models |
| **llama.cpp** | CPU inference, GGUF format | Edge/local deployment |
| **TensorRT-LLM** | NVIDIA optimization | Maximum GPU performance |

### Decoding Strategy Decision Tree

```
Need deterministic output?
├── YES → do_sample=False
│   ├── Best quality needed? → Beam search (num_beams=4-5)
│   └── Speed matters? → Greedy (num_beams=1)
│
└── NO → do_sample=True
    ├── Factual task? → temperature=0.3, top_p=0.85
    ├── Conversational? → temperature=0.7, top_p=0.9
    └── Creative? → temperature=1.0, top_p=0.95
```

---

## Summary

Text generation is the backbone of modern AI applications. The key concepts are:

1. **Autoregressive generation** — one token at a time, conditioned on all previous tokens
2. **Temperature** — controls randomness (low = focused, high = creative)
3. **Decoding strategies** — greedy (fast/repetitive), beam search (quality/generic), sampling (diverse/natural)
4. **Top-p/Top-k** — filter unlikely tokens before sampling
5. **Prompt engineering** — the primary interface for controlling LLM behavior
6. **Repetition control** — penalties and n-gram blocking prevent degenerate loops
7. **Evaluation** — perplexity for fluency, BLEU/ROUGE for reference-based, human eval for quality

The right generation parameters depend entirely on your use case. There is no universally "best" setting — factual tasks need low temperature and constrained decoding, while creative tasks benefit from higher temperature and sampling.
