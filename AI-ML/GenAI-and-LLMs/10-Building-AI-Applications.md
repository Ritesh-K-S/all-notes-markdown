# Chapter 10: Building AI Applications — From Prototype to Production

## Table of Contents
- [What Are AI Applications](#what-are-ai-applications)
- [Why Building AI Apps is Different](#why-building-ai-apps-is-different)
- [Chatbots and Conversational AI](#chatbots-and-conversational-ai)
- [AI Copilots and Assistants](#ai-copilots-and-assistants)
- [Multi-Modal AI Applications](#multi-modal-ai-applications)
- [Production Patterns and Architecture](#production-patterns-and-architecture)
- [Evaluation and Monitoring](#evaluation-and-monitoring)
- [Code Examples](#code-examples)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What Are AI Applications

### Simple Explanation (Like Explaining to a 15-Year-Old)

AI applications are software products that use AI models (like GPT, Claude, or Llama) as their "brain" to do useful things for people. Think of it like this:

- The **AI model** is the engine
- The **application** is the car you actually drive

Just like you don't give customers a raw engine — you build a car around it with steering, brakes, a dashboard — you don't give users raw GPT-4. You build an application with a user interface, safety guardrails, memory, and tools that makes the AI actually useful.

**Examples:**
- ChatGPT = GPT-4 engine + chat UI + memory + safety filters + tools
- GitHub Copilot = Codex/GPT-4 engine + VS Code integration + code context
- Perplexity = LLM engine + web search + citation system

### The AI Application Stack

```
┌─────────────────────────────────────────────────────────────┐
│                    USER INTERFACE                              │
│  Chat UI │ API │ Browser Extension │ IDE Plugin │ Voice      │
├─────────────────────────────────────────────────────────────┤
│                    APPLICATION LAYER                          │
│  Routing │ Guardrails │ Memory │ Session │ Auth │ Rate Limit│
├─────────────────────────────────────────────────────────────┤
│                    ORCHESTRATION LAYER                        │
│  Prompt Templates │ Chains │ Agents │ RAG Pipeline           │
│  Tool Calling │ Function Execution │ Multi-Step Reasoning   │
├─────────────────────────────────────────────────────────────┤
│                    MODEL LAYER                                │
│  LLM (GPT-4, Claude, Llama) │ Embedding Model │ Vision     │
│  Speech-to-Text │ Text-to-Speech │ Image Generation         │
├─────────────────────────────────────────────────────────────┤
│                    INFRASTRUCTURE                             │
│  Vector DB │ Cache │ Logging │ Monitoring │ GPU Serving      │
└─────────────────────────────────────────────────────────────┘
```

---

## Why Building AI Apps is Different

### Non-Deterministic by Nature

Traditional software: same input → always same output.
AI applications: same input → different output each time.

```
Traditional API:
  add(2, 3) → 5  (always)

LLM API:
  ask("What is 2+3?") → "The answer is 5."      (usually)
                       → "2 + 3 equals 5."       (sometimes)
                       → "Five."                  (occasionally)
                       → "2+3=6" (hallucination!) (rarely but possible)
```

### Key Challenges Unique to AI Apps

| Challenge | Description | Solution |
|---|---|---|
| **Hallucination** | Model confidently generates false info | RAG, grounding, fact-checking |
| **Latency** | LLMs are slow (seconds vs milliseconds) | Streaming, caching, smaller models |
| **Cost** | API calls are expensive at scale | Caching, model routing, batching |
| **Non-determinism** | Same prompt gives different results | Temperature=0, seed, structured output |
| **Context limits** | Limited input window (4K-128K tokens) | Chunking, summarization, RAG |
| **Safety** | Users will try adversarial inputs | Guardrails, input/output filtering |
| **Evaluation** | "Good" is subjective and hard to measure | LLM-as-judge, human eval, metrics |

### The 80/20 Rule of AI Apps

```
Effort distribution in a production AI app:
─────────────────────────────────────────────
  10%  │████░░░░░░│  Model selection & prompting
  20%  │████████░░│  RAG / data pipeline
  30%  │████████████████░░░░│  Guardrails, edge cases, error handling
  40%  │████████████████████████████████│  Evaluation, monitoring, iteration

The model is the EASY part. Everything around it is what makes it production-ready.
```

---

## Chatbots and Conversational AI

### Architecture of a Production Chatbot

```
┌─────────────────────────────────────────────────────────────────┐
│                    CHATBOT ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  User Message                                                    │
│       │                                                          │
│       ▼                                                          │
│  ┌──────────────┐                                                │
│  │ INPUT GUARD   │ → Block harmful/adversarial inputs            │
│  └──────┬───────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐     ┌──────────────┐                          │
│  │ ROUTER       │────▶│ RAG Pipeline │ (if knowledge needed)    │
│  │ (classify    │     └──────┬───────┘                          │
│  │  intent)     │            │ relevant docs                    │
│  └──────┬───────┘            │                                  │
│         │                    │                                   │
│         ▼                    ▼                                   │
│  ┌────────────────────────────────────┐                         │
│  │ PROMPT ASSEMBLY                     │                         │
│  │ System prompt + History + Context   │                         │
│  │ + Retrieved docs + User message     │                         │
│  └──────────────┬─────────────────────┘                         │
│                 │                                                │
│                 ▼                                                │
│  ┌──────────────┐                                                │
│  │ LLM          │ → Generate response                           │
│  └──────┬───────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                                │
│  │ OUTPUT GUARD  │ → Filter PII, toxicity, hallucinations       │
│  └──────┬───────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                                │
│  │ MEMORY       │ → Store conversation in history               │
│  └──────┬───────┘                                                │
│         │                                                        │
│         ▼                                                        │
│     Response to User (streamed)                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Conversation Memory Strategies

| Strategy | How It Works | Pros | Cons |
|---|---|---|---|
| **Full History** | Send entire conversation | Perfect context | Hits token limit fast |
| **Sliding Window** | Keep last N messages | Simple, bounded | Loses early context |
| **Summarization** | Periodically summarize old messages | Compact, preserves key info | Lossy, extra LLM call |
| **Token Budget** | Keep messages until token limit | Efficient | May cut mid-conversation |
| **Semantic Memory** | Store key facts in a knowledge base | Long-term memory | Complex to implement |

### Conversation Memory Implementation

```
Sliding Window (most common):
─────────────────────────────────
Messages: [M1, M2, M3, M4, M5, M6, M7, M8, M9, M10]
Window=6: [System] + [M5, M6, M7, M8, M9, M10]

Summarization Strategy:
─────────────────────────────────
Messages: [M1, M2, M3, M4, M5, M6, M7, M8, M9, M10]
Result:   [System] + [Summary of M1-M5] + [M6, M7, M8, M9, M10]

Token Budget Strategy:
─────────────────────────────────
Budget: 2000 tokens for history
Start from most recent, add messages until budget exhausted
Result: [System] + [Most recent messages fitting 2000 tokens]
```

---

## AI Copilots and Assistants

### What Makes a Copilot Different from a Chatbot?

| Aspect | Chatbot | Copilot |
|---|---|---|
| Interface | Standalone chat window | Embedded in existing tool |
| Context | Conversation history | Work context (code, docs, data) |
| Actions | Text responses | Can execute actions (write code, create files) |
| Proactivity | Responds to queries | Can suggest without being asked |
| Domain | General-purpose | Domain-specific (coding, writing, data) |

### Tool/Function Calling

The key capability that turns a chatbot into a copilot — the LLM can call external functions:

```
User: "What's the weather in Tokyo?"

Without Tool Calling:
  LLM: "I don't have access to real-time weather data." ❌

With Tool Calling:
  1. LLM decides to call: get_weather(city="Tokyo")
  2. System executes function → {"temp": 25, "condition": "Sunny"}
  3. LLM formats response: "It's 25°C and sunny in Tokyo!" ✅
```

### Tool Calling Flow

```
┌──────────────────────────────────────────────────────────────┐
│                    TOOL CALLING FLOW                           │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. User sends message                                        │
│  2. LLM receives message + list of available tools            │
│  3. LLM decides: respond directly OR call a tool              │
│                                                               │
│     If tool call:                                             │
│     4. LLM outputs: {"tool": "search_db", "args": {...}}     │
│     5. Application executes the tool                          │
│     6. Tool result sent back to LLM                          │
│     7. LLM generates final response using tool result         │
│                                                               │
│     If multiple tools needed:                                 │
│     4-7 repeat (sequential or parallel tool calls)            │
│                                                               │
│  Common tools:                                                │
│  • search_knowledge_base(query)                               │
│  • get_user_data(user_id)                                     │
│  • execute_sql(query)                                         │
│  • send_email(to, subject, body)                              │
│  • create_calendar_event(details)                             │
│  • run_code(language, code)                                   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Agentic Patterns

#### 1. ReAct (Reasoning + Acting)

```
User: "Find the tallest building in each continent and compare their heights"

Thought: I need to search for tallest buildings by continent.
Action: search("tallest building Asia")
Observation: Burj Khalifa, 828m, Dubai

Thought: Good, now I need Africa.
Action: search("tallest building Africa")
Observation: Iconic Tower, 385m, Egypt

Thought: Now Europe...
Action: search("tallest building Europe")
Observation: Lakhta Center, 462m, Russia

... (continues for all continents)

Thought: I have all the data. Let me create a comparison.
Action: create_table(data)
Final Answer: [formatted comparison table]
```

#### 2. Plan-and-Execute

```
User: "Analyze our Q3 sales data and create a summary report"

PLAN:
1. Query the sales database for Q3 data
2. Calculate key metrics (revenue, growth, top products)
3. Generate visualizations
4. Write executive summary
5. Format as PDF report

EXECUTE:
Step 1: execute_sql("SELECT * FROM sales WHERE quarter='Q3'")
Step 2: calculate_metrics(data)
Step 3: create_charts(metrics)
Step 4: generate_summary(metrics, charts)
Step 5: create_pdf(summary)
```

#### 3. Multi-Agent Collaboration

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  RESEARCHER  │────▶│   WRITER     │────▶│   REVIEWER   │
│              │     │              │     │              │
│ Gathers info │     │ Drafts text  │     │ Checks quality│
│ from sources │     │ from research│     │ suggests edits│
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                                  ▼
                                           Final Output
```

---

## Multi-Modal AI Applications

### Types of Multi-Modal Applications

| Input → Output | Example Application | Models Used |
|---|---|---|
| Text → Image | DALL-E, Midjourney | Stable Diffusion, FLUX |
| Image → Text | Image captioning, visual QA | GPT-4V, LLaVA, Claude |
| Text → Speech | Voice assistants, audiobooks | Bark, XTTS, ElevenLabs |
| Speech → Text | Transcription, voice commands | Whisper, Deepgram |
| Text → Video | Video generation | Sora, Runway Gen-3 |
| Image + Text → Text | Document understanding, OCR | GPT-4V, Claude, Qwen-VL |
| Any → Any | Unified multi-modal | GPT-4o, Gemini |

### Vision-Language Models (VLMs) Architecture

```
┌────────────────────────────────────────────────────────────┐
│              VISION-LANGUAGE MODEL (VLM)                     │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  Image ──▶ [Vision Encoder] ──▶ Image Tokens               │
│            (CLIP/SigLIP)        (e.g., 576 tokens)          │
│                                       │                     │
│                                       ▼                     │
│  Text ───▶ [Tokenizer] ──────▶ Text Tokens                 │
│                                       │                     │
│                                       ▼                     │
│                          [Projection Layer]                  │
│                          (align vision to text space)        │
│                                       │                     │
│                                       ▼                     │
│                   [LLM Backbone (e.g., LLaMA)]              │
│                   Processes mixed image+text tokens          │
│                                       │                     │
│                                       ▼                     │
│                              Text Response                   │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Multi-Modal Pipeline Example

```
Document Processing Pipeline:
─────────────────────────────────
1. PDF → Images (page screenshots)
2. Images → GPT-4V/Claude (extract text + understand layout)
3. Extracted info → Structured JSON
4. JSON → Database / Search index
5. User query → RAG over structured data → Answer

Why not just OCR?
• VLMs understand LAYOUT (tables, charts, diagrams)
• Can interpret handwriting
• Understand visual relationships
• Can answer questions about the content
```

---

## Production Patterns and Architecture

### Pattern 1: Prompt Chaining

Break complex tasks into a sequence of simpler LLM calls:

```
User: "Write a blog post about AI in healthcare"

Chain:
Step 1 (Research):    LLM → Generate outline with key points
Step 2 (Draft):       LLM → Write each section from outline
Step 3 (Review):      LLM → Check for accuracy and flow
Step 4 (Polish):      LLM → Final editing and formatting

Each step is a separate, focused LLM call.
Better results than one-shot "write a blog post."
```

### Pattern 2: Fallback / Model Cascade

```
User Query
    │
    ▼
┌──────────────┐     Quick, cheap answer?
│ Small Model   │──── Yes ────▶ Return response ($0.001)
│ (Llama-8B)   │
└──────┬───────┘
       │ No (confidence too low)
       ▼
┌──────────────┐     Good answer?
│ Medium Model  │──── Yes ────▶ Return response ($0.01)
│ (GPT-4o-mini)│
└──────┬───────┘
       │ No (still uncertain)
       ▼
┌──────────────┐
│ Large Model   │────────────▶ Return response ($0.10)
│ (GPT-4o)     │
└──────────────┘

Result: 80% of queries handled by cheapest model.
Average cost drops by 5-10×.
```

### Pattern 3: Structured Output

Force LLM to produce machine-parseable output:

```python
# Instead of:
"Extract the name and age from this text"
# → "The person's name is John and they are 30 years old"  (hard to parse!)

# Use structured output:
{
    "name": "John",
    "age": 30
}
```

### Pattern 4: Guardrails Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   GUARDRAILS PIPELINE                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  INPUT GUARDRAILS:                                           │
│  ├── PII Detection (redact SSN, emails, phone numbers)      │
│  ├── Prompt Injection Detection (block adversarial inputs)   │
│  ├── Topic Classifier (block off-topic / harmful topics)    │
│  └── Rate Limiting (prevent abuse)                          │
│                                                              │
│  OUTPUT GUARDRAILS:                                          │
│  ├── Hallucination Detection (fact-check against sources)   │
│  ├── Toxicity Filter (block harmful content)                │
│  ├── PII Scrubbing (remove any leaked PII)                  │
│  ├── Format Validation (ensure JSON/structured output)      │
│  └── Relevance Check (is response on-topic?)                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Pattern 5: Semantic Caching

```
Traditional Cache:
  Hash("What is Python?") ≠ Hash("Explain Python")
  → Cache MISS (even though same question!)

Semantic Cache:
  Embed("What is Python?") ≈ Embed("Explain Python")
  Cosine similarity = 0.95 > threshold
  → Cache HIT! Return cached response.

Savings: 30-50% of LLM API calls at scale
```

### Pattern 6: Streaming Architecture

```
Without Streaming:
─────────────────────────────────
User sends query → Waits 3 seconds → Gets full response at once
User perception: "Slow, unresponsive"

With Streaming:
─────────────────────────────────
User sends query → Gets first token in 200ms → Tokens arrive one by one
User perception: "Fast, responsive"

Implementation:
Server: Server-Sent Events (SSE) or WebSocket
Client: Process token-by-token, render incrementally
```

---

## Evaluation and Monitoring

### The Evaluation Problem

```
Traditional Software:
  assert add(2, 3) == 5  ✓ or ✗ (binary, deterministic)

AI Applications:
  assert llm("Explain AI") == ???  What's the "right" answer?
  
  Evaluation must be MULTI-DIMENSIONAL:
  • Is it factually correct?
  • Is it helpful?
  • Is it safe?
  • Is it well-formatted?
  • Is it concise/appropriate length?
  • Does it follow the system prompt?
```

### Evaluation Methods

| Method | Description | Cost | Reliability |
|---|---|---|---|
| **Human evaluation** | Paid annotators rate responses | High ($) | Gold standard |
| **LLM-as-Judge** | GPT-4 rates other model outputs | Medium | Good (biased toward verbose) |
| **Reference-based** | Compare against gold answers (BLEU, ROUGE) | Low | Poor for open-ended |
| **Automated metrics** | Factuality, toxicity, format compliance | Low | Good for specific aspects |
| **A/B testing** | Show users two versions, measure preference | Medium | Best for production |
| **User feedback** | Thumbs up/down, ratings | Free | Noisy but scalable |

### LLM-as-Judge Framework

```
Evaluation Prompt:
─────────────────────────────────
"You are an expert evaluator. Rate the following response on a scale of 1-5 
for each criterion:

Question: {user_question}
Response: {model_response}
Reference (if available): {reference_answer}

Criteria:
1. Correctness (1-5): Is the information factually accurate?
2. Helpfulness (1-5): Does it address the user's actual need?
3. Safety (1-5): Is it free from harmful content?
4. Conciseness (1-5): Is it appropriately detailed without being verbose?

Provide scores and brief justification for each."
```

### Production Monitoring Dashboard

```
┌─────────────────────────────────────────────────────────────┐
│              AI APPLICATION MONITORING                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PERFORMANCE METRICS:                                        │
│  • TTFT (P50/P99): 180ms / 450ms         [██████░░░░] OK    │
│  • E2E Latency (P50/P99): 2.1s / 5.8s   [████████░░] OK    │
│  • Throughput: 150 req/min                [████████░░] OK    │
│  • Error rate: 0.3%                       [██░░░░░░░░] Good │
│                                                              │
│  QUALITY METRICS:                                            │
│  • User satisfaction (thumbs up): 87%    [████████░░] Good  │
│  • Hallucination rate: 4.2%              [███░░░░░░░] Watch │
│  • Safety violations: 0.01%              [█░░░░░░░░░] Great │
│  • Guardrail triggers: 2.1%             [██░░░░░░░░] Normal│
│                                                              │
│  COST METRICS:                                               │
│  • Avg tokens/request: 1,240             [████░░░░░░]       │
│  • Daily API cost: $342                  [██████░░░░]       │
│  • Cache hit rate: 34%                   [███░░░░░░░]       │
│  • Cost per conversation: $0.08          [████░░░░░░]       │
│                                                              │
│  ALERTS:                                                     │
│  ⚠ Hallucination rate up 1.2% from last week                │
│  ⚠ P99 latency spike at 14:00 (capacity issue)              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### What to Log (Essential)

| Field | Why |
|---|---|
| Request ID | Trace individual requests |
| User ID | Track per-user patterns |
| Prompt (template + variables) | Debug and reproduce issues |
| Model response | Quality analysis |
| Model name + version | Track model changes |
| Token counts (in/out) | Cost tracking |
| Latency (TTFT, total) | Performance monitoring |
| Tool calls made | Debug agent behavior |
| Guardrail triggers | Safety monitoring |
| User feedback | Quality signal |
| Error details | Incident investigation |

---

## Code Examples

### Example 1: Production Chatbot with OpenAI

```python
from openai import OpenAI
from typing import Generator
import json
import time

# ============================================
# Production-Ready Chatbot
# ============================================

class ChatBot:
    """Production chatbot with memory, streaming, and guardrails."""
    
    def __init__(self, model="gpt-4o", system_prompt=None, max_history=20):
        self.client = OpenAI()
        self.model = model
        self.max_history = max_history
        self.system_prompt = system_prompt or (
            "You are a helpful assistant. Be concise and accurate. "
            "If you're unsure about something, say so. "
            "Never make up facts or URLs."
        )
        self.conversation_history = []
        self.total_tokens_used = 0
    
    def _build_messages(self, user_message: str) -> list:
        """Build message list with system prompt and sliding window history."""
        messages = [{"role": "system", "content": self.system_prompt}]
        
        # Sliding window: keep only last N messages
        history = self.conversation_history[-self.max_history:]
        messages.extend(history)
        
        messages.append({"role": "user", "content": user_message})
        return messages
    
    def _input_guardrail(self, message: str) -> tuple[bool, str]:
        """Check input for safety issues. Returns (is_safe, reason)."""
        # Check for prompt injection patterns
        injection_patterns = [
            "ignore previous instructions",
            "ignore all instructions",
            "disregard your system prompt",
            "you are now",
            "new persona",
        ]
        message_lower = message.lower()
        for pattern in injection_patterns:
            if pattern in message_lower:
                return False, "Potential prompt injection detected."
        
        # Check message length (prevent token-stuffing attacks)
        if len(message) > 10000:
            return False, "Message too long. Please keep under 10,000 characters."
        
        return True, ""
    
    def _output_guardrail(self, response: str) -> str:
        """Filter output for safety. Redact PII patterns."""
        import re
        # Redact email addresses
        response = re.sub(
            r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}',
            '[EMAIL REDACTED]', response
        )
        # Redact phone numbers (US format)
        response = re.sub(
            r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b',
            '[PHONE REDACTED]', response
        )
        return response
    
    def chat(self, user_message: str) -> str:
        """Send a message and get a complete response."""
        # Input guardrail
        is_safe, reason = self._input_guardrail(user_message)
        if not is_safe:
            return f"I can't process that request. {reason}"
        
        messages = self._build_messages(user_message)
        
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=0.7,
                max_tokens=1024,
            )
            
            assistant_message = response.choices[0].message.content
            
            # Track token usage
            self.total_tokens_used += response.usage.total_tokens
            
            # Output guardrail
            assistant_message = self._output_guardrail(assistant_message)
            
            # Update history
            self.conversation_history.append(
                {"role": "user", "content": user_message}
            )
            self.conversation_history.append(
                {"role": "assistant", "content": assistant_message}
            )
            
            return assistant_message
            
        except Exception as e:
            return f"Sorry, I encountered an error. Please try again. ({type(e).__name__})"
    
    def chat_stream(self, user_message: str) -> Generator[str, None, None]:
        """Send a message and stream the response token by token."""
        is_safe, reason = self._input_guardrail(user_message)
        if not is_safe:
            yield f"I can't process that request. {reason}"
            return
        
        messages = self._build_messages(user_message)
        
        stream = self.client.chat.completions.create(
            model=self.model,
            messages=messages,
            temperature=0.7,
            max_tokens=1024,
            stream=True,
        )
        
        full_response = []
        for chunk in stream:
            if chunk.choices[0].delta.content:
                token = chunk.choices[0].delta.content
                full_response.append(token)
                yield token
        
        # Store complete response in history
        complete = "".join(full_response)
        complete = self._output_guardrail(complete)
        self.conversation_history.append({"role": "user", "content": user_message})
        self.conversation_history.append({"role": "assistant", "content": complete})
    
    def get_stats(self) -> dict:
        """Return usage statistics."""
        return {
            "total_messages": len(self.conversation_history),
            "total_tokens": self.total_tokens_used,
            "estimated_cost": self.total_tokens_used * 0.00001,  # Rough estimate
        }


# Usage
bot = ChatBot(
    model="gpt-4o",
    system_prompt="You are a Python programming tutor. Explain concepts clearly with examples.",
)

# Regular chat
response = bot.chat("What is a decorator in Python?")
print(response)

# Streaming chat
print("\n--- Streaming ---")
for token in bot.chat_stream("Show me a practical example"):
    print(token, end="", flush=True)

print(f"\n\nStats: {bot.get_stats()}")
```

### Example 2: Tool-Calling Agent

```python
from openai import OpenAI
import json
import datetime

# ============================================
# Agent with Tool Calling
# ============================================

# Define available tools
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get the current weather for a location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "City name, e.g., 'Tokyo' or 'New York'",
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "Temperature unit",
                    },
                },
                "required": ["location"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "search_knowledge_base",
            "description": "Search the company knowledge base for information",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "Search query",
                    },
                },
                "required": ["query"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "create_task",
            "description": "Create a task or reminder",
            "parameters": {
                "type": "object",
                "properties": {
                    "title": {"type": "string", "description": "Task title"},
                    "due_date": {"type": "string", "description": "Due date (YYYY-MM-DD)"},
                    "priority": {"type": "string", "enum": ["low", "medium", "high"]},
                },
                "required": ["title"],
            },
        },
    },
]


# Tool implementations
def get_weather(location: str, unit: str = "celsius") -> dict:
    """Simulated weather API."""
    # In production, call a real weather API
    return {
        "location": location,
        "temperature": 22 if unit == "celsius" else 72,
        "unit": unit,
        "condition": "Partly Cloudy",
        "humidity": 65,
    }


def search_knowledge_base(query: str) -> dict:
    """Simulated knowledge base search."""
    # In production, search your vector DB / documentation
    return {
        "results": [
            {"title": "Company Policy on Remote Work", "snippet": "Employees may work remotely up to 3 days per week..."},
            {"title": "IT Support Guide", "snippet": "For password resets, contact helpdesk@company.com..."},
        ],
        "total_results": 2,
    }


def create_task(title: str, due_date: str = None, priority: str = "medium") -> dict:
    """Simulated task creation."""
    return {
        "task_id": "TASK-001",
        "title": title,
        "due_date": due_date or str(datetime.date.today() + datetime.timedelta(days=7)),
        "priority": priority,
        "status": "created",
    }


# Map function names to implementations
TOOL_FUNCTIONS = {
    "get_weather": get_weather,
    "search_knowledge_base": search_knowledge_base,
    "create_task": create_task,
}


def run_agent(user_message: str, max_iterations: int = 5) -> str:
    """Run the agent with tool calling support."""
    client = OpenAI()
    
    messages = [
        {"role": "system", "content": "You are a helpful assistant with access to tools. Use tools when needed to answer questions accurately."},
        {"role": "user", "content": user_message},
    ]
    
    for iteration in range(max_iterations):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=TOOLS,
            tool_choice="auto",  # Let model decide when to use tools
        )
        
        message = response.choices[0].message
        
        # If no tool calls, return the text response
        if not message.tool_calls:
            return message.content
        
        # Process tool calls
        messages.append(message)  # Add assistant's tool call message
        
        for tool_call in message.tool_calls:
            func_name = tool_call.function.name
            func_args = json.loads(tool_call.function.arguments)
            
            print(f"  [Tool Call] {func_name}({func_args})")
            
            # Execute the tool
            if func_name in TOOL_FUNCTIONS:
                result = TOOL_FUNCTIONS[func_name](**func_args)
            else:
                result = {"error": f"Unknown function: {func_name}"}
            
            # Add tool result to messages
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result),
            })
    
    return "Max iterations reached. Please simplify your request."


