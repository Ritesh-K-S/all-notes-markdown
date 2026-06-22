# Chapter 04: Fine-Tuning Large Language Models

## Table of Contents
- [What is Fine-Tuning?](#what-is-fine-tuning)
- [Why Fine-Tuning Matters](#why-fine-tuning-matters)
- [Types of Fine-Tuning](#types-of-fine-tuning)
- [Full Fine-Tuning](#full-fine-tuning)
- [Parameter-Efficient Fine-Tuning (PEFT)](#parameter-efficient-fine-tuning-peft)
- [LoRA (Low-Rank Adaptation)](#lora-low-rank-adaptation)
- [QLoRA (Quantized LoRA)](#qlora-quantized-lora)
- [Dataset Preparation](#dataset-preparation)
- [Training Configuration & Hyperparameters](#training-configuration--hyperparameters)
- [Evaluation & Monitoring](#evaluation--monitoring)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Fine-Tuning?

### Simple Explanation (ELI15)

Imagine you hire a brilliant university graduate (the pre-trained LLM). They know a LOT about the world — history, science, language — but they've never worked at YOUR company. Fine-tuning is like giving them a week of on-the-job training with your specific documents, your tone of voice, and your domain expertise. After training, they can do YOUR specific job much better, while still retaining all their general knowledge.

### Formal Definition

Fine-tuning is the process of taking a pre-trained language model and continuing its training on a smaller, domain-specific dataset to adapt it for a particular task or behavior. It adjusts the model's weights to specialize its capabilities while leveraging the vast knowledge learned during pre-training.

```
┌─────────────────────────────────────────────────────────────────┐
│                    FINE-TUNING OVERVIEW                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pre-trained Model ──► Fine-tuning Process ──► Specialized Model│
│  (General Knowledge)     (Your Data)          (Your Expert)     │
│                                                                 │
│  GPT-4, LLaMA, etc.   Domain-specific       Follows YOUR       │
│  Knows everything      examples, Q&A,        instructions,      │
│  but nothing specific  conversations         YOUR format        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Why Fine-Tuning Matters

### When to Fine-Tune vs. Other Approaches

| Approach | When to Use | Cost | Effort |
|----------|-------------|------|--------|
| **Prompt Engineering** | Simple tasks, few formatting needs | Low (API calls) | Low |
| **Few-Shot Prompting** | Need specific output format | Medium (longer prompts) | Low |
| **RAG** | Need up-to-date/private knowledge | Medium | Medium |
| **Fine-Tuning** | Need consistent behavior/style/format | High (compute) | High |
| **Fine-Tuning + RAG** | Need both custom behavior AND knowledge | Highest | Highest |

### Real-World Use Cases

1. **Customer Support Bot** — Fine-tune to match your brand voice and handle domain-specific queries
2. **Code Assistant** — Fine-tune on your codebase conventions and internal APIs
3. **Medical Report Generator** — Fine-tune to follow specific medical terminology and report formats
4. **Legal Document Analyzer** — Fine-tune to understand jurisdiction-specific legal language
5. **Content Moderation** — Fine-tune to classify content according to YOUR platform's policies

### Decision Framework: Should You Fine-Tune?

```
┌──────────────────────────────────────────────────┐
│         DO YOU NEED TO FINE-TUNE?                 │
├──────────────────────────────────────────────────┤
│                                                  │
│  ✅ Fine-tune when:                              │
│  • Prompt engineering isn't consistent enough     │
│  • You need a specific output format always      │
│  • You need to reduce token usage (shorter       │
│    prompts after fine-tuning)                    │
│  • You need domain-specific behavior             │
│  • Latency matters (no long system prompts)      │
│                                                  │
│  ❌ DON'T fine-tune when:                        │
│  • You just need factual knowledge (use RAG)     │
│  • Few-shot prompting works well enough          │
│  • You don't have quality training data          │
│  • The base model already does what you need     │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

## Types of Fine-Tuning

### Overview Spectrum

```
┌─────────────────────────────────────────────────────────────────────┐
│                   FINE-TUNING SPECTRUM                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  MORE Parameters Changed ◄──────────────────► FEWER Parameters      │
│                                                                     │
│  Full Fine-Tuning    Feature Extraction    LoRA    Prompt Tuning    │
│  (All weights)       (Last layers)         (Low-rank) (Soft prompts)│
│                                                                     │
│  Most Expensive ◄────────────────────────────► Least Expensive      │
│  Most Flexible  ◄────────────────────────────► Least Flexible       │
│  Most Risk of   ◄────────────────────────────► Least Risk of        │
│  Catastrophic                                   Catastrophic         │
│  Forgetting                                     Forgetting           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Full Fine-Tuning

### What It Is

Full fine-tuning updates **ALL** parameters of the model during training. For a 7B parameter model, that means adjusting all 7 billion weights.

### How It Works

```
┌───────────────────────────────────────────────────┐
│           FULL FINE-TUNING PROCESS                 │
├───────────────────────────────────────────────────┤
│                                                   │
│  Input: "Summarize this medical report"           │
│         ↓                                         │
│  ┌─────────────────────────────┐                  │
│  │ ALL Layers Updated          │                  │
│  │ • Embedding Layer    ✓      │                  │
│  │ • Attention Layers   ✓      │  Backpropagation │
│  │ • FFN Layers         ✓      │  through ALL     │
│  │ • Output Layer       ✓      │  parameters      │
│  └─────────────────────────────┘                  │
│         ↓                                         │
│  Output: "Patient presents with..."               │
│         ↓                                         │
│  Loss = CrossEntropy(predicted, expected)         │
│         ↓                                         │
│  Update ALL 7B parameters via gradient descent    │
│                                                   │
└───────────────────────────────────────────────────┘
```

### Requirements

| Model Size | GPU Memory (FP16) | GPU Memory (FP32) | Approx. Cost |
|-----------|-------------------|-------------------|--------------|
| 7B | ~28 GB | ~56 GB | 4x A100 40GB |
| 13B | ~52 GB | ~104 GB | 8x A100 40GB |
| 70B | ~280 GB | ~560 GB | 64x A100 80GB |

> **Memory Rule of Thumb**: You need approximately 4x the model size in memory for full fine-tuning (model weights + gradients + optimizer states + activations).

### Code Example: Full Fine-Tuning with Hugging Face

```python
# Full Fine-Tuning of a Small LLM (e.g., GPT-2 or small LLaMA)
# WARNING: This requires significant GPU memory

from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    TrainingArguments,
    Trainer,
    DataCollatorForLanguageModeling
)
from datasets import load_dataset

# Step 1: Load the pre-trained model and tokenizer
model_name = "meta-llama/Llama-2-7b-hf"  # Requires access approval
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype="auto",       # Automatically select best dtype
    device_map="auto"         # Distribute across available GPUs
)

# Step 2: Add padding token if not present (common for LLaMA)
if tokenizer.pad_token is None:
    tokenizer.pad_token = tokenizer.eos_token
    model.config.pad_token_id = tokenizer.eos_token_id

# Step 3: Load and prepare dataset
dataset = load_dataset("json", data_files="training_data.jsonl")

# Step 4: Tokenize the dataset
def tokenize_function(examples):
    """Convert text to token IDs with proper formatting."""
    # Format: instruction + input → output
    texts = []
    for instruction, output in zip(examples["instruction"], examples["output"]):
        text = f"### Instruction:\n{instruction}\n\n### Response:\n{output}"
        texts.append(text)
    
    tokenized = tokenizer(
        texts,
        truncation=True,
        max_length=512,          # Adjust based on your data
        padding="max_length",
        return_tensors="pt"
    )
    tokenized["labels"] = tokenized["input_ids"].clone()
    return tokenized

tokenized_dataset = dataset.map(tokenize_function, batched=True)

# Step 5: Define training arguments
training_args = TrainingArguments(
    output_dir="./full-ft-llama",
    num_train_epochs=3,
    per_device_train_batch_size=4,      # Reduce if OOM
    gradient_accumulation_steps=8,       # Effective batch = 4 * 8 = 32
    learning_rate=2e-5,                  # Lower LR for fine-tuning
    weight_decay=0.01,
    warmup_steps=100,
    logging_steps=10,
    save_strategy="epoch",
    fp16=True,                           # Mixed precision training
    gradient_checkpointing=True,         # Trade compute for memory
    dataloader_num_workers=4,
    report_to="wandb"                    # Track experiments
)

# Step 6: Initialize trainer and train
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_dataset["train"],
    data_collator=DataCollatorForLanguageModeling(
        tokenizer=tokenizer, 
        mlm=False  # Causal LM, not masked LM
    )
)

# Step 7: Start training
trainer.train()

# Step 8: Save the fine-tuned model
trainer.save_model("./full-ft-llama-final")
tokenizer.save_pretrained("./full-ft-llama-final")
```

### Catastrophic Forgetting

The biggest risk with full fine-tuning is **catastrophic forgetting** — the model "forgets" its general capabilities while learning the new task.

**Mitigation Strategies:**
1. Use a low learning rate (1e-5 to 5e-5)
2. Train for fewer epochs (1-3 typically)
3. Mix in some general-purpose data during fine-tuning
4. Use regularization techniques

---

## Parameter-Efficient Fine-Tuning (PEFT)

### What It Is

PEFT methods fine-tune only a small subset of model parameters (often <1% of total), keeping most of the original model frozen. This dramatically reduces memory requirements and training time while achieving comparable performance to full fine-tuning.

### Why PEFT Exists

```
The Problem with Full Fine-Tuning:
┌────────────────────────────────────────────────┐
│  7B Model Full Fine-Tuning:                    │
│  • Model weights:        14 GB (FP16)          │
│  • Gradients:            14 GB                 │
│  • Optimizer states:     28 GB (Adam)          │
│  • Activations:          ~10 GB               │
│  ────────────────────────────────              │
│  TOTAL:                  ~66 GB               │
│                                                │
│  PEFT (LoRA) Fine-Tuning:                     │
│  • Model weights:        14 GB (frozen)        │
│  • LoRA weights:         ~0.1 GB              │
│  • Gradients:            ~0.1 GB              │
│  • Optimizer states:     ~0.2 GB              │
│  ────────────────────────────────              │
│  TOTAL:                  ~15 GB               │
│                                                │
│  Memory Savings: ~75%!                         │
└────────────────────────────────────────────────┘
```

### PEFT Methods Comparison

| Method | Trainable Params | Memory | Performance | Complexity |
|--------|-----------------|--------|-------------|------------|
| **LoRA** | 0.1-1% | Low | High | Low |
| **QLoRA** | 0.1-1% | Very Low | High | Medium |
| **Prefix Tuning** | <0.1% | Very Low | Medium | Low |
| **Prompt Tuning** | <0.01% | Minimal | Lower | Very Low |
| **Adapter Layers** | 1-5% | Low | High | Medium |
| **IA3** | <0.01% | Minimal | Medium | Low |

---

## LoRA (Low-Rank Adaptation)

### What It Is (ELI15)

Imagine a massive painting (the pre-trained model). Instead of repainting the entire canvas (full fine-tuning), LoRA adds a tiny transparent overlay (low-rank matrices) on top. The overlay slightly adjusts the colors and details for your specific needs, while the original painting stays untouched underneath.

### The Math Behind LoRA

In a standard neural network layer, the weight update during fine-tuning is:

$$W_{new} = W_{original} + \Delta W$$

LoRA's key insight: The weight update $\Delta W$ is typically **low-rank** — it can be decomposed into two smaller matrices:

$$\Delta W = B \times A$$

Where:
- $W \in \mathbb{R}^{d \times k}$ — Original weight matrix (e.g., 4096 × 4096)
- $A \in \mathbb{R}^{r \times k}$ — Down-projection (e.g., 16 × 4096)
- $B \in \mathbb{R}^{d \times r}$ — Up-projection (e.g., 4096 × 16)
- $r$ — Rank (typically 8, 16, 32, or 64)

**Parameter reduction:**
- Original $\Delta W$: $d \times k = 4096 \times 4096 = 16,777,216$ parameters
- LoRA $B \times A$: $d \times r + r \times k = 4096 \times 16 + 16 \times 4096 = 131,072$ parameters
- **Reduction: 128x fewer parameters!**

### How LoRA Works — Visual

```
┌─────────────────────────────────────────────────────────────────┐
│                    LoRA ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Input x                                                        │
│    │                                                            │
│    ├──────────────────────────────┐                             │
│    │                              │                             │
│    ▼                              ▼                             │
│  ┌──────────┐              ┌──────────┐                         │
│  │ W (frozen)│              │ A (d→r)  │  ← Trainable           │
│  │ d × k    │              │ r × k    │  (randomly initialized) │
│  └────┬─────┘              └────┬─────┘                         │
│       │                         │                               │
│       │                         ▼                               │
│       │                   ┌──────────┐                          │
│       │                   │ B (r→d)  │  ← Trainable            │
│       │                   │ d × r    │  (initialized to zero)   │
│       │                   └────┬─────┘                          │
│       │                        │                                │
│       │                        │ × (α/r) ← Scaling factor      │
│       │                        │                                │
│       ▼                        ▼                                │
│       └────────── + ───────────┘                                │
│                   │                                             │
│                   ▼                                             │
│              Output h = Wx + (α/r)BAx                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Key LoRA Hyperparameters

| Parameter | Description | Typical Values | Impact |
|-----------|-------------|----------------|--------|
| `r` (rank) | Dimension of low-rank matrices | 8, 16, 32, 64 | Higher = more capacity, more memory |
| `alpha` | Scaling factor | 16, 32 (often = 2×r) | Controls magnitude of updates |
| `target_modules` | Which layers to apply LoRA | q_proj, v_proj, k_proj, o_proj | More modules = more capacity |
| `dropout` | LoRA dropout rate | 0.05 - 0.1 | Regularization |

> **Pro Tip**: The effective scaling is `alpha/r`. If you set r=16 and alpha=32, the scaling is 2. Many practitioners set alpha = 2 × r as a starting point.

### Code Example: LoRA Fine-Tuning

```python
# LoRA Fine-Tuning with PEFT library
# Can run on a single GPU with 16-24 GB VRAM for 7B models

import torch
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    TrainingArguments,
    Trainer,
    DataCollatorForLanguageModeling
)
from peft import (
    LoraConfig,
    get_peft_model,
    TaskType,
    prepare_model_for_kbit_training
)
from datasets import load_dataset

# Step 1: Load base model
model_name = "meta-llama/Llama-2-7b-hf"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.float16,
    device_map="auto"
)

# Fix padding token
tokenizer.pad_token = tokenizer.eos_token
model.config.pad_token_id = tokenizer.eos_token_id

# Step 2: Define LoRA configuration
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,     # Type of task
    r=16,                              # Rank — start with 16
    lora_alpha=32,                     # Alpha — usually 2x rank
    lora_dropout=0.05,                 # Small dropout for regularization
    target_modules=[                   # Which layers to apply LoRA to
        "q_proj",                      # Query projection in attention
        "k_proj",                      # Key projection
        "v_proj",                      # Value projection
        "o_proj",                      # Output projection
        "gate_proj",                   # MLP gate (for LLaMA)
        "up_proj",                     # MLP up projection
        "down_proj"                    # MLP down projection
    ],
    bias="none"                        # Don't train bias terms
)

# Step 3: Create PEFT model (wraps original with LoRA layers)
model = get_peft_model(model, lora_config)

# Check trainable parameters
model.print_trainable_parameters()
# Output: trainable params: 33,554,432 || all params: 6,771,970,048 || trainable%: 0.4956%

# Step 4: Prepare dataset
dataset = load_dataset("json", data_files="training_data.jsonl")

def format_and_tokenize(examples):
    """Format data as instruction-response pairs and tokenize."""
    texts = []
    for inst, resp in zip(examples["instruction"], examples["response"]):
        # Alpaca-style format
        text = (
            f"Below is an instruction that describes a task. "
            f"Write a response that appropriately completes the request.\n\n"
            f"### Instruction:\n{inst}\n\n"
            f"### Response:\n{resp}{tokenizer.eos_token}"
        )
        texts.append(text)
    
    tokenized = tokenizer(
        texts,
        truncation=True,
        max_length=512,
        padding="max_length"
    )
    tokenized["labels"] = tokenized["input_ids"].copy()
    return tokenized

tokenized_data = dataset.map(format_and_tokenize, batched=True)

# Step 5: Training arguments (lighter than full fine-tuning)
training_args = TrainingArguments(
    output_dir="./lora-llama",
    num_train_epochs=3,
    per_device_train_batch_size=8,       # Can use larger batch with LoRA
    gradient_accumulation_steps=4,
    learning_rate=2e-4,                   # Higher LR is OK for LoRA
    weight_decay=0.01,
    warmup_ratio=0.03,
    lr_scheduler_type="cosine",
    logging_steps=10,
    save_strategy="steps",
    save_steps=200,
    fp16=True,
    optim="adamw_torch",
    report_to="wandb"
)

# Step 6: Train
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_data["train"],
    data_collator=DataCollatorForLanguageModeling(tokenizer, mlm=False)
)

trainer.train()

# Step 7: Save LoRA adapter (only saves the small LoRA weights!)
model.save_pretrained("./lora-llama-adapter")
# This saves only ~130MB instead of 14GB!

# Step 8: Inference with LoRA adapter
from peft import PeftModel

# Load base model
base_model = AutoModelForCausalLM.from_pretrained(
    model_name, torch_dtype=torch.float16, device_map="auto"
)
# Load LoRA adapter on top
model = PeftModel.from_pretrained(base_model, "./lora-llama-adapter")

# Merge LoRA weights into base model for faster inference (optional)
model = model.merge_and_unload()

# Generate
inputs = tokenizer("### Instruction:\nSummarize...", return_tensors="pt").to("cuda")
outputs = model.generate(**inputs, max_new_tokens=256)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

---

## QLoRA (Quantized LoRA)

### What It Is

QLoRA combines **quantization** (reducing weight precision) with LoRA to enable fine-tuning of massive models on consumer hardware. It quantizes the base model to 4-bit precision, then trains LoRA adapters in 16-bit on top.

### The Magic of QLoRA

```
┌─────────────────────────────────────────────────────────────────┐
│                    QLoRA vs LoRA vs Full FT                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Full Fine-Tuning (7B model):                                   │
│  ├── Model: FP16 (14 GB)                                       │
│  ├── Gradients: FP16 (14 GB)                                   │
│  ├── Optimizer: FP32 (28 GB)                                   │
│  └── TOTAL: ~56 GB → Need 4x A100                             │
│                                                                 │
│  LoRA (7B model):                                               │
│  ├── Model: FP16 (14 GB) [FROZEN]                              │
│  ├── LoRA weights: FP16 (~0.1 GB)                              │
│  ├── Gradients: FP16 (~0.1 GB)                                 │
│  └── TOTAL: ~16 GB → Need 1x A100 or 1x RTX 4090             │
│                                                                 │
│  QLoRA (7B model):                                              │
│  ├── Model: NF4 (3.5 GB) [FROZEN, 4-bit quantized]            │
│  ├── LoRA weights: FP16 (~0.1 GB)                              │
│  ├── Gradients: FP16 (~0.1 GB)                                 │
│  └── TOTAL: ~6 GB → Need 1x RTX 3090 or even RTX 3080!       │
│                                                                 │
│  QLoRA (70B model):                                             │
│  ├── Model: NF4 (35 GB) [FROZEN]                               │
│  ├── LoRA weights: FP16 (~0.5 GB)                              │
│  └── TOTAL: ~48 GB → Need 1x A100 80GB                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### QLoRA Key Innovations

1. **4-bit NormalFloat (NF4)** — A new data type optimized for normally-distributed neural network weights
2. **Double Quantization** — Quantizes the quantization constants themselves, saving additional memory
3. **Paged Optimizers** — Uses NVIDIA unified memory to handle memory spikes during training

### The Math: NormalFloat4 Quantization

Standard 4-bit quantization divides the range evenly. NF4 instead uses quantiles of a normal distribution:

$$q_i = \frac{1}{2}\left(\text{QNorm}\left(\frac{i}{2^k+1}\right) + \text{QNorm}\left(\frac{i+1}{2^k+1}\right)\right)$$

This means more precision around zero (where most weights cluster) and less at extremes.

### Code Example: QLoRA Fine-Tuning

```python
# QLoRA: Fine-tune a 7B model on a single consumer GPU (16GB+)
# This is the most cost-effective way to fine-tune large models

import torch
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    TrainingArguments,
    BitsAndBytesConfig
)
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from trl import SFTTrainer  # Supervised Fine-Tuning Trainer
from datasets import load_dataset

# Step 1: Configure 4-bit quantization
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,                    # Load model in 4-bit
    bnb_4bit_quant_type="nf4",           # Use NormalFloat4 (best for LLMs)
    bnb_4bit_compute_dtype=torch.bfloat16, # Compute in bfloat16
    bnb_4bit_use_double_quant=True        # Double quantization for extra savings
)

# Step 2: Load model with quantization
model_name = "meta-llama/Llama-2-7b-hf"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=bnb_config,      # Apply 4-bit quantization
    device_map="auto",
    trust_remote_code=True
)

# Step 3: Prepare model for k-bit training
# This handles gradient checkpointing and layer norm casting
model = prepare_model_for_kbit_training(model)

# Step 4: LoRA config (same as regular LoRA)
lora_config = LoraConfig(
    r=64,                                 # Can use higher rank since base is 4-bit
    lora_alpha=128,                       # 2x rank
    lora_dropout=0.05,
    target_modules=[
        "q_proj", "k_proj", "v_proj", "o_proj",
        "gate_proj", "up_proj", "down_proj"
    ],
    bias="none",
    task_type="CAUSAL_LM"
)

# Step 5: Wrap model with LoRA
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Output: trainable params: 83,886,080 || all params: 6,855,856,128 || trainable%: 1.22%

# Step 6: Load dataset
dataset = load_dataset("json", data_files="training_data.jsonl", split="train")

# Step 7: Define formatting function for SFTTrainer
def formatting_func(example):
    """Format each example as a conversation."""
    text = (
        f"<s>[INST] {example['instruction']} [/INST] "
        f"{example['response']}</s>"
    )
    return text

# Step 8: Training arguments optimized for QLoRA
training_args = TrainingArguments(
    output_dir="./qlora-llama",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    learning_rate=2e-4,
    weight_decay=0.01,
    warmup_ratio=0.03,
    lr_scheduler_type="cosine",
    logging_steps=10,
    save_strategy="steps",
    save_steps=100,
    bf16=True,                            # Use bfloat16 for training
    gradient_checkpointing=True,          # Save memory
    optim="paged_adamw_32bit",           # Paged optimizer for memory spikes
    max_grad_norm=0.3,                    # Gradient clipping
    group_by_length=True,                 # Group similar lengths for efficiency
    report_to="wandb"
)

# Step 9: Use SFTTrainer (handles formatting automatically)
trainer = SFTTrainer(
    model=model,
    args=training_args,
    train_dataset=dataset,
    formatting_func=formatting_func,
    tokenizer=tokenizer,
    max_seq_length=512,
    packing=True                          # Pack multiple short examples together
)

# Step 10: Train!
trainer.train()

# Step 11: Save adapter
trainer.save_model("./qlora-llama-adapter")

# ═══════════════════════════════════════════════════════════════
# INFERENCE: Load quantized model + adapter
# ═══════════════════════════════════════════════════════════════

from peft import AutoPeftModelForCausalLM

# Load the fine-tuned model (handles quantization + adapter automatically)
model = AutoPeftModelForCausalLM.from_pretrained(
    "./qlora-llama-adapter",
    torch_dtype=torch.bfloat16,
    device_map="auto"
)

# Generate text
prompt = "[INST] Write a Python function to calculate fibonacci numbers [/INST]"
inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
outputs = model.generate(
    **inputs,
    max_new_tokens=512,
    temperature=0.7,
    top_p=0.9,
    do_sample=True
)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

### Pro Tip: Merging and Exporting

```python
# After QLoRA training, you can merge and save as full-precision model
from peft import AutoPeftModelForCausalLM

model = AutoPeftModelForCausalLM.from_pretrained(
    "./qlora-llama-adapter",
    torch_dtype=torch.float16,
    device_map="auto",
    # Load in 16-bit for merging (not 4-bit!)
    low_cpu_mem_usage=True
)

# Merge LoRA weights into the base model
merged_model = model.merge_and_unload()

# Save as a standard model (can be quantized later for inference)
merged_model.save_pretrained("./merged-model", safe_serialization=True)
tokenizer.save_pretrained("./merged-model")

# Now you can upload to Hugging Face Hub
# merged_model.push_to_hub("your-username/your-model")
```

---

## Dataset Preparation

### The Most Important Step

> **80% of fine-tuning success is in the data.** A small, high-quality dataset (500-5000 examples) will almost always outperform a large, noisy dataset (50,000+ examples).

### Data Formats

#### 1. Instruction Format (Most Common)

```json
{
    "instruction": "Summarize the following medical report in 2-3 sentences.",
    "input": "Patient John, 45M, presented with chest pain...",
    "output": "A 45-year-old male patient presented with acute chest pain..."
}
```

#### 2. Conversational Format (Chat Models)

```json
{
    "messages": [
        {"role": "system", "content": "You are a helpful medical assistant."},
        {"role": "user", "content": "What are the symptoms of diabetes?"},
        {"role": "assistant", "content": "The common symptoms of diabetes include..."}
    ]
}
```

#### 3. Completion Format (Simple)

```json
{
    "text": "### Question: What is photosynthesis?\n### Answer: Photosynthesis is the process by which plants convert sunlight into energy..."
}
```

### Dataset Quality Checklist

```
┌─────────────────────────────────────────────────────┐
│           DATASET QUALITY CHECKLIST                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ✅ Diverse — covers all scenarios you want         │
│  ✅ Consistent — same format throughout             │
│  ✅ Accurate — outputs are correct                  │
│  ✅ Representative — matches real production input  │
│  ✅ Balanced — no over-representation of one class  │
│  ✅ Clean — no duplicates, no corrupted entries     │
│  ✅ Appropriate length — matches expected outputs   │
│  ✅ Includes edge cases — handles weird inputs      │
│                                                     │
│  ❌ Common Problems:                                │
│  • Too few examples (<100 is usually not enough)    │
│  • Inconsistent formatting                          │
│  • Errors in labels/outputs                         │
│  • All examples too similar (overfitting risk)      │
│  • Training data doesn't match inference input      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Code: Dataset Preparation Pipeline

```python
# Complete dataset preparation pipeline for fine-tuning

import json
import random
from typing import List, Dict
from datasets import Dataset, DatasetDict

# ═══════════════════════════════════════════════════════
# Step 1: Create/Load your raw data
# ═══════════════════════════════════════════════════════

def create_training_data() -> List[Dict]:
    """Create or load your training examples."""
    
    # Example: Customer support fine-tuning data
    examples = [
        {
            "instruction": "A customer asks about return policy for electronics.",
            "input": "Hi, I bought a laptop 20 days ago and it's not working. Can I return it?",
            "output": "I'd be happy to help with your return! Our electronics return policy allows returns within 30 days of purchase. Since you bought your laptop 20 days ago, you're still within the return window. Here's what you need to do:\n\n1. Ensure you have the original receipt\n2. Pack the laptop in its original box with all accessories\n3. Visit any of our stores or initiate a return online\n\nWould you like me to start the return process for you?"
        },
        # ... hundreds more examples
    ]
    return examples

# ═══════════════════════════════════════════════════════
# Step 2: Data validation and cleaning
# ═══════════════════════════════════════════════════════

def validate_example(example: Dict) -> bool:
    """Validate a single training example."""
    # Check required fields exist
    required_fields = ["instruction", "output"]
    if not all(field in example for field in required_fields):
        return False
    
    # Check minimum length (too short = low quality)
    if len(example["output"]) < 20:
        return False
    
    # Check maximum length (too long = might get truncated)
    if len(example["instruction"]) + len(example.get("input", "")) + len(example["output"]) > 4000:
        return False
    
    # Check for empty or whitespace-only fields
    if not example["instruction"].strip() or not example["output"].strip():
        return False
    
    return True

def deduplicate(examples: List[Dict]) -> List[Dict]:
    """Remove near-duplicate examples."""
    seen = set()
    unique = []
    for ex in examples:
        # Simple dedup by instruction (use more sophisticated methods for production)
        key = ex["instruction"].lower().strip()
        if key not in seen:
            seen.add(key)
            unique.append(ex)
    return unique

# ═══════════════════════════════════════════════════════
# Step 3: Format for specific model template
# ═══════════════════════════════════════════════════════

def format_for_llama2_chat(example: Dict) -> str:
    """Format example in Llama 2 chat template."""
    system = "You are a helpful, professional customer support agent."
    user_msg = example["instruction"]
    if example.get("input"):
        user_msg += f"\n\nContext: {example['input']}"
    
    return (
        f"<s>[INST] <<SYS>>\n{system}\n<</SYS>>\n\n"
        f"{user_msg} [/INST] {example['output']}</s>"
    )

def format_for_chatml(example: Dict) -> str:
    """Format example in ChatML template (Mistral, etc.)."""
    system = "You are a helpful, professional customer support agent."
    user_msg = example["instruction"]
    if example.get("input"):
        user_msg += f"\n\nContext: {example['input']}"
    
    return (
        f"<|im_start|>system\n{system}<|im_end|>\n"
        f"<|im_start|>user\n{user_msg}<|im_end|>\n"
        f"<|im_start|>assistant\n{example['output']}<|im_end|>"
    )

# ═══════════════════════════════════════════════════════
# Step 4: Split and save dataset
# ═══════════════════════════════════════════════════════

def prepare_dataset(examples: List[Dict], model_type: str = "llama2"):
    """Complete pipeline: validate, clean, format, split."""
    
    # Validate
    valid_examples = [ex for ex in examples if validate_example(ex)]
    print(f"Valid examples: {len(valid_examples)}/{len(examples)}")
    
    # Deduplicate
    unique_examples = deduplicate(valid_examples)
    print(f"After dedup: {len(unique_examples)}")
    
    # Format based on model
    formatter = {
        "llama2": format_for_llama2_chat,
        "chatml": format_for_chatml
    }[model_type]
    
    formatted = [{"text": formatter(ex)} for ex in unique_examples]
    
    # Shuffle
    random.shuffle(formatted)
    
    # Split: 90% train, 10% validation
    split_idx = int(len(formatted) * 0.9)
    train_data = formatted[:split_idx]
    val_data = formatted[split_idx:]
    
    # Create Hugging Face datasets
    dataset = DatasetDict({
        "train": Dataset.from_list(train_data),
        "validation": Dataset.from_list(val_data)
    })
    
    print(f"Train: {len(train_data)} | Validation: {len(val_data)}")
    
    # Save to disk
    dataset.save_to_disk("./prepared_dataset")
    
    # Also save as JSONL for inspection
    with open("train.jsonl", "w") as f:
        for item in train_data:
            f.write(json.dumps(item) + "\n")
    
    return dataset

# ═══════════════════════════════════════════════════════
# Usage
# ═══════════════════════════════════════════════════════

raw_data = create_training_data()
dataset = prepare_dataset(raw_data, model_type="llama2")
```

### Dataset Size Guidelines

| Use Case | Minimum Examples | Recommended | Notes |
|----------|-----------------|-------------|-------|
| Style/Format change | 50-100 | 500-1000 | The model already "knows" the task |
| New domain knowledge | 500-1000 | 5000-10000 | Need enough coverage |
| New language | 1000-5000 | 10000-50000 | More complex adaptation |
| Classification task | 100-500 per class | 1000+ per class | Balance across classes |

---

## Training Configuration & Hyperparameters

### Critical Hyperparameters

| Parameter | Full FT | LoRA | QLoRA | Notes |
|-----------|---------|------|-------|-------|
| Learning Rate | 1e-5 to 5e-5 | 1e-4 to 3e-4 | 1e-4 to 3e-4 | LoRA can handle higher LR |
| Epochs | 1-3 | 1-5 | 1-5 | Watch for overfitting |
| Batch Size | 32-128 | 32-128 | 16-64 | Use gradient accumulation |
| Warmup | 3-10% of steps | 3-5% | 3-5% | Prevents early divergence |
| Weight Decay | 0.01-0.1 | 0.01 | 0.01 | Regularization |
| Max Grad Norm | 1.0 | 1.0 | 0.3 | Gradient clipping |
| LR Scheduler | Cosine | Cosine | Cosine | Smooth decay |

### Learning Rate: The Most Important Hyperparameter

```
Learning Rate Impact:
                                                    
Loss │  Too High LR           Just Right            Too Low LR
     │  ┌─┐                                        
     │  │ │ ╱\  /\           \                      \_________
     │  │ │/  \/  \           \                     
     │  │ │        \___        \____                (Never converges
     │  │ │                         \____            in reasonable time)
     │  │ │  (Diverges/             
     │  │ │   Oscillates)           (Converges well)
     │──┼─┼──────────────────────────────────────── Steps
     │  │ │
```

### Training Monitoring

```python
# Key metrics to watch during training

"""
1. Training Loss — Should decrease smoothly
   - If plateaus early: LR too low or data too easy
   - If oscillates wildly: LR too high
   - If increases: Something is very wrong

2. Validation Loss — The REAL metric
   - Should decrease with training loss
   - If it increases while train loss decreases → OVERFITTING
   - Stop training when val loss starts increasing

3. Learning Rate — Should follow your schedule
   - Warmup → Peak → Decay

4. Gradient Norm — Should be stable
   - Spikes indicate problematic batches
   - If consistently high: LR too high
"""

# Wandb logging example
import wandb

wandb.init(project="llm-finetuning", config={
    "model": "llama-2-7b",
    "method": "qlora",
    "rank": 64,
    "alpha": 128,
    "lr": 2e-4,
    "epochs": 3,
    "dataset_size": 5000
})

# The TrainingArguments with report_to="wandb" handles the rest
```

---

## Evaluation & Monitoring

### How to Evaluate Fine-Tuned Models

```python
# Evaluation framework for fine-tuned LLMs

import torch
from transformers import AutoModelForCausalLM, AutoTokenizer
import json

def evaluate_model(model, tokenizer, test_cases: list):
    """Evaluate fine-tuned model on test cases."""
    
    results = []
    for case in test_cases:
        prompt = case["prompt"]
        expected = case["expected_output"]
        
        # Generate response
        inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
        with torch.no_grad():
            outputs = model.generate(
                **inputs,
                max_new_tokens=256,
                temperature=0.1,        # Low temp for evaluation
                do_sample=False          # Greedy for reproducibility
            )
        
        response = tokenizer.decode(
            outputs[0][inputs["input_ids"].shape[1]:],  # Only new tokens
            skip_special_tokens=True
        )
        
        results.append({
            "prompt": prompt,
            "expected": expected,
            "generated": response,
            "match": evaluate_quality(response, expected)
        })
    
    # Calculate metrics
    accuracy = sum(r["match"] for r in results) / len(results)
    print(f"Accuracy: {accuracy:.2%}")
    return results

def evaluate_quality(generated: str, expected: str) -> float:
    """Score the quality of generated vs expected output."""
    # Simple exact match
    if generated.strip() == expected.strip():
        return 1.0
    
    # For more sophisticated evaluation, use:
    # - ROUGE scores (for summarization)
    # - BLEU scores (for translation)
    # - LLM-as-judge (GPT-4 rates the output)
    # - Human evaluation (gold standard)
    
    # Basic keyword overlap
    gen_words = set(generated.lower().split())
    exp_words = set(expected.lower().split())
    if not exp_words:
        return 0.0
    overlap = len(gen_words & exp_words) / len(exp_words)
    return overlap
```

### LLM-as-Judge Evaluation

```python
# Using a stronger model to evaluate your fine-tuned model
from openai import OpenAI

client = OpenAI()

def llm_judge(prompt: str, generated: str, criteria: str) -> dict:
    """Use GPT-4 as a judge to evaluate generated output."""
    
    judge_prompt = f"""Rate the following AI response on a scale of 1-5 for each criterion.

User Prompt: {prompt}

AI Response: {generated}

Criteria: {criteria}

Rate on:
1. Accuracy (1-5): Is the information correct?
2. Relevance (1-5): Does it address the question?
3. Completeness (1-5): Does it cover all aspects?
4. Format (1-5): Is it well-structured?
5. Tone (1-5): Is the tone appropriate?

Respond in JSON format: {{"accuracy": X, "relevance": X, "completeness": X, "format": X, "tone": X, "explanation": "..."}}
"""
    
    response = client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": judge_prompt}],
        temperature=0
    )
    
    return json.loads(response.choices[0].message.content)
```

---

## Common Mistakes

### 1. Training on Too Little Data Without Enough Diversity

**Mistake**: Using 50 examples that all look the same.
**Fix**: Even with 50 examples, ensure diversity in input types, lengths, and edge cases.

### 2. Wrong Learning Rate

**Mistake**: Using pre-training LR (1e-3) for fine-tuning.
**Fix**: Start with 2e-5 for full FT, 2e-4 for LoRA/QLoRA. Monitor loss carefully.

### 3. Overfitting to Training Data

**Mistake**: Training for too many epochs without validation monitoring.
**Fix**: Always use a validation set. Stop when validation loss increases for 2+ evaluations.

### 4. Incorrect Chat Template

**Mistake**: Training with one template, inferencing with another.
**Fix**: ALWAYS use the exact same prompt template for training and inference.

```python
# WRONG — mismatch between train and inference templates
# Training: "### Instruction:\n{text}\n### Response:\n{response}"
# Inference: "Question: {text}\nAnswer:"

# CORRECT — same template everywhere
TEMPLATE = "<s>[INST] {instruction} [/INST] {response}</s>"
```

### 5. Not Setting Padding Correctly

**Mistake**: Left-padding during training or wrong pad token.
**Fix**: Use right-padding for training, left-padding for batch inference.

### 6. Forgetting to Set Labels for Masked Positions

**Mistake**: Computing loss on the instruction/input tokens too.
**Fix**: Set labels to -100 for tokens you don't want the model to learn to predict.

### 7. Using QLoRA Rank Too Low

**Mistake**: Using r=4 and wondering why the model doesn't learn.
**Fix**: Start with r=16-64. Increase if the model isn't learning. Only decrease if overfitting.

---

## Interview Questions

### Conceptual Questions

**Q1: What is the difference between pre-training and fine-tuning?**
> Pre-training teaches a model general language understanding from massive unlabeled text (self-supervised). Fine-tuning adapts a pre-trained model to a specific task using smaller, labeled data (supervised).

**Q2: Explain LoRA. Why does it work?**
> LoRA decomposes weight updates into low-rank matrices (W = BA where B is d×r, A is r×k, r << d,k). It works because empirical studies show that weight updates during fine-tuning have low intrinsic rank — the model only needs small adjustments in a low-dimensional subspace.

**Q3: When would you choose full fine-tuning over LoRA?**
> Full fine-tuning when: you have massive data, unlimited compute, need maximum performance, or the task is very different from pre-training distribution. LoRA when: compute is limited, you need to serve multiple adapters, or the task is similar to what the model already knows.

**Q4: How do you prevent catastrophic forgetting?**
> Use low learning rates, few epochs, mix general data with task data, use LoRA (inherently less forgetting since base weights are frozen), early stopping based on validation loss on both task and general benchmarks.

**Q5: What is the role of the alpha parameter in LoRA?**
> Alpha (α) is a scaling factor. The effective LoRA contribution is scaled by α/r. It controls how much the LoRA update influences the output. Higher alpha = stronger adaptation. Typically set to 2× the rank.

### Practical Questions

**Q6: You fine-tuned a model but it's just repeating the training data. What happened?**
> Classic overfitting. Causes: too many epochs, too high LR, too little data, or not enough diversity. Fix: reduce epochs, add dropout, use larger/more diverse data, or reduce rank in LoRA.

**Q7: How do you evaluate a fine-tuned model?**
> Held-out test set with task-specific metrics (accuracy, ROUGE, BLEU), human evaluation for quality, LLM-as-judge for scalable evaluation, A/B testing in production, and checking for regression on general capabilities.

**Q8: Your QLoRA model performs significantly worse than full fine-tuning. Why and how to fix?**
> Possible causes: rank too low (increase r), wrong target modules (add more), quantization artifacts (try load_in_8bit instead), insufficient training (more epochs/data). Fix by ablating each factor.

---

## Quick Reference

### Fine-Tuning Method Selection

| Situation | Recommended Method |
|-----------|-------------------|
| Unlimited budget, maximum quality | Full Fine-Tuning |
| Single consumer GPU (16-24 GB) | QLoRA |
| Multiple tasks, need to swap adapters | LoRA |
| Very large model (70B+) | QLoRA |
| Production deployment (latency-critical) | LoRA → Merge |
| Experimentation/prototyping | QLoRA |

### Essential Commands Cheat Sheet

```python
# Load model for QLoRA
from transformers import BitsAndBytesConfig
bnb = BitsAndBytesConfig(load_in_4bit=True, bnb_4bit_quant_type="nf4",
                         bnb_4bit_compute_dtype=torch.bfloat16,
                         bnb_4bit_use_double_quant=True)

# LoRA Config
from peft import LoraConfig
config = LoraConfig(r=16, lora_alpha=32, target_modules=["q_proj","v_proj"],
                    lora_dropout=0.05, task_type="CAUSAL_LM")

# Apply LoRA
from peft import get_peft_model
model = get_peft_model(model, config)

# Save adapter only
model.save_pretrained("adapter_path")

# Load adapter
from peft import PeftModel
model = PeftModel.from_pretrained(base_model, "adapter_path")

# Merge for production
model = model.merge_and_unload()
```

### Hyperparameter Quick Reference

```
┌────────────────────────────────────────────────┐
│        RECOMMENDED STARTING VALUES             │
├────────────────────────────────────────────────┤
│  LoRA Rank:           16 (small task) → 64     │
│  LoRA Alpha:          2 × rank                 │
│  Learning Rate:       2e-4                     │
│  Epochs:              2-3                      │
│  Batch Size:          32 (with accumulation)   │
│  Warmup:              3% of total steps        │
│  Max Seq Length:       512-2048                 │
│  Scheduler:           Cosine                   │
│  Optimizer:           AdamW (paged for QLoRA)  │
│  Gradient Clipping:   1.0 (0.3 for QLoRA)     │
└────────────────────────────────────────────────┘
```

---

*Next Chapter: [05-RAG-Retrieval-Augmented-Generation](05-RAG-Retrieval-Augmented-Generation.md)*
