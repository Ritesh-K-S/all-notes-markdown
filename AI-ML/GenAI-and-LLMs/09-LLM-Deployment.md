# Chapter 09: LLM Deployment — From Model to Production

## Table of Contents
- [What is LLM Deployment](#what-is-llm-deployment)
- [Why Deployment is Hard](#why-deployment-is-hard)
- [Model Quantization](#model-quantization)
- [Inference Optimization](#inference-optimization)
- [Serving Frameworks — vLLM and TGI](#serving-frameworks--vllm-and-tgi)
- [KV Cache and Memory Management](#kv-cache-and-memory-management)
- [Batching Strategies](#batching-strategies)
- [Deployment Architectures](#deployment-architectures)
- [Code Examples](#code-examples)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is LLM Deployment

### Simple Explanation (Like Explaining to a 15-Year-Old)

You've trained a brilliant AI model on a powerful supercomputer. Now you want millions of people to use it — like ChatGPT. The problem? Each user query needs the model to "think" (run computations), and that's incredibly expensive and slow.

**LLM Deployment** is about solving this puzzle:
- How do you make a 70-billion-parameter model respond in under 2 seconds?
- How do you serve 1000 users simultaneously without needing $1M/day in GPU costs?
- How do you shrink a model that needs 140GB of memory to fit on a single $1000 GPU?

Think of it like this: you have a brilliant chef (the model) who cooks amazing meals but takes 30 minutes per dish. Deployment is about opening a restaurant — you need to serve hundreds of customers efficiently without the food quality dropping.

### The Deployment Challenge in Numbers

```
LLaMA-2 70B Model:
─────────────────────────────────
• Parameters: 70 billion
• FP16 memory: ~140 GB (need 2× A100 80GB GPUs)
• Single token generation: ~50ms
• 500-token response: ~25 seconds (way too slow!)
• Cost per A100/hour: ~$2-4

After Optimization:
─────────────────────────────────
• INT4 quantized: ~35 GB (fits on 1× A100)
• With vLLM batching: ~5ms per token per user
• 500-token response: ~2.5 seconds ✓
• Can serve 50+ concurrent users
• 10× cost reduction
```

---

## Why Deployment is Hard

### The Three Constraints Triangle

```
          ┌──────────┐
         /            \
        /   QUALITY    \
       /   (accuracy)   \
      /                  \
     /────────────────────\
    /                      \
   ┌──────────┐    ┌──────────┐
   │  SPEED   │────│   COST   │
   │ (latency)│    │  ($/req) │
   └──────────┘    └──────────┘

You can optimize for 2, but the 3rd suffers.
Deployment is about finding the right balance.
```

### Key Metrics

| Metric | Definition | Target (Production) |
|---|---|---|
| **TTFT** | Time to First Token | < 500ms |
| **TPS** | Tokens Per Second (per user) | 30-50 TPS |
| **Throughput** | Total tokens/sec across all users | Maximize |
| **Latency (P99)** | 99th percentile response time | < 5 seconds |
| **Memory** | GPU VRAM usage | Fit within available GPUs |
| **Cost** | $/1M tokens | $0.10-$2.00 (depends on model) |

### Why LLMs Are Uniquely Hard to Deploy

1. **Autoregressive generation**: Each token depends on ALL previous tokens — can't parallelize across tokens
2. **Massive memory**: 7B model in FP16 = 14GB, just for weights (no activations, no KV cache)
3. **Memory-bound**: GPU computes faster than it can read weights from memory
4. **Variable-length I/O**: Inputs and outputs vary wildly in length
5. **KV cache explosion**: Each token adds to a growing cache that can dwarf model weights

---

## Model Quantization

### What is Quantization?

Reducing the numerical precision of model weights from high-precision (FP32/FP16) to lower precision (INT8/INT4) to reduce memory and increase speed.

### Analogy

Think of a photograph:
- **FP32** = RAW image (huge file, maximum detail)
- **FP16** = High-quality JPEG (half the size, nearly identical quality)
- **INT8** = Standard JPEG (quarter the size, very good quality)
- **INT4** = Compressed JPEG (⅛ the size, still looks great to most people)

### Number Format Comparison

```
FP32 (32 bits):  [1 sign][8 exponent][23 mantissa]
FP16 (16 bits):  [1 sign][5 exponent][10 mantissa]
BF16 (16 bits):  [1 sign][8 exponent][ 7 mantissa]  ← Same range as FP32!
INT8 ( 8 bits):  [1 sign][7 integer]                  ← Values: -128 to 127
INT4 ( 4 bits):  [1 sign][3 integer]                  ← Values: -8 to 7
```

### Memory Impact

| Format | Bits/Param | 7B Model | 13B Model | 70B Model |
|---|---|---|---|---|
| FP32 | 32 | 28 GB | 52 GB | 280 GB |
| FP16/BF16 | 16 | 14 GB | 26 GB | 140 GB |
| INT8 | 8 | 7 GB | 13 GB | 70 GB |
| INT4 | 4 | 3.5 GB | 6.5 GB | 35 GB |
| GPTQ 4-bit | ~4.5 | 4 GB | 7.5 GB | 40 GB |

### Quantization Methods

#### Post-Training Quantization (PTQ) — No Retraining

| Method | Bits | Speed | Quality | Key Idea |
|---|---|---|---|---|
| **LLM.int8()** | 8 | Fast | Excellent | Mixed-precision: outliers stay FP16 |
| **GPTQ** | 4/3/2 | Moderate | Very Good | Layer-wise optimal rounding using calibration data |
| **AWQ** | 4 | Fast | Very Good | Protect salient weight channels (1% of weights matter most) |
| **GGUF** | 2-8 | Fast | Good-Great | llama.cpp format, CPU-friendly, mixed precision |
| **SqueezeLLM** | 3-4 | Moderate | Good | Non-uniform quantization |
| **QuIP#** | 2-4 | Moderate | Good | Incoherence processing for extreme compression |

#### Quantization-Aware Training (QAT) — Retrain with Low Precision

| Method | Description | When to Use |
|---|---|---|
| QLoRA | 4-bit base + LoRA adapters in FP16 | Fine-tuning on consumer GPUs |
| QLORA+GPTQ | GPTQ quantized model + LoRA | Best of both worlds |

### How GPTQ Works (Simplified)

```
For each layer in the network:
─────────────────────────────────
1. Collect calibration data (128 random samples from C4/WikiText)
2. For each column of weight matrix:
   a. Round weights to nearest INT4 value
   b. Measure error introduced by rounding
   c. Distribute error to remaining columns (optimal error correction)
3. Result: Quantized weights with minimized accuracy loss

Key insight: Not all weights are equally important.
GPTQ focuses on minimizing output error, not weight error.
```

### How AWQ Works (Simplified)

```
Observation: Only ~1% of weight channels significantly affect output.
These "salient channels" correspond to large activation magnitudes.

Algorithm:
1. Run calibration data, measure activation magnitudes per channel
2. Identify top 1% salient channels
3. Scale these channels UP before quantization (preserves their precision)
4. Scale corresponding activations DOWN (mathematically equivalent)
5. Quantize all weights to INT4
6. Result: Salient channels effectively have higher precision

Why better than GPTQ:
• No layer-wise reconstruction (faster calibration)
• Hardware-friendly (no mixed-precision compute)
• Better quality at same bit-width
```

### Quality Comparison (Perplexity on WikiText-2, Lower = Better)

```
LLaMA-2 7B:
─────────────────────────────────
FP16:           5.47  (baseline)
GPTQ 4-bit:     5.63  (+0.16)
AWQ 4-bit:      5.60  (+0.13)
GGUF Q4_K_M:    5.65  (+0.18)
GPTQ 3-bit:     6.29  (+0.82)  ← Noticeable quality drop
GPTQ 2-bit:     8.15  (+2.68)  ← Significant degradation
```

> **Pro Tip**: 4-bit quantization is the sweet spot — it gives 4× memory savings with <1% quality loss. Going below 4-bit causes noticeable degradation. AWQ is generally preferred over GPTQ for production deployment.

---

## Inference Optimization

### The Two Phases of LLM Inference

```
┌────────────────────────────────────────────────────────────────┐
│                    LLM INFERENCE PHASES                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PHASE 1: PREFILL (Prompt Processing)                          │
│  ─────────────────────────────────────                          │
│  • Process ALL input tokens in PARALLEL                         │
│  • Compute-bound (lots of matrix multiplication)               │
│  • Builds the initial KV cache                                  │
│  • Determines TTFT (Time to First Token)                       │
│  • Example: 500-token prompt → one forward pass                │
│                                                                 │
│  PHASE 2: DECODE (Token Generation)                             │
│  ──────────────────────────────────                              │
│  • Generate tokens ONE AT A TIME                                │
│  • Memory-bound (reads entire model weights per token)         │
│  • Each token: read KV cache + model weights → 1 new token    │
│  • Determines TPS (Tokens Per Second)                          │
│  • Example: 500-token response → 500 sequential forward passes │
│                                                                 │
│  WHY IT MATTERS:                                                │
│  Prefill: GPU utilization ~80% (compute-bound, efficient)      │
│  Decode:  GPU utilization ~5%  (memory-bound, wasteful!)       │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### Key Optimization Techniques

#### 1. FlashAttention

Standard attention is $O(N^2)$ in memory because it materializes the full $N \times N$ attention matrix.

**FlashAttention** computes attention in tiles, never materializing the full matrix:

```
Standard Attention:
Q, K, V (in HBM) → S = QK^T (write to HBM) → P = softmax(S) (write to HBM) → O = PV
Memory: O(N²)   Slow due to HBM reads/writes

FlashAttention:
Q, K, V (in HBM) → Load tiles to SRAM → Compute attention in SRAM → O
Memory: O(N)     Fast: minimizes HBM access

Speed improvement: 2-4× faster
Memory improvement: O(N) instead of O(N²)
```

#### 2. Speculative Decoding

Use a small, fast "draft" model to propose multiple tokens, then verify them with the large model in parallel:

```
Without Speculative Decoding:
Large model: [tok1] → [tok2] → [tok3] → [tok4] → [tok5]
             50ms    50ms     50ms     50ms     50ms = 250ms

With Speculative Decoding:
Draft model:  [tok1, tok2, tok3, tok4, tok5]  → 25ms (small, fast)
Large model:  Verify all 5 in parallel         → 55ms (one forward pass)
Total: 80ms (3× faster!)

Acceptance rate typically: 70-85%
If a draft token is rejected, regenerate from there.
```

#### 3. Tensor Parallelism (Multi-GPU)

Split each layer's weight matrices across multiple GPUs:

```
Tensor Parallelism (TP=4):
─────────────────────────────────
GPU 0: First quarter of each layer's weights
GPU 1: Second quarter
GPU 2: Third quarter  
GPU 3: Fourth quarter

Each GPU computes partial results → All-reduce → Continue

When to use:
• TP=2: Model doesn't fit on 1 GPU
• TP=4: Need lower latency (more GPUs = faster per request)
• TP=8: Very large models (70B+)

Limitation: All-reduce adds ~1ms overhead per layer
Best within a single node (NVLink is fast)
```

#### 4. Pipeline Parallelism

Split layers across GPUs sequentially:

```
Pipeline Parallelism (PP=4):
─────────────────────────────────
GPU 0: Layers 0-7     → GPU 1: Layers 8-15
                              → GPU 2: Layers 16-23
                                     → GPU 3: Layers 24-31

Better for: throughput (micro-batching fills the pipeline)
Worse for: latency (each request goes through ALL GPUs serially)
```

#### 5. Prefix Caching

Cache KV values for common prefixes (system prompts):

```
Without Prefix Caching:
User 1: [System prompt (500 tokens)] + [User question (50 tokens)]
User 2: [System prompt (500 tokens)] + [User question (30 tokens)]
→ Processes system prompt twice (1000 tokens of wasted compute!)

With Prefix Caching:
Cache: [System prompt KV cache (computed once)]
User 1: [Cached KV] + [User question (50 tokens)]
User 2: [Cached KV] + [User question (30 tokens)]
→ Saves 500 tokens of compute per request
```

---

## Serving Frameworks — vLLM and TGI

### vLLM (Recommended for Most Use Cases)

**Key Innovation**: PagedAttention — manages KV cache like an operating system manages virtual memory.

```
Traditional KV Cache Management:
─────────────────────────────────
Pre-allocate MAX possible KV cache per request
Short response: ████░░░░░░░░  (75% wasted!)
Long response:  ████████████  (fully used)

vLLM PagedAttention:
─────────────────────────────────
Allocate KV cache in small PAGES (like virtual memory)
Short response: ████  (no waste)
Long response:  ████████████  (pages allocated on demand)
Non-contiguous storage → flexible memory management
```

**vLLM Features:**
- PagedAttention: ~4× better memory utilization
- Continuous batching: dynamically add/remove requests
- Tensor parallelism: multi-GPU support
- Prefix caching: share common prefixes
- Quantization: GPTQ, AWQ, FP8 support
- OpenAI-compatible API server
- Speculative decoding support

### Text Generation Inference (TGI) — by Hugging Face

**Features:**
- Continuous batching with token streaming
- FlashAttention and Paged Attention
- Tensor parallelism
- Quantization (GPTQ, AWQ, EETQ, bitsandbytes)
- Built-in watermarking for generated text
- Production-grade: used by Hugging Face Inference Endpoints

### Comparison

| Feature | vLLM | TGI | Ollama | llama.cpp |
|---|---|---|---|---|
| Best for | Production GPU serving | HF integration | Local/dev | CPU/Edge |
| PagedAttention | Yes | Yes | No | No |
| Continuous batching | Yes | Yes | No | No |
| GPU support | Excellent | Excellent | Good | Optional |
| CPU support | No | Limited | Yes | Excellent |
| Quantization | GPTQ, AWQ, FP8 | GPTQ, AWQ, EETQ | GGUF | GGUF |
| API | OpenAI-compatible | Custom (+ OpenAI) | OpenAI-compatible | Varies |
| Multi-GPU | TP, PP | TP | Limited | Limited |
| Ease of setup | Easy | Docker-based | Very Easy | Moderate |
| Throughput | Highest | Very High | Moderate | Lower |

---

## KV Cache and Memory Management

### What is the KV Cache?

During autoregressive generation, each new token needs to attend to ALL previous tokens. Without caching, we'd recompute K and V for the entire sequence at every step.

```
Without KV Cache (naive):
─────────────────────────────────
Step 1: Compute K,V for [token1]                    → 1 computation
Step 2: Compute K,V for [token1, token2]            → 2 computations
Step 3: Compute K,V for [token1, token2, token3]    → 3 computations
...
Step N: Compute K,V for [token1, ..., tokenN]       → N computations
Total: N(N+1)/2 = O(N²) computations

With KV Cache:
─────────────────────────────────
Step 1: Compute K,V for [token1], CACHE it          → 1 computation
Step 2: Compute K,V for [token2], APPEND to cache   → 1 computation
Step 3: Compute K,V for [token3], APPEND to cache   → 1 computation
...
Step N: Compute K,V for [tokenN], APPEND to cache   → 1 computation
Total: N computations = O(N)
```

### KV Cache Memory Formula

$$\text{KV Cache per token} = 2 \times n_{layers} \times n_{heads} \times d_{head} \times \text{precision\_bytes}$$

| Model | Layers | Heads | d_head | KV per Token (FP16) | 2048 Tokens |
|---|---|---|---|---|---|
| LLaMA-2 7B | 32 | 32 | 128 | 512 KB | 1 GB |
| LLaMA-2 13B | 40 | 40 | 128 | 800 KB | 1.6 GB |
| LLaMA-2 70B | 80 | 64 | 128 | 2.5 MB | 5 GB |
| Mixtral 8x7B | 32 | 32 | 128 | 512 KB | 1 GB |

> **Key Insight**: For long contexts (32K+ tokens), the KV cache can be LARGER than the model weights! This is why techniques like GQA, MQA, and paged attention are critical.

### Grouped-Query Attention (GQA) and Multi-Query Attention (MQA)

```
Multi-Head Attention (MHA): Standard, each head has own K, V
─────────────────────────────────
Heads:  Q₁ Q₂ Q₃ Q₄ Q₅ Q₆ Q₇ Q₈
        K₁ K₂ K₃ K₄ K₅ K₆ K₇ K₈   ← 8 unique K,V pairs
        V₁ V₂ V₃ V₄ V₅ V₆ V₇ V₈
KV Cache: 8× per layer

Grouped-Query Attention (GQA): Groups share K, V
─────────────────────────────────
Heads:  Q₁ Q₂ Q₃ Q₄ Q₅ Q₆ Q₇ Q₈
        K₁ K₁ K₂ K₂ K₃ K₃ K₄ K₄   ← 4 unique K,V pairs
        V₁ V₁ V₂ V₂ V₃ V₃ V₄ V₄     (groups of 2 share)
KV Cache: 4× per layer (50% reduction)

Multi-Query Attention (MQA): ALL heads share one K, V
─────────────────────────────────
Heads:  Q₁ Q₂ Q₃ Q₄ Q₅ Q₆ Q₇ Q₈
        K₁ K₁ K₁ K₁ K₁ K₁ K₁ K₁   ← 1 unique K,V pair
        V₁ V₁ V₁ V₁ V₁ V₁ V₁ V₁
KV Cache: 1× per layer (87.5% reduction!)
```

- LLaMA-2 70B uses **GQA** (8 KV groups for 64 query heads = 8:1 ratio)
- Falcon uses **MQA** (1 KV group)
- Most modern models (Mistral, Gemma, Qwen) use **GQA**

---

## Batching Strategies

### Static Batching (Naive)

```
Request 1: "Hello"        → Generates 10 tokens  ████████████░░░░░
Request 2: "Tell me..."   → Generates 50 tokens  ██████████████████████████████████████████████████████
Request 3: "Hi"           → Generates 5 tokens   █████░░░░░░░░░░░░

Problem: ALL requests must wait for the LONGEST one to finish!
         Request 1 sits idle for 40 extra token-generation steps.
         GPU utilization: ~30%
```

### Continuous Batching (Used by vLLM/TGI)

```
Time →
Slot 1: [Req1 ████████]                [Req4 ██████████████████]
Slot 2: [Req2 ██████████████████████████████████████████████████]
Slot 3: [Req3 ████]     [Req5 ████████████]    [Req7 ██████]
Slot 4:                 [Req6 ████████████████████████████████]

As soon as a request finishes, a new one takes its slot!
No idle GPU cycles. Utilization: ~85%+
```

### Iteration-Level Scheduling

```
At each decode step:
1. Check if any request has finished
2. Remove finished requests from batch
3. Add waiting requests to batch (up to memory limit)
4. Run one decode step for entire batch
5. Repeat

Benefits:
• No request waits for the longest one
• GPU always busy
• Memory dynamically allocated/freed
```

---

## Deployment Architectures

### Architecture Options

```
Option 1: SINGLE GPU (Simple)
─────────────────────────────────
[Client] → [API Server] → [vLLM on 1× A100]
Best for: ≤13B models, low traffic

Option 2: MULTI-GPU (Scale Up)
─────────────────────────────────
[Client] → [API Server] → [vLLM on 4× A100 (TP=4)]
Best for: 70B models, medium traffic

Option 3: MULTI-INSTANCE (Scale Out)
─────────────────────────────────
                    ┌→ [vLLM Instance 1 (2× A100)]
[Client] → [Load   ├→ [vLLM Instance 2 (2× A100)]
            Balancer├→ [vLLM Instance 3 (2× A100)]
                    └→ [vLLM Instance 4 (2× A100)]
Best for: High traffic, production

Option 4: HYBRID (Multi-Tier)
─────────────────────────────────
[Client] → [Router] → Small model (fast, cheap) for simple queries
                     → Large model (slow, quality) for complex queries
Best for: Cost optimization at scale
```

### Cost Optimization Strategies

| Strategy | Savings | Trade-off |
|---|---|---|
| INT4 quantization | 4× memory → cheaper GPU | ~1% quality loss |
| Smaller model (7B vs 70B) | 10× cheaper | Significant quality drop |
| Prompt caching | 30-50% compute savings | Memory for cache |
| Batching | 5-10× throughput/$ | Slightly higher latency |
| Spot instances | 60-90% GPU cost | Can be interrupted |
| Model routing | 50%+ cost reduction | Complexity of routing logic |

---

## Code Examples

### Example 1: Quantizing with AutoGPTQ

```python
# pip install auto-gptq transformers

from transformers import AutoTokenizer
from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig

# ============================================
# GPTQ 4-bit Quantization
# ============================================

model_name = "meta-llama/Llama-2-7b-hf"
quantized_model_dir = "./llama2-7b-gptq-4bit"

# Step 1: Configure quantization
quantize_config = BaseQuantizeConfig(
    bits=4,               # Quantize to 4 bits
    group_size=128,       # Group weights for better accuracy (128 is standard)
    desc_act=False,       # Don't reorder by activation (faster)
    damp_percent=0.1,     # Dampening factor for Hessian
)

# Step 2: Load model and tokenizer
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoGPTQForCausalLM.from_pretrained(
    model_name,
    quantize_config=quantize_config,
)

# Step 3: Prepare calibration data (128 random samples)
import random
from datasets import load_dataset

dataset = load_dataset("c4", "en", split="train", streaming=True)
calibration_data = []
for i, example in enumerate(dataset):
    if i >= 128:
        break
    tokenized = tokenizer(example["text"], return_tensors="pt", max_length=2048, truncation=True)
    calibration_data.append(tokenized.input_ids)

# Step 4: Quantize (takes ~30 min for 7B model)
model.quantize(calibration_data)

# Step 5: Save quantized model
model.save_quantized(quantized_model_dir)
tokenizer.save_pretrained(quantized_model_dir)

print(f"Original model: ~14 GB")
print(f"Quantized model: ~4 GB (3.5× smaller)")

# Step 6: Load and use quantized model
model = AutoGPTQForCausalLM.from_quantized(
    quantized_model_dir,
    device="cuda:0",
    use_safetensors=True,
)

inputs = tokenizer("Explain quantum computing:", return_tensors="pt").to("cuda")
output = model.generate(**inputs, max_new_tokens=200)
print(tokenizer.decode(output[0], skip_special_tokens=True))
```

### Example 2: AWQ Quantization

```python
# pip install autoawq transformers

from awq import AutoAWQForCausalLM
from transformers import AutoTokenizer

# ============================================
# AWQ 4-bit Quantization (Generally better than GPTQ)
# ============================================

model_name = "meta-llama/Llama-2-7b-hf"
quant_path = "./llama2-7b-awq"

# Load model
model = AutoAWQForCausalLM.from_pretrained(model_name)
tokenizer = AutoTokenizer.from_pretrained(model_name)

# Quantize with AWQ
quant_config = {
    "zero_point": True,     # Use zero-point quantization
    "q_group_size": 128,    # Group size for quantization
    "w_bit": 4,             # Weight bit-width
    "version": "GEMM",      # Use GEMM kernels (fastest)
}

# AWQ calibration is FAST (~5 minutes for 7B)
model.quantize(tokenizer, quant_config=quant_config)

# Save
model.save_quantized(quant_path)
tokenizer.save_pretrained(quant_path)

# Load quantized model for inference
model = AutoAWQForCausalLM.from_quantized(
    quant_path,
    fuse_layers=True,  # Fuse QKV projections for speed
)

# Benchmark
import time
inputs = tokenizer("The meaning of life is", return_tensors="pt").to("cuda")
start = time.time()
output = model.generate(**inputs, max_new_tokens=100, do_sample=False)
elapsed = time.time() - start
num_tokens = output.shape[1] - inputs['input_ids'].shape[1]
print(f"Generated {num_tokens} tokens in {elapsed:.2f}s ({num_tokens/elapsed:.1f} tok/s)")
```

### Example 3: Deploying with vLLM

```python
# pip install vllm

# ============================================
# vLLM: High-Throughput LLM Serving
# ============================================

# --- Option A: Python API (for scripts/notebooks) ---

from vllm import LLM, SamplingParams

# Load model with vLLM (handles PagedAttention automatically)
llm = LLM(
    model="meta-llama/Llama-2-7b-chat-hf",
    tensor_parallel_size=1,       # Number of GPUs for tensor parallelism
    gpu_memory_utilization=0.90,  # Use 90% of GPU memory for KV cache
    max_model_len=4096,           # Maximum sequence length
    quantization="awq",           # Use AWQ quantized model (optional)
    dtype="half",                 # FP16 inference
    enforce_eager=False,          # Use CUDA graphs for speed
)

# Configure sampling
sampling_params = SamplingParams(
    temperature=0.7,
    top_p=0.9,
    max_tokens=512,
    stop=["</s>", "Human:"],  # Stop sequences
    presence_penalty=0.1,
    frequency_penalty=0.1,
)

# Single request
prompts = ["[INST] Explain the theory of relativity in simple terms. [/INST]"]
outputs = llm.generate(prompts, sampling_params)

for output in outputs:
    print(f"Prompt: {output.prompt[:50]}...")
    print(f"Response: {output.outputs[0].text}")
    print(f"Tokens generated: {len(output.outputs[0].token_ids)}")


# Batch inference (vLLM handles batching efficiently)
batch_prompts = [
    "[INST] What is Python? [/INST]",
    "[INST] Explain machine learning. [/INST]",
    "[INST] How does the internet work? [/INST]",
    "[INST] What is quantum computing? [/INST]",
]

batch_outputs = llm.generate(batch_prompts, sampling_params)
for output in batch_outputs:
    print(f"Q: {output.prompt[:40]}... → A: {output.outputs[0].text[:100]}...")
```

```bash
# --- Option B: OpenAI-Compatible API Server (for production) ---

# Start the server
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-7b-chat-hf \
    --tensor-parallel-size 2 \
    --gpu-memory-utilization 0.90 \
    --max-model-len 4096 \
    --host 0.0.0.0 \
    --port 8000 \
    --quantization awq
```

```python
# Client code (uses standard OpenAI SDK!)
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed",  # vLLM doesn't require auth by default
)

# Chat completion (same as OpenAI API!)
response = client.chat.completions.create(
    model="meta-llama/Llama-2-7b-chat-hf",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain recursion in programming."},
    ],
    temperature=0.7,
    max_tokens=512,
    stream=True,  # Streaming support
)

# Stream tokens as they're generated
for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

### Example 4: Using GGUF with llama.cpp (CPU/Local Deployment)

```python
# pip install llama-cpp-python

# ============================================
# llama.cpp: CPU-Friendly Local Inference
# ============================================

from llama_cpp import Llama

# Download a GGUF model (e.g., from TheBloke on HuggingFace)
# Model file: llama-2-7b-chat.Q4_K_M.gguf (~4.1 GB)

llm = Llama(
    model_path="./models/llama-2-7b-chat.Q4_K_M.gguf",
    n_ctx=4096,       # Context window
    n_threads=8,      # CPU threads (match your CPU cores)
    n_gpu_layers=35,  # Offload layers to GPU (0 for CPU-only)
    verbose=False,
)

# Generate with llama.cpp
output = llm(
    "[INST] Write a haiku about programming. [/INST]",
    max_tokens=100,
    temperature=0.7,
    top_p=0.9,
    echo=False,       # Don't include prompt in output
    stop=["</s>"],
)

print(output['choices'][0]['text'])
print(f"Tokens generated: {output['usage']['completion_tokens']}")
print(f"Speed: {output['usage']['completion_tokens'] / output['usage'].get('total_time', 1):.1f} tok/s")


# Chat completion API (OpenAI-compatible)
output = llm.create_chat_completion(
    messages=[
        {"role": "system", "content": "You are a helpful coding assistant."},
        {"role": "user", "content": "Write a Python function to reverse a string."},
    ],
    temperature=0.3,
    max_tokens=200,
)

print(output['choices'][0]['message']['content'])
```

### Example 5: Benchmarking Inference Performance

```python
import time
import torch
import numpy as np
from transformers import AutoModelForCausalLM, AutoTokenizer

# ============================================
# Inference Benchmarking
# ============================================

def benchmark_model(model, tokenizer, prompt, num_runs=5, max_tokens=100):
    """Benchmark TTFT and TPS for a model."""
    inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
    input_len = inputs['input_ids'].shape[1]
    
    ttft_times = []
    tps_values = []
    
    for run in range(num_runs):
        # Warm up on first run
        if run == 0:
            with torch.no_grad():
                model.generate(**inputs, max_new_tokens=1)
        
        # Measure TTFT (time to generate first token)
        start = time.perf_counter()
        with torch.no_grad():
            output = model.generate(
                **inputs,
                max_new_tokens=1,
                do_sample=False,
            )
        ttft = time.perf_counter() - start
        ttft_times.append(ttft)
        
        # Measure full generation speed
        start = time.perf_counter()
        with torch.no_grad():
            output = model.generate(
                **inputs,
                max_new_tokens=max_tokens,
                do_sample=False,
            )
        total_time = time.perf_counter() - start
        num_generated = output.shape[1] - input_len
        tps = num_generated / total_time
        tps_values.append(tps)
    
    results = {
        'ttft_mean': np.mean(ttft_times) * 1000,   # Convert to ms
        'ttft_std': np.std(ttft_times) * 1000,
        'tps_mean': np.mean(tps_values),
        'tps_std': np.std(tps_values),
        'tokens_generated': num_generated,
    }
    
    print(f"{'='*50}")
    print(f"Model: {model.config._name_or_path}")
    print(f"Input tokens: {input_len}")
    print(f"Output tokens: {results['tokens_generated']}")
    print(f"TTFT: {results['ttft_mean']:.1f} ± {results['ttft_std']:.1f} ms")
    print(f"TPS:  {results['tps_mean']:.1f} ± {results['tps_std']:.1f} tokens/sec")
    print(f"{'='*50}")
    
    return results


# Compare FP16 vs INT4
model_fp16 = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf", torch_dtype=torch.float16, device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-2-7b-hf")

prompt = "Explain the concept of artificial intelligence in detail:"

print("\n--- FP16 Model ---")
fp16_results = benchmark_model(model_fp16, tokenizer, prompt)

# Load INT4 quantized version
from auto_gptq import AutoGPTQForCausalLM

model_int4 = AutoGPTQForCausalLM.from_quantized(
    "TheBloke/Llama-2-7B-GPTQ", device="cuda:0", use_safetensors=True
)

print("\n--- INT4 (GPTQ) Model ---")
int4_results = benchmark_model(model_int4, tokenizer, prompt)

# Compare
print(f"\nSpeedup: {int4_results['tps_mean']/fp16_results['tps_mean']:.2f}×")
```

### Example 6: Docker Deployment with TGI

```dockerfile
# docker-compose.yml for TGI deployment
# version: "3.8"
# services:
#   tgi:
#     image: ghcr.io/huggingface/text-generation-inference:latest
#     ports:
#       - "8080:80"
#     volumes:
#       - ./models:/data
#     environment:
#       - MODEL_ID=meta-llama/Llama-2-7b-chat-hf
#       - QUANTIZE=awq
#       - MAX_INPUT_LENGTH=4096
#       - MAX_TOTAL_TOKENS=8192
#       - MAX_BATCH_PREFILL_TOKENS=4096
#     deploy:
#       resources:
#         reservations:
#           devices:
#             - capabilities: [gpu]
#               count: 1
```

```python
# Client for TGI server
import requests

# ============================================
# TGI Client
# ============================================

TGI_URL = "http://localhost:8080"

def generate_tgi(prompt, max_tokens=200, temperature=0.7, stream=False):
    """Send generation request to TGI server."""
    payload = {
        "inputs": prompt,
        "parameters": {
            "max_new_tokens": max_tokens,
            "temperature": temperature,
            "top_p": 0.9,
            "do_sample": True,
            "return_full_text": False,
        }
    }
    
    if stream:
        # Streaming response
        response = requests.post(
            f"{TGI_URL}/generate_stream",
            json=payload,
            stream=True,
        )
        for line in response.iter_lines():
            if line:
                import json
                data = json.loads(line.decode().removeprefix("data:"))
                token = data.get("token", {}).get("text", "")
                print(token, end="", flush=True)
    else:
        # Full response
        response = requests.post(f"{TGI_URL}/generate", json=payload)
        return response.json()

# Usage
result = generate_tgi(
    "[INST] What are the benefits of exercise? [/INST]",
    max_tokens=200,
    stream=True,
)
```

---

## Common Mistakes

### 1. Not Using Quantization for Inference
```
❌ Deploying FP32 model (28 GB for 7B model)
   4× more memory, no quality benefit for inference

✅ Use INT4/FP16: 3.5-14 GB with <1% quality loss
   Rule: FP16 minimum, INT4 for cost optimization
```

### 2. Static Batching in Production
```
❌ Wait for batch to fill, process all together
   One long response holds up all others

✅ Use continuous batching (vLLM/TGI)
   Each request enters/exits independently
   5-10× better throughput
```

### 3. Ignoring KV Cache Memory
```
❌ Setting max_model_len=32768 without calculating memory
   KV cache for 32K context on 70B model = ~80 GB!

✅ Calculate: KV_memory = 2 × layers × heads × d_head × seq_len × 2bytes
   Set max_model_len based on available GPU memory AFTER model weights
```

### 4. Wrong GPU Memory Utilization Setting
```
❌ gpu_memory_utilization=0.99  → OOM crashes under load
❌ gpu_memory_utilization=0.50  → Wasting half your GPU

✅ gpu_memory_utilization=0.85-0.90 (leave buffer for overhead)
```

### 5. Not Measuring the Right Metrics
```
❌ Only measuring average latency
   (P99 could be 10× worse due to long-context requests)

✅ Measure: TTFT (P50, P99), TPS, throughput, and tail latency
   Monitor under realistic load, not just single requests
```

### 6. Running Large Models on Wrong Hardware
```
❌ Running 70B model on single A100 (needs 140 GB in FP16)
   Even INT4 needs ~40 GB (A100 only has 80 GB, but KV cache needs space)

✅ Hardware planning:
   7B INT4  → 1× RTX 4090 (24 GB) or T4 (16 GB)
   13B INT4 → 1× A100 (40/80 GB)
   70B INT4 → 2× A100 80 GB (TP=2)
```

---

## Interview Questions

### Conceptual Questions

**Q1: What are the two phases of LLM inference and why do they matter?**
> Prefill (prompt processing): processes all input tokens in parallel, compute-bound, determines TTFT. Decode (generation): generates tokens sequentially one at a time, memory-bound (reads entire model weights per token), determines TPS. Decode is the bottleneck because GPU utilization is very low (~5%) — the GPU reads weights from memory faster than it can do useful computation per token.

**Q2: Explain PagedAttention and why it's important.**
> PagedAttention (used by vLLM) manages KV cache like virtual memory — allocating small fixed-size pages on demand instead of pre-allocating maximum possible cache per request. This eliminates memory fragmentation and waste, improving memory utilization by ~4× and enabling ~2-3× more concurrent requests on the same GPU.

**Q3: Compare GPTQ vs AWQ quantization. When would you use each?**
> GPTQ: Uses calibration data to find optimal rounding of weights layer-by-layer, minimizing output error. Slower calibration (~30 min for 7B). AWQ: Identifies the ~1% most important weight channels (based on activation magnitudes) and preserves their precision by scaling. Faster calibration (~5 min). AWQ generally produces slightly better quality and faster inference. Use AWQ as default for production; GPTQ when AWQ isn't available for your architecture.

**Q4: What is speculative decoding?**
> Use a small, fast "draft" model to generate multiple candidate tokens, then verify them in parallel with the large model. Since verification is a single forward pass (prefill), it's much faster than sequential generation. Accepted tokens provide ~2-3× speedup. Quality is identical to the large model (rejected tokens are regenerated).

**Q5: How does continuous batching differ from static batching?**
> Static: waits for a batch to fill, processes together, all wait for the longest. Continuous: at each decode step, finished requests exit and waiting requests enter. Result: 5-10× better throughput, lower tail latency. Implemented by vLLM and TGI.

**Q6: Calculate the memory needed to serve LLaMA-2 70B in INT4 with 4096 context.**
> Model weights (INT4): 70B × 0.5 bytes = 35 GB. KV cache per token: 2 × 80 layers × 8 KV heads (GQA) × 128 dim × 2 bytes = 327 KB. For 4096 context: 327 KB × 4096 = 1.3 GB per request. With 10 concurrent requests: 35 GB + 13 GB = ~48 GB → fits on 1× A100 80 GB.

**Q7: What is the relationship between model size, batch size, and GPU memory?**
> GPU memory = Model weights + KV cache (per request × batch size) + Activation memory + Overhead. Larger batch → more KV cache → more memory. This is why `gpu_memory_utilization` in vLLM controls how much memory is available for KV cache after loading weights. More KV cache space → larger batches → higher throughput.

**Q8: Explain GQA (Grouped-Query Attention) and its impact on deployment.**
> GQA groups multiple query heads to share the same K and V heads. LLaMA-2 70B: 64 query heads but only 8 KV heads (8:1 ratio). This reduces KV cache by 8× compared to standard MHA, enabling longer contexts and more concurrent users within the same memory budget.

### Practical Questions

**Q9: You need to deploy a 7B chat model for a startup with limited budget. What's your approach?**
> 1. Quantize to INT4 using AWQ (~4 GB). 2. Serve with vLLM on a single A10G/T4 GPU (cheapest option). 3. Use continuous batching for concurrent users. 4. Enable prefix caching for common system prompts. 5. Monitor TTFT, TPS, and error rates. Estimated cost: ~$0.50-1.50/hr on cloud.

**Q10: How would you reduce the cost of serving an LLM API by 5×?**
> Combine strategies: (1) INT4 quantization: 4× memory → smaller/fewer GPUs. (2) Use smaller model where quality is acceptable (route simple queries to 7B, complex to 70B). (3) Prompt caching: reduce redundant computation for shared prefixes. (4) Spot/preemptible instances: 60-70% cost savings. (5) Optimize batch sizes for throughput.

---

## Quick Reference

### Quantization Cheat Sheet

| Method | Bits | Memory (7B) | Quality Loss | Speed | Best For |
|---|---|---|---|---|---|
| FP16 | 16 | 14 GB | None | Baseline | When quality is paramount |
| GPTQ | 4 | 4 GB | Minimal | Good | GPU inference |
| AWQ | 4 | 4 GB | Minimal | Better | GPU inference (recommended) |
| GGUF Q4_K_M | 4 | 4 GB | Minimal | Good | CPU/hybrid inference |
| GGUF Q2_K | 2 | 2.5 GB | Noticeable | Fast | Extreme compression |

### Serving Framework Decision Guide

```
Need GPU production serving?     → vLLM (best throughput)
Need HuggingFace integration?    → TGI
Need local/development?          → Ollama (easiest setup)
Need CPU/edge deployment?        → llama.cpp / GGUF
Need maximum customization?      → vLLM Python API
```

### GPU Memory Planning

| Model | FP16 | INT4 | Min GPU (INT4) |
|---|---|---|---|
| 7B | 14 GB | 4 GB | T4 16 GB |
| 13B | 26 GB | 7 GB | A10G 24 GB |
| 34B | 68 GB | 18 GB | A100 40 GB |
| 70B | 140 GB | 35 GB | 2× A100 80 GB |

### vLLM Quick Start Commands

```bash
# Basic serving
python -m vllm.entrypoints.openai.api_server --model <model_name>

# With quantization
python -m vllm.entrypoints.openai.api_server --model <model> --quantization awq

# Multi-GPU
python -m vllm.entrypoints.openai.api_server --model <model> --tensor-parallel-size 4

# Memory tuning
python -m vllm.entrypoints.openai.api_server --model <model> --gpu-memory-utilization 0.90
```

### Performance Optimization Checklist

- [ ] Use FP16 or INT4 (never FP32 for inference)
- [ ] Use continuous batching (vLLM or TGI)
- [ ] Enable FlashAttention
- [ ] Enable prefix caching for shared system prompts
- [ ] Use CUDA graphs (`enforce_eager=False`)
- [ ] Match `max_model_len` to actual needs
- [ ] Set `gpu_memory_utilization` to 0.85-0.90
- [ ] Monitor TTFT, TPS, P99 latency, GPU utilization
- [ ] Consider speculative decoding for latency-sensitive apps
- [ ] Use GQA/MQA models for long-context deployments

---

> **Key Takeaway**: LLM deployment is about maximizing throughput while meeting latency requirements within a memory budget. INT4 quantization + vLLM with continuous batching + proper KV cache management is the production standard. Always measure TTFT, TPS, and P99 latency — not just average performance.