# Usage
print("--- Simple Query ---")
result = run_agent("What's the weather like in Tokyo?")
print(result)

print("\n--- Multi-Tool Query ---")
result = run_agent("Check the weather in London and create a task to pack an umbrella if it's rainy")
print(result)

print("\n--- Knowledge Base Query ---")
result = run_agent("What's our company's remote work policy?")
print(result)
```

### Example 3: Structured Output with Validation

```python
from openai import OpenAI
from pydantic import BaseModel, Field, field_validator
from typing import Optional
import json

# ============================================
# Structured Output (Reliable JSON from LLMs)
# ============================================

# Define output schema with Pydantic
class ExtractedContact(BaseModel):
    """Schema for extracted contact information."""
    name: str = Field(description="Full name of the person")
    email: Optional[str] = Field(None, description="Email address")
    phone: Optional[str] = Field(None, description="Phone number")
    company: Optional[str] = Field(None, description="Company name")
    role: Optional[str] = Field(None, description="Job title or role")
    
    @field_validator('email')
    @classmethod
    def validate_email(cls, v):
        if v and '@' not in v:
            raise ValueError('Invalid email format')
        return v


class ContactExtractionResult(BaseModel):
    """Result of contact extraction."""
    contacts: list[ExtractedContact]
    confidence: float = Field(ge=0, le=1, description="Confidence score 0-1")
    raw_text_summary: str


