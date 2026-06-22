# Chapter 10: Advanced NLP Topics

## Table of Contents
- [Multilingual NLP](#multilingual-nlp)
- [Zero-Shot Learning](#zero-shot-learning)
- [Few-Shot Learning](#few-shot-learning)
- [Instruction Tuning](#instruction-tuning)
- [Parameter-Efficient Fine-Tuning (PEFT)](#parameter-efficient-fine-tuning-peft)
- [Constitutional AI and Alignment](#constitutional-ai-and-alignment)
- [Multimodal NLP](#multimodal-nlp)
- [Efficient NLP](#efficient-nlp)
- [NLP in Production](#nlp-in-production)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Multilingual NLP

### What It Is

Multilingual NLP builds systems that work across multiple languages — sometimes simultaneously. Instead of building separate models for English, French, Chinese, etc., a single model handles all of them.

### Simple Analogy

Imagine a translator who speaks 100 languages. Instead of hiring 100 separate translators, you have one person who can read, write, and reason in all of them. That's what multilingual models like mBERT and XLM-RoBERTa do. The remarkable finding: they can even transfer knowledge between languages they were never explicitly taught to connect.

### Why It Matters

- **5 billion+ internet users** speak languages other than English
- Most NLP training data is English-heavy — multilingual models bridge this gap
- **Cross-lingual transfer**: train on English, apply to Swahili (zero-shot!)
- Business applications need to work globally

### Key Multilingual Models

| Model | Languages | Parameters | Strengths |
|-------|-----------|-----------|-----------|
| **mBERT** | 104 languages | 110M | Pioneer, widely studied |
| **XLM-RoBERTa** | 100 languages | 270M/550M | State-of-the-art cross-lingual |
| **mT5** | 101 languages | 300M-13B | Generative multilingual |
| **BLOOM** | 46 languages | 176B | Open-source, diverse languages |
| **Aya** | 101 languages | 13B | Instruction-tuned multilingual |
| **Llama-3** | Many languages | 8B-405B | Strong multilingual despite English focus |

### Cross-Lingual Transfer

```
┌──────────────────────────────────────────────────┐
│           Cross-Lingual Transfer                   │
│                                                    │
│  Train on English NER data                         │
│        │                                           │
│        ▼                                           │
│  ┌──────────────────┐                              │
│  │  XLM-RoBERTa     │  Shared multilingual         │
│  │  (fine-tuned)     │  representation space        │
│  └──────────────────┘                              │
│        │                                           │
│        ├──→ Works on French (never trained!) ✓     │
│        ├──→ Works on German (never trained!) ✓     │
│        ├──→ Works on Hindi  (never trained!) ✓     │
│        └──→ Works on Japanese (reduced accuracy)   │
│                                                    │
│  Why? Languages share structural patterns that     │
│  the model learns to represent in a shared space   │
└──────────────────────────────────────────────────┘
```

### Code: Multilingual NER

```python
from transformers import pipeline

# Multilingual NER model
ner = pipeline(
    "ner",
    model="Davlan/xlm-roberta-large-ner-hrl",
    aggregation_strategy="simple"
)

# Test across multiple languages
texts = {
    "English": "Barack Obama was born in Honolulu, Hawaii.",
    "French": "Emmanuel Macron est le président de la France.",
    "German": "Angela Merkel lebte in Berlin.",
    "Spanish": "Gabriel García Márquez nació en Colombia.",
    "Chinese": "习近平访问了北京大学。",
    "Arabic": "محمد بن سلمان زار الرياض.",
}

for lang, text in texts.items():
    entities = ner(text)
    ent_str = ", ".join([f"{e['word']}({e['entity_group']})" for e in entities])
    print(f"  [{lang:8s}] {ent_str}")

# Output:
#   [English ] Barack Obama(PER), Honolulu(LOC), Hawaii(LOC)
#   [French  ] Emmanuel Macron(PER), France(LOC)
#   [German  ] Angela Merkel(PER), Berlin(LOC)
#   [Spanish ] Gabriel García Márquez(PER), Colombia(LOC)
#   [Chinese ] 习近平(PER), 北京大学(ORG)
#   [Arabic  ] محمد بن سلمان(PER), الرياض(LOC)
```

### Code: Multilingual Sentiment Analysis

```python
from transformers import pipeline

# Multilingual sentiment model
sentiment = pipeline(
    "sentiment-analysis",
    model="nlptown/bert-base-multilingual-uncased-sentiment"
)

reviews = [
    "This movie was absolutely fantastic!",           # English
    "Ce film était absolument fantastique!",          # French
    "Dieser Film war absolut fantastisch!",           # German
    "¡Esta película fue absolutamente fantástica!",   # Spanish
    "このの映画は本当に素晴らしかった！",                    # Japanese
]

for review in reviews:
    result = sentiment(review)
    stars = result[0]["label"]  # "1 star" to "5 stars"
    score = result[0]["score"]
    print(f"  {stars} ({score:.3f}) | {review[:50]}")
```

### Code: Machine Translation

```python
from transformers import pipeline

# Translation pipelines
en_to_fr = pipeline("translation", model="Helsinki-NLP/opus-mt-en-fr")
en_to_de = pipeline("translation", model="Helsinki-NLP/opus-mt-en-de")
en_to_zh = pipeline("translation", model="Helsinki-NLP/opus-mt-en-zh")

text = "Machine learning is transforming the healthcare industry."

print(f"English: {text}")
print(f"French:  {en_to_fr(text)[0]['translation_text']}")
print(f"German:  {en_to_de(text)[0]['translation_text']}")
print(f"Chinese: {en_to_zh(text)[0]['translation_text']}")

# Multi-language with mBART/NLLB
from transformers import AutoModelForSeq2SeqLM, AutoTokenizer

model_name = "facebook/nllb-200-distilled-600M"  # 200 languages!
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSeq2SeqLM.from_pretrained(model_name)

def translate(text, src_lang="eng_Latn", tgt_lang="fra_Latn"):
    """Translate between any of 200 languages"""
    tokenizer.src_lang = src_lang
    inputs = tokenizer(text, return_tensors="pt")
    
    translated = model.generate(
        **inputs,
        forced_bos_token_id=tokenizer.convert_tokens_to_ids(tgt_lang),
        max_length=256
    )
    return tokenizer.decode(translated[0], skip_special_tokens=True)

# English to Hindi
result = translate("Artificial intelligence is the future.", "eng_Latn", "hin_Deva")
print(f"Hindi: {result}")
```

### Challenges in Multilingual NLP

| Challenge | Description | Mitigation |
|-----------|-------------|------------|
| **Curse of multilinguality** | Performance drops as you add more languages | Increase model capacity |
| **Script diversity** | Chinese, Arabic, Thai have very different scripts | Byte-level or character-level tokenization |
| **Low-resource languages** | Little training data for many languages | Cross-lingual transfer, data augmentation |
| **Tokenization bias** | Subword tokenizers favor high-resource languages | Language-specific or balanced tokenizers |
| **Cultural context** | Idioms, humor, sarcasm vary by culture | Culture-aware training data |
| **Code-switching** | People mix languages in one sentence | Train on code-switched data |

---

## Zero-Shot Learning

### What It Is

Zero-shot learning means performing a task the model was **never explicitly trained on**, by leveraging its general understanding. Like asking a human who's never studied biology to answer a biology question — they use general knowledge and reasoning.

### How Zero-Shot Classification Works

```
Traditional: Train specifically on labeled sentiment data → Predict sentiment
Zero-Shot:   Use a model trained on NLI (natural language inference) → 
             Reframe ANY classification as an NLI problem

NLI Framing:
  Premise:   "I love this restaurant, the food was amazing!"
  Hypothesis: "This text is about positive sentiment"
  → Entailment? YES → Label: positive

  Premise:   "I love this restaurant, the food was amazing!"  
  Hypothesis: "This text is about food safety concerns"
  → Entailment? NO → Not this label
```

### Code: Zero-Shot Classification

```python
from transformers import pipeline

# Zero-shot classifier (based on NLI)
classifier = pipeline(
    "zero-shot-classification",
    model="facebook/bart-large-mnli"
)

# Classify WITHOUT any training data for these categories!
text = "The new iPhone has an incredible camera with 48MP sensor and improved night mode."

# Custom labels — can be ANYTHING
result = classifier(
    text,
    candidate_labels=["technology", "sports", "politics", "food", "photography"],
    multi_label=False  # True if text can belong to multiple categories
)

print(f"Text: {text[:60]}...")
for label, score in zip(result["labels"], result["scores"]):
    bar = "█" * int(score * 30)
    print(f"  {label:15s} {score:.3f} {bar}")

# Output:
#   technology      0.523 ███████████████
#   photography     0.412 ████████████
#   sports          0.032 █
#   food            0.019 
#   politics        0.014 
```

### Multi-Label Zero-Shot

```python
# A text can belong to multiple categories
text = "The new electric vehicle from Tesla features advanced AI autopilot and costs $35,000."

result = classifier(
    text,
    candidate_labels=["automotive", "technology", "business", "environment", "sports"],
    multi_label=True  # Independent classification per label
)

for label, score in zip(result["labels"], result["scores"]):
    tag = "✓" if score > 0.5 else "✗"
    print(f"  {tag} {label:15s} {score:.3f}")

# Output:
#   ✓ automotive      0.943
#   ✓ technology      0.891
#   ✓ business        0.724
#   ✓ environment     0.612
#   ✗ sports          0.023
```

### Zero-Shot with LLMs

```python
from transformers import pipeline

# Use instruction-tuned model for zero-shot tasks
generator = pipeline("text2text-generation", model="google/flan-t5-large")

# Zero-shot sentiment
prompt = """Classify the sentiment of this review as positive, negative, or neutral.

Review: "The hotel room was clean but the service was slow and unfriendly."
Sentiment:"""

result = generator(prompt, max_length=10)
print(f"Sentiment: {result[0]['generated_text']}")  # "negative"

# Zero-shot NER (no NER training needed!)
prompt = """Extract all person names and organizations from this text.

Text: "Sundar Pichai announced that Google will invest $10 billion in AI research, 
partnering with Stanford University and MIT."

Persons:"""

result = generator(prompt, max_length=50)
print(result[0]['generated_text'])  # "Sundar Pichai"

# Zero-shot translation
prompt = """Translate to French: "The weather is beautiful today."
French:"""
result = generator(prompt, max_length=50)
print(result[0]['generated_text'])
```

---

## Few-Shot Learning

### What It Is

Few-shot learning performs tasks with just **a handful of examples** (typically 1-32) instead of thousands. The model learns the task pattern from the examples provided in the prompt.

### Terminology

| Term | # Examples | Description |
|------|-----------|-------------|
| **Zero-shot** | 0 | Only instructions, no examples |
| **One-shot** | 1 | Single example |
| **Few-shot** | 2-32 | A handful of examples |
| **Many-shot** | 32+ | More examples (with large context windows) |

### In-Context Learning (ICL)

```python
from transformers import pipeline

generator = pipeline("text-generation", model="gpt2-large")

# Few-shot sentiment classification via in-context learning
prompt = """Classify each review's sentiment.

Review: "Absolutely loved it, best purchase ever!"
Sentiment: positive

Review: "Terrible quality, broke after one day."
Sentiment: negative

Review: "It's okay, nothing special but works fine."
Sentiment: neutral

Review: "The battery life is impressive but the screen is too small."
Sentiment:"""

result = generator(prompt, max_new_tokens=5, do_sample=False)
# Extracts "neutral" or "mixed" by pattern matching from examples
```

### Few-Shot with SetFit (Efficient Few-Shot)

```python
# pip install setfit
from setfit import SetFitModel, SetFitTrainer, sample_dataset
from datasets import load_dataset

# Load dataset
dataset = load_dataset("sst2")

# Sample only 8 examples per class (few-shot!)
train_dataset = sample_dataset(dataset["train"], label_column="label", num_samples=8)
print(f"Training with only {len(train_dataset)} examples!")  # 16 total

# Create SetFit model
model = SetFitModel.from_pretrained("sentence-transformers/paraphrase-mpnet-base-v2")

# Train
trainer = SetFitTrainer(
    model=model,
    train_dataset=train_dataset,
    eval_dataset=dataset["validation"],
    metric="accuracy",
    column_mapping={"sentence": "text", "label": "label"}
)

trainer.train()

# Evaluate
metrics = trainer.evaluate()
print(f"Accuracy with 16 examples: {metrics['accuracy']:.3f}")
# Typically 85-90%+ accuracy with just 16 examples!

# Predict
predictions = model.predict([
    "This movie was fantastic!",
    "Worst experience of my life.",
    "It was an average film, nothing memorable."
])
print(predictions)  # [1, 0, 0]
```

### Few-Shot vs Fine-Tuning Trade-offs

| Aspect | Few-Shot (ICL) | Full Fine-Tuning |
|--------|---------------|------------------|
| **Data needed** | 1-32 examples | 1000s of examples |
| **Training time** | None (inference only) | Hours/days |
| **Model changes** | None | Weights updated |
| **Flexibility** | Change task instantly | Retrain for new tasks |
| **Quality** | Good for simple tasks | Best for complex tasks |
| **Cost** | Higher inference cost (long prompts) | Higher training cost |
| **Consistency** | Can vary with prompt wording | More stable |

---

## Instruction Tuning

### What It Is

Instruction tuning trains a model to follow natural language instructions. Instead of learning one specific task, the model learns the meta-skill of "following directions." It's what turns a raw language model (that just predicts next words) into an assistant (that does what you ask).

### The Journey of a Language Model

```
Stage 1: Pre-training (Base Model)
  └─ Learns language patterns from internet text
  └─ Can complete text but can't follow instructions well
  └─ "What is 2+2?" → "What is 2+3? What is 2+4? What is..."
  
Stage 2: Instruction Tuning (Instruct Model)
  └─ Trained on (instruction, response) pairs
  └─ Learns to follow diverse instructions
  └─ "What is 2+2?" → "4"
  
Stage 3: RLHF/DPO (Aligned Model)
  └─ Trained with human preferences
  └─ Learns to be helpful, harmless, honest
  └─ "What is 2+2?" → "2+2 equals 4."
```

### How Instruction Tuning Works

```python
# Training data format for instruction tuning
instruction_data = [
    {
        "instruction": "Summarize the following text in one sentence.",
        "input": "The Amazon rainforest covers 5.5 million km² and spans nine countries...",
        "output": "The Amazon rainforest is a massive tropical forest covering 5.5M km² across nine South American nations."
    },
    {
        "instruction": "Translate to French.",
        "input": "Hello, how are you?",
        "output": "Bonjour, comment allez-vous ?"
    },
    {
        "instruction": "Classify the sentiment as positive, negative, or neutral.",
        "input": "The movie was okay but nothing special.",
        "output": "neutral"
    },
    {
        "instruction": "Write a Python function that reverses a string.",
        "input": "",
        "output": "def reverse_string(s):\n    return s[::-1]"
    },
    {
        "instruction": "Extract all named entities from the text.",
        "input": "Apple CEO Tim Cook visited London yesterday.",
        "output": "Apple (ORG), Tim Cook (PER), London (LOC), yesterday (DATE)"
    }
]

# The model learns: given ANY instruction + input → produce appropriate output
# Key: diversity of tasks matters more than volume per task
```

### Instruction Tuning with HuggingFace

```python
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    TrainingArguments,
    Trainer,
    DataCollatorForSeq2Seq
)
from datasets import load_dataset

# Load an instruction dataset
dataset = load_dataset("tatsu-lab/alpaca")  # 52K instruction examples

model_name = "meta-llama/Llama-2-7b-hf"
tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token

# Format instructions into a template
TEMPLATE = """### Instruction:
{instruction}

### Input:
{input}

### Response:
{output}"""

def format_instruction(example):
    """Format a single example into the instruction template"""
    text = TEMPLATE.format(
        instruction=example["instruction"],
        input=example.get("input", ""),
        output=example["output"]
    )
    return {"text": text}

# Apply formatting
formatted_dataset = dataset.map(format_instruction)

def tokenize_function(examples):
    """Tokenize and prepare labels (causal LM training)"""
    result = tokenizer(
        examples["text"],
        truncation=True,
        max_length=512,
        padding="max_length"
    )
    result["labels"] = result["input_ids"].copy()
    return result

tokenized = formatted_dataset.map(tokenize_function, batched=True)

# Training (would use PEFT in practice for large models)
training_args = TrainingArguments(
    output_dir="./instruct-model",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=8,  # Effective batch size = 32
    learning_rate=2e-5,
    warmup_ratio=0.03,
    fp16=True,
    logging_steps=50,
    save_strategy="epoch",
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized["train"],
    tokenizer=tokenizer,
)

trainer.train()
```

### Key Instruction Tuning Datasets

| Dataset | Size | Source | Description |
|---------|------|--------|-------------|
| **FLAN** (Google) | 1.8K tasks | Research tasks | Diverse NLP tasks with templates |
| **Alpaca** (Stanford) | 52K | GPT-generated | General instructions |
| **Dolly** (Databricks) | 15K | Human-written | High quality, commercially licensed |
| **OpenAssistant** | 160K | Crowdsourced | Multi-turn conversations |
| **ShareGPT** | 90K+ | User-shared | Real ChatGPT conversations |
| **UltraChat** | 1.5M | Synthetic | Large-scale multi-turn |
| **Aya Dataset** | 204K | 119 languages | Multilingual instructions |

---

## Parameter-Efficient Fine-Tuning (PEFT)

### What It Is

PEFT methods fine-tune only a **tiny fraction** of model parameters (0.1-5%) while keeping the rest frozen. This makes fine-tuning large models feasible on consumer hardware.

### Why PEFT

Full fine-tuning of a 7B model requires:
- ~28GB VRAM (FP16 weights)
- ~56GB for optimizer states (Adam)
- ~28GB for gradients
- Total: ~112GB — needs multiple expensive GPUs!

PEFT reduces this to **6-12GB** — fits on a single consumer GPU.

### LoRA (Low-Rank Adaptation) — The Most Popular Method

```
Original weight matrix W (d × d):
┌─────────────────────────────┐
│                             │   d=4096
│         W (frozen)          │   Parameters: 4096 × 4096 = 16.7M
│                             │
└─────────────────────────────┘

LoRA decomposition: W' = W + BA
┌─────────────────────────────┐   ┌───┐   ┌─────────────────────────────┐
│                             │   │   │   │                             │
│      W (frozen, d×d)        │ + │ B │ × │        A (r × d)            │
│                             │   │d×r│   │                             │
└─────────────────────────────┘   └───┘   └─────────────────────────────┘
     16.7M params (frozen)      r=16      Trainable: 2 × 4096 × 16 = 131K
                                          That's <1% of original parameters!
```

$$W' = W + \frac{\alpha}{r} \cdot BA$$

Where:
- $W$ = original frozen weights
- $B \in \mathbb{R}^{d \times r}$, $A \in \mathbb{R}^{r \times d}$ = low-rank trainable matrices
- $r$ = rank (typically 4-64, lower = fewer params)
- $\alpha$ = scaling factor (controls adaptation strength)

### LoRA Implementation

```python
# pip install peft
from peft import LoraConfig, get_peft_model, TaskType
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

# Load base model
model_name = "meta-llama/Llama-2-7b-hf"
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.float16,
    device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained(model_name)

# Configure LoRA
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,                  # Rank — lower = fewer params, higher = more capacity
    lora_alpha=32,         # Scaling factor (often 2×r)
    lora_dropout=0.05,     # Dropout for regularization
    target_modules=[       # Which layers to apply LoRA to
        "q_proj",          # Query projection in attention
        "k_proj",          # Key projection
        "v_proj",          # Value projection
        "o_proj",          # Output projection
        # "gate_proj",     # Optional: MLP layers too
        # "up_proj",
        # "down_proj",
    ],
    bias="none",           # Don't train biases
)

# Apply LoRA
model = get_peft_model(model, lora_config)

# Check parameter efficiency
model.print_trainable_parameters()
# Output: trainable params: 4,194,304 || all params: 6,742,609,920 || trainable%: 0.0622
# Only 0.06% of parameters are trainable!

# Train normally with HuggingFace Trainer
# The frozen weights stay as-is, only LoRA matrices are updated
```

### QLoRA (Quantized LoRA)

```python
# QLoRA = 4-bit quantized base model + LoRA adapters
# Enables fine-tuning 7B models on a single 8GB GPU!

from transformers import AutoModelForCausalLM, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training

# 4-bit quantization config
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",           # NormalFloat4 quantization
    bnb_4bit_compute_dtype=torch.float16, # Compute in FP16
    bnb_4bit_use_double_quant=True,       # Nested quantization
)

# Load quantized model
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=bnb_config,
    device_map="auto"
)

# Prepare for training
model = prepare_model_for_kbit_training(model)

# Apply LoRA on top of quantized model
lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type=TaskType.CAUSAL_LM,
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Now fits in ~6GB VRAM!
```

### Other PEFT Methods

| Method | How It Works | Params Trained | Best For |
|--------|-------------|---------------|----------|
| **LoRA** | Low-rank matrices added to attention | 0.1-1% | General fine-tuning |
| **QLoRA** | LoRA on 4-bit quantized model | 0.1-1% | Memory-constrained |
| **Prefix Tuning** | Learnable tokens prepended to each layer | 0.1% | Generation tasks |
| **Prompt Tuning** | Learnable soft prompt prepended to input | <0.1% | Classification |
| **IA3** | Learned vectors that rescale activations | 0.01% | Minimal overhead |
| **Adapters** | Small bottleneck layers between transformer layers | 1-5% | Multi-task learning |

### Merging LoRA Adapters

```python
from peft import PeftModel

# Load base model
base_model = AutoModelForCausalLM.from_pretrained(model_name)

# Load and merge LoRA adapter
model = PeftModel.from_pretrained(base_model, "path/to/lora/adapter")

# Merge LoRA weights into base model (no inference overhead!)
merged_model = model.merge_and_unload()

# Save the merged model
merged_model.save_pretrained("./merged-model")
tokenizer.save_pretrained("./merged-model")
# Now it's a regular model with no PEFT dependency needed for inference
```

---

## Constitutional AI and Alignment

### What It Is

Alignment ensures AI systems behave in ways that are helpful, harmless, and honest. It's about making models do what humans actually want, not just what they're literally asked.

### RLHF (Reinforcement Learning from Human Feedback)

```
Step 1: Supervised Fine-Tuning (SFT)
  ─ Train on high-quality (instruction, response) pairs
  ─ Model learns to follow instructions

Step 2: Reward Model Training
  ─ Humans rank multiple responses to the same prompt
  ─ Train a model to predict which response humans prefer
  ─ "Response A > Response B" → reward model learns human preferences

Step 3: RL Optimization (PPO)
  ─ Use the reward model as the objective
  ─ Optimize the policy (language model) to maximize reward
  ─ KL divergence penalty prevents model from straying too far from SFT

┌──────────┐    ┌──────────────┐    ┌─────────────┐
│  Prompt  │───>│  LM (Policy) │───>│  Response    │
└──────────┘    └──────────────┘    └──────┬──────┘
                      ▲                     │
                      │                     ▼
                      │              ┌──────────────┐
                      └──────────────│ Reward Model │
                    Update weights   │ (Human prefs)│
                    to maximize      └──────────────┘
                    reward
```

### DPO (Direct Preference Optimization)

A simpler alternative to RLHF that skips the reward model:

```python
# DPO directly optimizes the policy using preference pairs
# No separate reward model needed!

# Training data format:
preferences = [
    {
        "prompt": "Explain quantum computing simply.",
        "chosen": "Quantum computing uses quantum bits (qubits) that can be 0, 1, or both at once...",
        "rejected": "Quantum computing is a type of computation that harnesses quantum mechanical phenomena..."
    },
    # ... more preference pairs
]

# DPO Loss:
# L_DPO = -E[log σ(β(log π_θ(y_w|x)/π_ref(y_w|x) - log π_θ(y_l|x)/π_ref(y_l|x)))]
# Where y_w = chosen (winning) response, y_l = rejected (losing) response
```

```python
# DPO training with TRL library
# pip install trl
from trl import DPOTrainer, DPOConfig
from transformers import AutoModelForCausalLM, AutoTokenizer
from datasets import load_dataset

# Load model and reference model
model = AutoModelForCausalLM.from_pretrained("your-sft-model")
ref_model = AutoModelForCausalLM.from_pretrained("your-sft-model")  # Frozen copy
tokenizer = AutoTokenizer.from_pretrained("your-sft-model")

# Load preference dataset
dataset = load_dataset("Anthropic/hh-rlhf")

# DPO training config
training_args = DPOConfig(
    output_dir="./dpo-model",
    num_train_epochs=1,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    learning_rate=5e-7,        # Very small LR for alignment
    beta=0.1,                   # KL penalty strength
    max_length=512,
    max_prompt_length=256,
    fp16=True,
)

# Train
trainer = DPOTrainer(
    model=model,
    ref_model=ref_model,
    args=training_args,
    train_dataset=dataset["train"],
    tokenizer=tokenizer,
)

trainer.train()
```

### Constitutional AI (Anthropic's Approach)

```
┌─────────────────────────────────────────────────────┐
│  Constitutional AI: Self-Improvement Loop            │
│                                                      │
│  1. Model generates response to a harmful prompt     │
│  2. Model critiques its own response using           │
│     constitutional principles                        │
│  3. Model revises based on its critique              │
│  4. Train on the improved responses                  │
│                                                      │
│  Principles (Constitution):                          │
│  • "Choose the response that is most helpful"        │
│  • "Choose the response that is least harmful"       │
│  • "Choose the response that is most honest"         │
│  • "Choose the response that avoids stereotypes"     │
└─────────────────────────────────────────────────────┘
```

---

## Multimodal NLP

### What It Is

Multimodal NLP combines text with other modalities — images, audio, video — enabling models to understand and generate across different types of data.

### Key Multimodal Tasks

| Task | Input | Output | Example Model |
|------|-------|--------|---------------|
| **Image Captioning** | Image | Text description | BLIP-2, LLaVA |
| **Visual QA** | Image + Question | Text answer | LLaVA, GPT-4V |
| **Text-to-Image** | Text prompt | Image | DALL-E, Stable Diffusion |
| **Speech-to-Text** | Audio | Text | Whisper |
| **Text-to-Speech** | Text | Audio | Bark, XTTS |
| **Video Understanding** | Video + Question | Text | Video-LLaMA |
| **Document QA** | Document image | Text answer | LayoutLM, Donut |

### Code: Image Captioning

```python
from transformers import pipeline

# Image captioning
captioner = pipeline("image-to-text", model="Salesforce/blip-image-captioning-base")

# From URL
result = captioner("https://upload.wikimedia.org/wikipedia/commons/4/47/PNG_transparency_demonstration_1.png")
print(f"Caption: {result[0]['generated_text']}")

# From local file
# result = captioner("./my_image.jpg")
```

### Code: Visual Question Answering

```python
from transformers import pipeline

# Visual QA
vqa = pipeline("visual-question-answering", model="dandelin/vilt-b32-finetuned-vqa")

image_url = "https://example.com/dog_park.jpg"
questions = [
    "What animal is in the image?",
    "What color is the dog?",
    "Is it outdoors or indoors?",
]

for q in questions:
    result = vqa(image=image_url, question=q)
    print(f"Q: {q}")
    print(f"A: {result[0]['answer']} (confidence: {result[0]['score']:.3f})")
```

### Code: Speech-to-Text with Whisper

```python
from transformers import pipeline

# Speech recognition (supports 100+ languages)
whisper = pipeline(
    "automatic-speech-recognition",
    model="openai/whisper-base",
    device=0  # GPU
)

# Transcribe audio
result = whisper("audio_file.mp3")
print(f"Transcription: {result['text']}")

# With timestamps
result = whisper(
    "audio_file.mp3",
    return_timestamps=True,
    chunk_length_s=30  # Process in 30-second chunks
)

for chunk in result["chunks"]:
    start, end = chunk["timestamp"]
    print(f"[{start:.1f}s - {end:.1f}s] {chunk['text']}")

# Translate (any language → English)
result = whisper("french_audio.mp3", generate_kwargs={"task": "translate"})
print(f"Translation: {result['text']}")
```

### Code: Document Understanding

```python
from transformers import pipeline

# Document QA (works on images of documents, receipts, forms)
doc_qa = pipeline(
    "document-question-answering",
    model="impira/layoutlm-document-qa"
)

# Ask questions about a document image
result = doc_qa(
    image="invoice.png",
    question="What is the total amount?"
)
print(f"Total: {result[0]['answer']}")

result = doc_qa(
    image="invoice.png",
    question="What is the invoice date?"
)
print(f"Date: {result[0]['answer']}")
```

---

## Efficient NLP

### Knowledge Distillation

Train a small "student" model to mimic a large "teacher" model:

```python
"""
Knowledge Distillation: Transfer knowledge from large to small model.

Teacher (BERT-large, 340M params) → Student (DistilBERT, 66M params)
Student learns from teacher's soft probability outputs, not just hard labels.
"""
import torch
import torch.nn as nn
import torch.nn.functional as F

class DistillationLoss(nn.Module):
    def __init__(self, temperature=4.0, alpha=0.5):
        """
        temperature: Higher = softer probabilities (more knowledge transfer)
        alpha: Balance between distillation loss and task loss
        """
        super().__init__()
        self.temperature = temperature
        self.alpha = alpha
        self.kl_div = nn.KLDivLoss(reduction="batchmean")
    
    def forward(self, student_logits, teacher_logits, labels):
        # Soft distillation loss: KL divergence between soft distributions
        soft_student = F.log_softmax(student_logits / self.temperature, dim=-1)
        soft_teacher = F.softmax(teacher_logits / self.temperature, dim=-1)
        distill_loss = self.kl_div(soft_student, soft_teacher) * (self.temperature ** 2)
        
        # Hard label loss: standard cross-entropy
        hard_loss = F.cross_entropy(student_logits, labels)
        
        # Combined loss
        total_loss = self.alpha * distill_loss + (1 - self.alpha) * hard_loss
        return total_loss

# Popular distilled models:
# DistilBERT: 40% smaller, 60% faster, 97% of BERT performance
# TinyBERT: 7.5x smaller than BERT
# MiniLM: Distilled from large transformers, great quality/size ratio
```

### Model Quantization

```python
# Post-training quantization with PyTorch
import torch
from transformers import AutoModelForSequenceClassification

# Load model
model = AutoModelForSequenceClassification.from_pretrained("bert-base-cased")

# Dynamic quantization (easiest, no calibration data needed)
quantized_model = torch.quantization.quantize_dynamic(
    model,
    {torch.nn.Linear},  # Quantize linear layers
    dtype=torch.qint8    # 8-bit integers
)

# Compare sizes
import os
torch.save(model.state_dict(), "original.pt")
torch.save(quantized_model.state_dict(), "quantized.pt")

orig_size = os.path.getsize("original.pt") / 1e6
quant_size = os.path.getsize("quantized.pt") / 1e6
print(f"Original: {orig_size:.1f}MB | Quantized: {quant_size:.1f}MB")
print(f"Compression: {orig_size/quant_size:.1f}x")
# Typical: ~4x smaller with <1% accuracy loss
```

### Pruning

```python
# Remove unimportant weights (set to zero)
import torch.nn.utils.prune as prune

model = AutoModelForSequenceClassification.from_pretrained("bert-base-cased")

# Prune 30% of weights in all linear layers
for name, module in model.named_modules():
    if isinstance(module, torch.nn.Linear):
        prune.l1_unstructured(module, name="weight", amount=0.3)

# Check sparsity
total_params = 0
zero_params = 0
for name, param in model.named_parameters():
    total_params += param.numel()
    zero_params += (param == 0).sum().item()

print(f"Sparsity: {zero_params/total_params:.1%}")  # ~30%
```

### Comparison of Efficiency Techniques

| Technique | Size Reduction | Speed Improvement | Quality Loss | Effort |
|-----------|---------------|-------------------|-------------|--------|
| Distillation | 2-10x smaller | 2-5x faster | 1-5% | High |
| Quantization (INT8) | 2-4x smaller | 1.5-3x faster | <1% | Low |
| Quantization (INT4) | 4-8x smaller | 2-4x faster | 1-3% | Low |
| Pruning | 2-5x smaller | Variable | 1-5% | Medium |
| ONNX export | Same size | 1.5-3x faster | None | Low |

---

## NLP in Production

### End-to-End NLP Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Production NLP System                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
│  │  Input    │→│  Pre-     │→│  Model    │→│  Post-         │  │
│  │  (API)    │  │  process  │  │  Inference│  │  processing    │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────┬───────┘  │
│       │              │            │                   │          │
│       │         ┌────▼────┐  ┌───▼──────┐      ┌────▼────┐    │
│       │         │Tokenize │  │GPU/CPU   │      │Format   │    │
│       │         │Truncate │  │Batching  │      │Filter   │    │
│       │         │Validate │  │Caching   │      │Validate │    │
│       │         └─────────┘  └──────────┘      └─────────┘    │
│       │                                                        │
│  ┌────▼────────────────────────────────────────────────────┐   │
│  │  Supporting Infrastructure                               │   │
│  │  • Model versioning (MLflow, W&B)                        │   │
│  │  • Monitoring & alerting (Prometheus, Grafana)           │   │
│  │  • A/B testing framework                                 │   │
│  │  • Data pipeline (feature store, embedding cache)        │   │
│  │  • Logging & audit trail                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Production Serving Example

```python
"""
FastAPI-based NLP service with proper production patterns.
"""
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from transformers import pipeline
from functools import lru_cache
import logging
import time

app = FastAPI(title="NLP Service")
logger = logging.getLogger(__name__)

# Model loading (singleton pattern)
@lru_cache(maxsize=1)
def get_model():
    """Load model once, cache in memory"""
    logger.info("Loading NLP model...")
    return pipeline("sentiment-analysis", model="distilbert-base-uncased-finetuned-sst-2-english")

class PredictionRequest(BaseModel):
    text: str = Field(..., min_length=1, max_length=5000)

class PredictionResponse(BaseModel):
    label: str
    confidence: float
    latency_ms: float

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    """Classify sentiment of input text"""
    start = time.time()
    
    try:
        model = get_model()
        result = model(request.text[:512])[0]  # Truncate for safety
        
        latency = (time.time() - start) * 1000
        
        return PredictionResponse(
            label=result["label"],
            confidence=round(result["score"], 4),
            latency_ms=round(latency, 2)
        )
    except Exception as e:
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail="Model inference failed")

@app.get("/health")
async def health():
    """Health check endpoint"""
    model = get_model()
    # Quick sanity check
    result = model("test")
    return {"status": "healthy", "model_loaded": result is not None}
```

### Key Production Considerations

| Concern | Solution |
|---------|----------|
| **Latency** | Model quantization, distillation, caching, batching |
| **Throughput** | Async serving, dynamic batching, GPU optimization |
| **Reliability** | Health checks, fallback models, circuit breakers |
| **Monitoring** | Track accuracy drift, latency p99, error rates |
| **Safety** | Input validation, output filtering, rate limiting |
| **Cost** | Right-size models, spot instances, edge deployment |
| **Updates** | Blue-green deployment, A/B testing, gradual rollout |

---

## Common Mistakes

### 1. Using English-Only Models for Multilingual Tasks
```python
# WRONG — BERT is English-only
model = AutoModel.from_pretrained("bert-base-uncased")
# Fails on French, Chinese, Arabic text

# RIGHT — use multilingual model
model = AutoModel.from_pretrained("xlm-roberta-base")  # 100 languages
```

### 2. Misunderstanding Zero-Shot Capabilities
```python
# WRONG — expecting perfect zero-shot accuracy
# Zero-shot is a fallback when you have NO labeled data
# Always compare: zero-shot vs few-shot vs fine-tuned

# RIGHT — use zero-shot as a starting point, then improve
# Step 1: Try zero-shot (baseline)
# Step 2: If insufficient, try few-shot (5-20 examples)
# Step 3: If still insufficient, fine-tune with PEFT
```

### 3. LoRA Rank Too High or Too Low
```python
# WRONG — rank too high wastes resources, may overfit
lora_config = LoraConfig(r=256)  # Way too high for most tasks

# WRONG — rank too low loses capacity
lora_config = LoraConfig(r=1)  # Too constrained

# RIGHT — start with r=16, adjust based on results
# Simple tasks (classification): r=8
# Medium tasks (QA, summarization): r=16-32
# Complex tasks (instruction tuning): r=32-64
```

### 4. Not Applying LoRA to Enough Layers
```python
# WRONG — only applying to query projection
lora_config = LoraConfig(target_modules=["q_proj"])

# RIGHT — apply to all attention projections (at minimum)
lora_config = LoraConfig(
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"]
)
# Even better: include MLP layers too for complex tasks
```

### 5. Forgetting to Merge LoRA Before Deployment
```python
# WRONG — deploying with PEFT wrapper (adds latency)
model = PeftModel.from_pretrained(base_model, "adapter_path")
# Every forward pass has overhead from the adapter computation

# RIGHT — merge and save for deployment
model = PeftModel.from_pretrained(base_model, "adapter_path")
model = model.merge_and_unload()
model.save_pretrained("./production-model")
# Zero inference overhead — just a regular model
```

### 6. Ignoring Tokenizer Fertility Across Languages
```python
# Problem: English "Hello" = 1 token, Thai "สวัสดี" = 5+ tokens
# This means: same text in different languages produces different numbers of tokens
# Implications:
# - Thai text hits max_length faster → more truncation
# - Thai inference is slower (more tokens to process)
# - Multilingual models have unequal capacity per language

# Mitigation: Use language-balanced tokenizers (XLM-RoBERTa)
# Or increase max_length for languages with high fertility
```

---

## Interview Questions

### Conceptual

**Q1: What is cross-lingual transfer and why does it work?**
> Cross-lingual transfer means training a model on one language (e.g., English) and applying it to another (e.g., Hindi) without any target-language training data. It works because multilingual models learn shared representations: structurally similar concepts across languages map to similar vector spaces. This emerges from shared vocabulary (cognates, borrowed words), shared subword tokens, and common syntactic patterns that the model discovers during pre-training on 100+ languages.

**Q2: Explain the difference between zero-shot, few-shot, and fine-tuning. When would you use each?**
> **Zero-shot**: No task-specific examples. Use when you have no labeled data and need a quick baseline, or when the task maps naturally to NLI. **Few-shot**: 1-32 examples in the prompt. Use when you have minimal labeled data and the task is pattern-matchable. **Fine-tuning**: Update model weights on labeled data. Use when you need maximum accuracy, have sufficient data (100+), and the task is well-defined. In practice: start zero-shot → try few-shot → fine-tune with PEFT if needed.

**Q3: What is LoRA and why is it important?**
> LoRA (Low-Rank Adaptation) adds small trainable matrices to frozen model weights. Instead of updating all 7B parameters, you train <0.1% of parameters while achieving 90-99% of full fine-tuning performance. It's important because it democratizes LLM customization — you can fine-tune a 7B model on a single consumer GPU (8-16GB), store multiple task-specific adapters efficiently (each ~10-50MB vs 14GB), and switch between tasks by swapping adapters.

**Q4: What is instruction tuning and how does it differ from standard pre-training?**
> Pre-training teaches a model to predict the next token from internet text — it learns language but not how to be helpful. Instruction tuning trains on (instruction, response) pairs across diverse tasks, teaching the model the meta-skill of following directions. The key insight: a model trained on "Translate X to French", "Summarize this article", "Write code for X" learns to follow novel instructions it hasn't seen. It's the difference between a language model and an assistant.

**Q5: Explain DPO and how it simplifies RLHF.**
> RLHF requires: (1) training a separate reward model, (2) complex RL optimization with PPO. DPO eliminates both by deriving a closed-form loss directly from preference data. Given pairs (chosen_response, rejected_response), DPO optimizes the policy model to increase the probability of chosen responses relative to rejected ones, while staying close to a reference model. The math shows that optimizing this objective is equivalent to optimizing the RLHF objective, but it's simpler, more stable, and requires fewer hyperparameters.

### Practical

**Q6: How would you build a chatbot that works in 50 languages with limited resources?**
> 1. Start with a strong multilingual base model (XLM-RoBERTa or multilingual LLM)
> 2. Use English instruction data + machine-translated instructions for diversity
> 3. Fine-tune with QLoRA to fit on limited hardware
> 4. Apply cross-lingual transfer — fine-tune primarily on English + a few high-resource languages
> 5. Evaluate on all 50 languages using multilingual benchmarks
> 6. For low-resource languages with poor performance, add 100-200 examples
> 7. Use language detection to route to language-specific fallback if quality is too low

**Q7: Your model's accuracy drops 5% after deploying to production. How do you diagnose?**
> 1. **Data drift**: Compare production input distribution to training data (vocabulary, length, topics)
> 2. **Preprocessing mismatch**: Verify tokenization, cleaning, encoding are identical to training
> 3. **Label distribution shift**: Check if class balance changed
> 4. **Adversarial inputs**: Look for unusual/malicious inputs
> 5. **Infrastructure**: Verify model version, quantization, hardware differences
> 6. **Evaluation gap**: Ensure production metrics match offline evaluation methodology
> 7. Implement ongoing monitoring with alert thresholds on key metrics

**Q8: Compare LoRA vs full fine-tuning vs prompt engineering for adapting an LLM to a new domain.**
> **Prompt engineering**: Zero cost, instant, but limited by context window and inconsistent results. Best for simple, well-defined tasks. **LoRA**: Minimal compute (~1 GPU), trains 0.1% of params, achieves 90-99% of full fine-tuning quality. Best balance of cost and quality. **Full fine-tuning**: Highest quality but needs multiple GPUs, risks catastrophic forgetting, produces a full model copy per task. Choose based on: data availability, quality requirements, compute budget, and how many tasks you need to support.

---

## Quick Reference

### Multilingual Model Selection

| Need | Model | Languages | Size |
|------|-------|-----------|------|
| Embeddings | `xlm-roberta-base` | 100 | 270M |
| Classification | `xlm-roberta-large` | 100 | 550M |
| Generation | `bigscience/bloom-7b1` | 46 | 7B |
| Translation | `facebook/nllb-200-distilled-600M` | 200 | 600M |
| Speech | `openai/whisper-large-v3` | 100+ | 1.5B |

### PEFT Method Selection

| Method | When to Use | Trainable % | Memory |
|--------|-------------|-------------|--------|
| **Full Fine-Tune** | Lots of data + compute | 100% | Very high |
| **LoRA** | General PEFT, most tasks | 0.1-1% | Medium |
| **QLoRA** | Limited GPU memory | 0.1-1% | Low |
| **Prompt Tuning** | Simple classification | <0.01% | Very low |
| **Prefix Tuning** | Generation tasks | ~0.1% | Low |
| **Adapters** | Multi-task deployment | 1-5% | Medium |

### LoRA Hyperparameter Guide

| Hyperparameter | Range | Guidance |
|---------------|-------|----------|
| `r` (rank) | 4-64 | Start with 16; increase for complex tasks |
| `lora_alpha` | r to 2×r | Typically 2×r (e.g., r=16, alpha=32) |
| `lora_dropout` | 0.0-0.1 | 0.05 is a safe default |
| `target_modules` | Varies | At minimum: q,k,v,o projections |
| Learning rate | 1e-4 to 3e-4 | Higher than full fine-tuning |
| Epochs | 1-5 | Watch for overfitting on small data |

### Alignment Methods Comparison

| Method | Complexity | Data Needed | Stability | Quality |
|--------|-----------|-------------|-----------|---------|
| **SFT** | Low | Instruction pairs | High | Good |
| **RLHF (PPO)** | Very high | Preferences + reward model | Low | Best |
| **DPO** | Medium | Preference pairs | High | Great |
| **KTO** | Low | Binary (good/bad) labels | High | Good |
| **ORPO** | Low | Preference pairs | High | Good |
| **Constitutional AI** | Medium | Principles (no human labels) | Medium | Great |

### NLP Task → Approach Mapping

| Task | Zero-Shot | Few-Shot | Fine-Tune | Best Model Type |
|------|----------|---------|-----------|----------------|
| Sentiment | ✓ Good | ✓ Great | ✓ Best | Encoder (BERT) |
| NER | △ Fair | ✓ Good | ✓ Best | Encoder (BERT) |
| QA | ✓ Good | ✓ Good | ✓ Best | Encoder (BERT) or Enc-Dec (T5) |
| Summarization | ✓ Good | ✓ Good | ✓ Best | Enc-Dec (BART, T5) |
| Translation | △ Fair | ✓ Good | ✓ Best | Enc-Dec (mBART, NLLB) |
| Generation | ✓ Good | ✓ Great | ✓ Best | Decoder (GPT, Llama) |
| Code | ✓ Good | ✓ Great | ✓ Best | Decoder (CodeLlama, StarCoder) |

### Production Deployment Checklist

```
□ Model optimized (quantized / distilled / ONNX)
□ Input validation and sanitization
□ Output filtering (toxicity, PII, hallucination checks)
□ Rate limiting and authentication
□ Health check endpoint
□ Monitoring: latency, throughput, error rate, accuracy drift
□ Logging: inputs, outputs, model version (with PII masking)
□ Fallback strategy (simpler model, cached responses)
□ A/B testing infrastructure
□ Model versioning and rollback capability
□ Load testing completed
□ Security review (prompt injection, data leakage)
```

---

## Summary

| Topic | Key Takeaway |
|-------|-------------|
| **Multilingual NLP** | Single model for 100+ languages; cross-lingual transfer enables zero-shot to new languages |
| **Zero-Shot** | Leverage NLI or instruction-tuned models to perform tasks without task-specific training |
| **Few-Shot** | In-context learning with 1-32 examples; SetFit for efficient few-shot classification |
| **Instruction Tuning** | Teach models to follow instructions across diverse tasks; transforms base model → assistant |
| **PEFT (LoRA)** | Fine-tune <1% of parameters; QLoRA enables 7B model tuning on consumer GPUs |
| **Alignment** | RLHF/DPO make models helpful, harmless, honest; DPO simpler than RLHF |
| **Multimodal** | Vision-language models, speech recognition, document understanding |
| **Efficiency** | Distillation, quantization, pruning for production deployment |
| **Production** | Monitoring, safety, scaling, versioning — the engineering matters as much as the model |