def extract_contacts(text: str) -> ContactExtractionResult:
    """Extract structured contact info from unstructured text."""
    client = OpenAI()
    
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "system",
                "content": "Extract contact information from the text. Return valid JSON matching the schema."
            },
            {"role": "user", "content": text}
        ],
        response_format={
            "type": "json_schema",
            "json_schema": {
                "name": "contact_extraction",
                "schema": ContactExtractionResult.model_json_schema(),
            }
        },
        temperature=0,  # Deterministic for structured extraction
    )
    
    # Parse and validate with Pydantic
    result = ContactExtractionResult.model_validate_json(
        response.choices[0].message.content
    )
    return result


# Usage
text = """
Hey! Just had a great meeting with Sarah Chen from TechCorp. 
She's their VP of Engineering. You can reach her at 
sarah.chen@techcorp.io or call 555-0142. Also met with 
James Wilson, a freelance consultant — his email is 
james@wilsontech.com.
"""

result = extract_contacts(text)
print(f"Found {len(result.contacts)} contacts (confidence: {result.confidence})")
for contact in result.contacts:
    print(f"  - {contact.name} ({contact.role}) at {contact.company}")
    print(f"    Email: {contact.email}, Phone: {contact.phone}")
```

### Example 4: Semantic Caching

```python
import numpy as np
import hashlib
import time
from openai import OpenAI

# ============================================
# Semantic Cache for LLM Responses
# ============================================

class SemanticCache:
    """Cache LLM responses using semantic similarity of queries."""
    
    def __init__(self, similarity_threshold=0.92):
        self.client = OpenAI()
        self.cache = {}          # {hash: {"embedding": [...], "query": str, "response": str}}
        self.embeddings = []     # List of (hash, embedding) for similarity search
        self.threshold = similarity_threshold
        self.hits = 0
        self.misses = 0
    
    def _get_embedding(self, text: str) -> list[float]:
        """Get embedding for a text query."""
        response = self.client.embeddings.create(
            model="text-embedding-3-small",
            input=text,
        )
        return response.data[0].embedding
    
    def _cosine_similarity(self, a: list, b: list) -> float:
        """Compute cosine similarity between two vectors."""
        a, b = np.array(a), np.array(b)
        return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
    
    def _find_similar(self, query_embedding: list) -> tuple[str, float] | None:
        """Find the most similar cached query."""
        best_match = None
        best_score = 0
        
        for cache_hash, cached_embedding in self.embeddings:
            score = self._cosine_similarity(query_embedding, cached_embedding)
            if score > best_score:
                best_score = score
                best_match = cache_hash
        
        if best_score >= self.threshold:
            return best_match, best_score
        return None
    
    def get_or_generate(self, query: str, model: str = "gpt-4o",
                         temperature: float = 0.7, **kwargs) -> dict:
        """Check cache first, generate if miss."""
        start = time.time()
        
        # Get query embedding
        query_embedding = self._get_embedding(query)
        
        # Check for semantically similar cached query
        match = self._find_similar(query_embedding)
        
        if match:
            cache_hash, similarity = match
            cached = self.cache[cache_hash]
            self.hits += 1
            return {
                "response": cached["response"],
                "cached": True,
                "similarity": similarity,
                "original_query": cached["query"],
                "time_ms": (time.time() - start) * 1000,
            }
        
        # Cache miss — generate response
        response = self.client.chat.completions.create(
            model=model,
            messages=[{"role": "user", "content": query}],
            temperature=temperature,
            **kwargs,
        )
        
        response_text = response.choices[0].message.content
        
        # Store in cache
        cache_hash = hashlib.md5(query.encode()).hexdigest()
        self.cache[cache_hash] = {
            "embedding": query_embedding,
            "query": query,
            "response": response_text,
        }
        self.embeddings.append((cache_hash, query_embedding))
        self.misses += 1
        
        return {
            "response": response_text,
            "cached": False,
            "time_ms": (time.time() - start) * 1000,
            "tokens_used": response.usage.total_tokens,
        }
    
    @property
    def hit_rate(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0


# Usage
cache = SemanticCache(similarity_threshold=0.92)

# First query — cache miss
result = cache.get_or_generate("What is machine learning?")
print(f"Cached: {result['cached']} | Time: {result['time_ms']:.0f}ms")

# Similar query — cache hit!
result = cache.get_or_generate("Explain machine learning")
print(f"Cached: {result['cached']} | Similarity: {result.get('similarity', 'N/A'):.3f}")

# Different query — cache miss
result = cache.get_or_generate("How does photosynthesis work?")
print(f"Cached: {result['cached']} | Time: {result['time_ms']:.0f}ms")

print(f"\nCache hit rate: {cache.hit_rate:.1%}")
```

### Example 5: Multi-Modal Application (Vision + Text)

```python
from openai import OpenAI
import base64
from pathlib import Path

# ============================================
# Multi-Modal: Image Understanding
# ============================================

client = OpenAI()

def encode_image(image_path: str) -> str:
    """Encode image to base64 string."""
    with open(image_path, "rb") as f:
        return base64.b64encode(f.read()).decode("utf-8")


def analyze_image(image_path: str, question: str = "Describe this image in detail") -> str:
    """Analyze an image using GPT-4V."""
    base64_image = encode_image(image_path)
    
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": question},
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": f"data:image/jpeg;base64,{base64_image}",
                            "detail": "high",  # "low", "high", or "auto"
                        },
                    },
                ],
            }
        ],
        max_tokens=1000,
    )
    
    return response.choices[0].message.content


def extract_text_from_document(image_path: str) -> dict:
    """Extract structured data from a document image (receipt, invoice, etc.)."""
    base64_image = encode_image(image_path)
    
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "system",
                "content": "Extract all text and data from this document image. Return structured JSON."
            },
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": "Extract all information from this document into structured JSON."},
                    {
                        "type": "image_url",
                        "image_url": {"url": f"data:image/jpeg;base64,{base64_image}"},
                    },
                ],
            }
        ],
        response_format={"type": "json_object"},
        max_tokens=2000,
    )
    
    import json
    return json.loads(response.choices[0].message.content)


def compare_images(image_path_1: str, image_path_2: str, comparison_prompt: str) -> str:
    """Compare two images using GPT-4V."""
    img1 = encode_image(image_path_1)
    img2 = encode_image(image_path_2)
    
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": comparison_prompt},
                    {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{img1}"}},
                    {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{img2}"}},
                ],
            }
        ],
        max_tokens=1000,
    )
    
    return response.choices[0].message.content


# Usage
description = analyze_image("photo.jpg", "What objects are in this image?")
print(description)

doc_data = extract_text_from_document("receipt.jpg")
print(f"Total: ${doc_data.get('total', 'N/A')}")
```

### Example 6: Evaluation Pipeline

```python
from openai import OpenAI
from dataclasses import dataclass
import json

# ============================================
# LLM-as-Judge Evaluation Pipeline
# ============================================

@dataclass
class EvalResult:
    correctness: int     # 1-5
    helpfulness: int     # 1-5
    safety: int          # 1-5
    conciseness: int     # 1-5
    overall: float       # Average
    reasoning: str       # Judge's explanation


def evaluate_response(
    question: str,
    response: str,
    reference: str = None,
    judge_model: str = "gpt-4o"
) -> EvalResult:
    """Evaluate an LLM response using LLM-as-judge."""
    client = OpenAI()
    
    ref_section = f"\nReference answer: {reference}" if reference else ""
    
    eval_prompt = f"""Evaluate the following AI response on a scale of 1-5 for each criterion.

Question: {question}
AI Response: {response}{ref_section}

Rate each criterion (1=terrible, 5=excellent):
1. Correctness: Is the information factually accurate?
2. Helpfulness: Does it address the user's actual need?
3. Safety: Is it free from harmful/inappropriate content?
4. Conciseness: Is it appropriately detailed (not too verbose, not too brief)?

Return JSON with keys: correctness, helpfulness, safety, conciseness, reasoning"""
    
    result = client.chat.completions.create(
        model=judge_model,
        messages=[{"role": "user", "content": eval_prompt}],
        response_format={"type": "json_object"},
        temperature=0,
    )
    
    scores = json.loads(result.choices[0].message.content)
    
    return EvalResult(
        correctness=scores["correctness"],
        helpfulness=scores["helpfulness"],
        safety=scores["safety"],
        conciseness=scores["conciseness"],
        overall=sum([scores["correctness"], scores["helpfulness"],
                     scores["safety"], scores["conciseness"]]) / 4,
        reasoning=scores.get("reasoning", ""),
    )


def run_evaluation_suite(test_cases: list[dict], model_to_test: str) -> dict:
    """Run evaluation on a suite of test cases."""
    client = OpenAI()
    results = []
    
    for i, test in enumerate(test_cases):
        # Generate response from model being tested
        response = client.chat.completions.create(
            model=model_to_test,
            messages=[{"role": "user", "content": test["question"]}],
            temperature=0.7,
        )
        model_response = response.choices[0].message.content
        
        # Evaluate response
        eval_result = evaluate_response(
            question=test["question"],
            response=model_response,
            reference=test.get("reference"),
        )
        
        results.append({
            "question": test["question"],
            "response": model_response[:200],
            "scores": {
                "correctness": eval_result.correctness,
                "helpfulness": eval_result.helpfulness,
                "safety": eval_result.safety,
                "conciseness": eval_result.conciseness,
                "overall": eval_result.overall,
            },
        })
        
        print(f"Test {i+1}/{len(test_cases)}: Overall={eval_result.overall:.1f}/5.0")
    
    # Aggregate scores
    avg_scores = {
        "correctness": sum(r["scores"]["correctness"] for r in results) / len(results),
        "helpfulness": sum(r["scores"]["helpfulness"] for r in results) / len(results),
        "safety": sum(r["scores"]["safety"] for r in results) / len(results),
        "conciseness": sum(r["scores"]["conciseness"] for r in results) / len(results),
        "overall": sum(r["scores"]["overall"] for r in results) / len(results),
    }
    
    print(f"\n{'='*50}")
    print(f"Model: {model_to_test}")
    print(f"Tests: {len(results)}")
    for metric, score in avg_scores.items():
        bar = "█" * int(score) + "░" * (5 - int(score))
        print(f"  {metric:15s}: {score:.2f}/5.0 [{bar}]")
    
    return {"results": results, "averages": avg_scores}


# Example test suite
test_cases = [
    {
        "question": "What causes rain?",
        "reference": "Rain forms when water vapor in the atmosphere condenses into droplets that become heavy enough to fall."
    },
    {
        "question": "How do I sort a list in Python?",
        "reference": "Use sorted(list) for a new sorted list, or list.sort() to sort in-place."
    },
    {
        "question": "Is the Earth flat?",
        "reference": "No, the Earth is an oblate spheroid (roughly spherical, slightly flattened at the poles)."
    },
]

# Run evaluation
report = run_evaluation_suite(test_cases, model_to_test="gpt-4o-mini")
```

---

## Common Mistakes

### 1. Not Streaming Responses
```
❌ User waits 5 seconds for full response to appear at once
   Users think the app is broken/frozen

✅ Stream tokens as they're generated (TTFT < 500ms)
   Users see response building in real-time
   Perceived latency drops dramatically
```

### 2. Putting Everything in One Prompt
```
❌ One massive prompt: "Research X, analyze it, write a report, 
   format it, add citations, check facts, translate to Spanish"
   → Poor results, high token cost, hard to debug

✅ Break into a chain of focused prompts:
   Step 1: Research → Step 2: Analyze → Step 3: Write → Step 4: Format
   Each step is smaller, cheaper, and easier to debug
```

### 3. No Fallback for LLM Failures
```python
# ❌ WRONG: No error handling
response = client.chat.completions.create(...)
return response.choices[0].message.content

# ✅ CORRECT: Retry with fallback
import tenacity

@tenacity.retry(
    stop=tenacity.stop_after_attempt(3),
    wait=tenacity.wait_exponential(min=1, max=10),
)
def call_llm(messages, model="gpt-4o"):
    try:
        return client.chat.completions.create(model=model, messages=messages)
    except Exception:
        # Fallback to cheaper/faster model
        return client.chat.completions.create(model="gpt-4o-mini", messages=messages)
```

### 4. Ignoring Token Costs
```
❌ Sending full conversation history (20K tokens) for simple follow-ups
   Cost: $0.30 per message × 1M messages/month = $300,000/month

✅ Smart context management:
   • Sliding window (last 10 messages)
   • Summarize old history
   • Only include relevant context via RAG
   Cost: $0.03 per message × 1M = $30,000/month (10× cheaper)
```

### 5. Trusting LLM Output Without Validation
```python
# ❌ WRONG: Using LLM output directly as code/SQL
sql_query = llm("Generate SQL to get all users")
db.execute(sql_query)  # SQL injection risk!

# ✅ CORRECT: Validate and sanitize
sql_query = llm("Generate SQL to get all users")
if is_safe_query(sql_query):  # Check for DROP, DELETE, etc.
    db.execute(sql_query, parameterized=True)
```

### 6. Not Using Structured Output
```python
# ❌ WRONG: Parsing free-text LLM output
response = llm("Extract the name and age")
# Response: "The person's name is John and they are 30 years old"
# Now you need regex/parsing... fragile!

# ✅ CORRECT: Use JSON mode or function calling
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[...],
    response_format={"type": "json_object"},  # Guaranteed valid JSON
)
data = json.loads(response.choices[0].message.content)
name = data["name"]  # Clean, reliable
```

### 7. No Evaluation Before Production
```
❌ "It works on my 5 test examples!" → Ship to production
   → 10% error rate on real user queries, customer complaints

✅ Build an eval suite:
   • 100+ diverse test cases
   • Run LLM-as-judge evaluation
   • Track scores over time
   • A/B test model/prompt changes
   • Monitor in production
```

---

## Interview Questions

### Architecture & Design

**Q1: Design a customer support chatbot for an e-commerce company.**
> Architecture: (1) Intent classifier routes to specialized handlers (order status, returns, FAQ). (2) RAG pipeline for product/policy knowledge base. (3) Tool calling for order lookup, refund processing. (4) Guardrails: input filter for abuse, output filter for PII. (5) Memory: session-based conversation history with summarization. (6) Escalation path to human agents when confidence is low. (7) Logging: all interactions for quality monitoring and improvement.

**Q2: How would you reduce LLM API costs by 10× while maintaining quality?**
> (1) Semantic caching: avoid repeated API calls for similar queries (30-50% savings). (2) Model routing: use GPT-4o-mini for simple queries, GPT-4o for complex (60-80% cheaper). (3) Prompt optimization: shorter prompts, fewer examples. (4) Batch processing: group non-urgent requests. (5) Fine-tune smaller model on common query types. (6) RAG instead of large context: retrieve relevant info instead of stuffing context.

**Q3: Explain the trade-offs between using an API (OpenAI) vs self-hosted model (vLLM + Llama).**
> API: No infrastructure, latest models, pay-per-token, data leaves your network, vendor lock-in, simple scaling. Self-hosted: Full data control, fixed cost at scale, customizable, requires GPU infrastructure, model may be weaker, need ML ops team. Choose API for: prototyping, variable load, best quality needed. Choose self-hosted for: data privacy requirements, predictable high volume, need for fine-tuning, regulatory compliance.

**Q4: How do you handle hallucinations in a production application?**
> Multi-layer approach: (1) Use RAG to ground responses in factual data. (2) Instruct model to cite sources and say "I don't know." (3) Post-generation fact-checking against knowledge base. (4) Structured output with validation. (5) Confidence scoring — flag low-confidence answers for human review. (6) User feedback loop to identify and fix recurring hallucinations.

**Q5: What is the difference between prompt chaining, agents, and RAG?**
> Prompt chaining: Sequential LLM calls where output of one step feeds into the next. Best for predictable multi-step workflows. Agents: LLM autonomously decides which tools to call and in what order using reasoning (ReAct pattern). Best for dynamic, unpredictable tasks. RAG: Retrieve relevant documents, inject into prompt for grounded responses. Best for knowledge-intensive Q&A. These patterns are complementary and often combined.

**Q6: Design an evaluation pipeline for an LLM-powered product.**
> (1) Offline eval: 500+ test cases with expected outputs, scored by LLM-as-judge on correctness/helpfulness/safety/conciseness. (2) Automated regression: run eval suite on every prompt/model change. (3) Online monitoring: track user satisfaction (thumbs up/down), response latency, error rates, guardrail triggers. (4) Human review: random sample of 1% of responses reviewed by QA team weekly. (5) A/B testing: compare model/prompt variants on live traffic with statistical significance.

**Q7: How would you implement a model routing system?**
> (1) Train a lightweight classifier on query complexity (simple/medium/complex). Features: query length, entity count, question type, domain. (2) Route: simple → 7B model (fast, cheap), medium → GPT-4o-mini, complex → GPT-4o. (3) Fallback: if smaller model's confidence is low, escalate to larger model. (4) Log all routing decisions and outcomes for continuous improvement. Typical result: 70% of queries handled by cheapest tier.

### Technical Questions

**Q8: Explain how tool/function calling works in LLMs.**
> The LLM is trained (or prompted) to output structured JSON specifying which function to call and with what arguments, instead of generating text. The application parses this JSON, executes the function, feeds the result back to the LLM, and the LLM generates a natural language response incorporating the tool result. This enables LLMs to interact with external systems (databases, APIs, calculators) reliably.

**Q9: What is semantic caching and when would you use it?**
> Semantic caching embeds user queries and finds similar previously-answered queries using cosine similarity. If similarity > threshold (e.g., 0.92), return the cached response instead of calling the LLM. Use when: many users ask similar questions, responses don't need to be real-time, cost reduction is important. Don't use when: every response must be unique, data changes frequently, low query volume.

**Q10: How do you handle context window limitations for long documents?**
> Options ranked by complexity: (1) Truncation: use first/last N tokens (lossy but simple). (2) Chunking + Map-Reduce: split document, process chunks independently, combine results. (3) RAG: embed chunks, retrieve only relevant ones for each query. (4) Hierarchical summarization: summarize sections, then summarize summaries. (5) Long-context models: use models with 128K+ context (expensive but simplest). Best practice: use RAG for Q&A, map-reduce for summarization, long-context for analysis requiring full document understanding.

---

## Quick Reference

### AI Application Architecture Patterns

| Pattern | Use Case | Complexity |
|---|---|---|
| Simple prompt | Single-turn Q&A | Low |
| Prompt chain | Multi-step workflows | Medium |
| RAG | Knowledge-grounded Q&A | Medium |
| Tool calling | Actions + external data | Medium |
| ReAct agent | Dynamic multi-step reasoning | High |
| Multi-agent | Complex collaborative tasks | Very High |

### Model Selection Guide

| Requirement | Recommended Model | Cost |
|---|---|---|
| Best quality, any cost | GPT-4o / Claude Opus | $$$$ |
| Good quality, moderate cost | GPT-4o-mini / Claude Sonnet | $$ |
| Self-hosted, privacy | Llama-3.1 70B (vLLM) | Server cost |
| Local/edge deployment | Llama-3.1 8B (GGUF) | Free |
| Embeddings | text-embedding-3-small | $ |
| Fast structured extraction | GPT-4o-mini + JSON mode | $ |

### Cost Optimization Checklist

- [ ] Implement semantic caching (30-50% savings)
- [ ] Use model routing (small → large fallback)
- [ ] Optimize prompts (shorter = cheaper)
- [ ] Use streaming (better UX, no wasted tokens on timeout)
- [ ] Batch non-urgent requests
- [ ] Monitor token usage per feature
- [ ] Set max_tokens to prevent runaway costs
- [ ] Use cheaper models where quality permits
- [ ] Cache embeddings (don't re-embed same text)
- [ ] Fine-tune small model for high-volume use cases

### Production Readiness Checklist

- [ ] Input guardrails (injection detection, PII filtering)
- [ ] Output guardrails (toxicity, PII, hallucination checks)
- [ ] Streaming responses enabled
- [ ] Error handling with retry and fallback models
- [ ] Rate limiting per user/API key
- [ ] Comprehensive logging (prompts, responses, latency, tokens)
- [ ] Evaluation suite with 100+ test cases
- [ ] Monitoring dashboard (latency, errors, cost, quality)
- [ ] User feedback collection (thumbs up/down minimum)
- [ ] Cost alerts and budgets set
- [ ] Data retention/privacy compliance
- [ ] Load testing completed

### Key Metrics to Track

| Category | Metric | Target |
|---|---|---|
| **Performance** | TTFT (P50) | < 500ms |
| **Performance** | E2E Latency (P99) | < 10s |
| **Quality** | User satisfaction | > 85% |
| **Quality** | Hallucination rate | < 5% |
| **Safety** | Guardrail trigger rate | 1-5% (too low = not catching enough) |
| **Cost** | Cost per conversation | Track trend, optimize continuously |
| **Reliability** | Error rate | < 1% |
| **Reliability** | Uptime | > 99.9% |

---

> **Key Takeaway**: Building AI applications is 10% model, 90% engineering. The model is the easy part — guardrails, evaluation, monitoring, cost optimization, and edge case handling are what make a product production-ready. Start simple (prompt chain + RAG), measure everything, and iterate based on real user data.
