# Chapter 06: LangChain and AI Agents

## Table of Contents
- [What is LangChain?](#what-is-langchain)
- [Why LangChain Matters](#why-langchain-matters)
- [Core Concepts: LCEL and Chains](#core-concepts-lcel-and-chains)
- [Prompts and Output Parsers](#prompts-and-output-parsers)
- [Memory and Conversation History](#memory-and-conversation-history)
- [Tools and Function Calling](#tools-and-function-calling)
- [AI Agents — The Big Picture](#ai-agents--the-big-picture)
- [Building Agents with LangChain](#building-agents-with-langchain)
- [LangGraph — Stateful Agent Workflows](#langgraph--stateful-agent-workflows)
- [Production Patterns](#production-patterns)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is LangChain?

### Simple Explanation (ELI15)

Think of LLMs like brilliant employees who can only do one thing: answer questions from text. LangChain is like a project management system that helps these employees work together — it connects them to databases, lets them use tools (search the web, run code, query APIs), gives them memory of past conversations, and coordinates multi-step workflows. It turns a simple Q&A bot into a full AI application.

### Formal Definition

**LangChain** is an open-source framework for building applications powered by language models. It provides abstractions for:
- Connecting LLMs to external data sources
- Chaining multiple LLM calls together
- Giving LLMs tools to interact with the world
- Managing conversation memory
- Building autonomous agents

```
┌─────────────────────────────────────────────────────────────────┐
│                  LANGCHAIN ECOSYSTEM                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  langchain   │  │  langgraph   │  │  langsmith   │          │
│  │  (Core lib)  │  │  (Agents &   │  │  (Observa-   │          │
│  │              │  │   Workflows) │  │   bility)    │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                   │
│         ▼                  ▼                  ▼                   │
│  • Chains         • State machines    • Tracing                  │
│  • Prompts        • Cycles/loops      • Debugging                │
│  • Memory         • Human-in-loop     • Evaluation               │
│  • Tools          • Multi-agent       • Monitoring               │
│  • Retrievers     • Checkpointing     • Datasets                 │
│                                                                  │
│  ┌────────────────────────────────────────────────────┐         │
│  │            langchain-community                      │         │
│  │   (Integrations: OpenAI, Anthropic, Pinecone,      │         │
│  │    ChromaDB, Google, AWS, etc.)                     │         │
│  └────────────────────────────────────────────────────┘         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Why LangChain Matters

### The Problem It Solves

Without LangChain (raw OpenAI API):
```python
# You have to manually manage everything:
# - Prompt formatting
# - API calls
# - Output parsing
# - Error handling
# - Memory/history
# - Tool integration
# - Chaining steps
# - Retry logic
# This becomes VERY complex for real applications
```

With LangChain:
```python
# All of the above is handled by composable abstractions
# You focus on the LOGIC, not the plumbing
```

### When to Use LangChain

| Use Case | Use LangChain? | Why |
|----------|---------------|-----|
| Simple API call | ❌ No | Overkill — use OpenAI SDK directly |
| Multi-step chain | ✅ Yes | LCEL makes it clean |
| RAG application | ✅ Yes | Built-in retrievers, vector stores |
| Conversational chatbot | ✅ Yes | Memory management |
| Agent with tools | ✅ Yes | Tool calling + agent loop |
| Complex workflows | ✅ Use LangGraph | State machines, cycles |
| Production monitoring | ✅ Use LangSmith | Tracing, evaluation |

### LangChain vs Alternatives

| Framework | Strength | Weakness | Best For |
|-----------|----------|----------|----------|
| **LangChain** | Ecosystem, integrations | Abstraction overhead | Full applications |
| **LlamaIndex** | Document indexing/RAG | Less flexible for agents | Data-heavy RAG |
| **Haystack** | Production pipelines | Smaller community | Enterprise NLP |
| **CrewAI** | Multi-agent systems | Less flexible | Agent teams |
| **Raw SDK** | Full control, minimal deps | More boilerplate | Simple use cases |

---

## Core Concepts: LCEL and Chains

### LCEL (LangChain Expression Language)

LCEL is the modern way to compose LangChain components using the pipe operator (`|`). It provides streaming, async, batching, and fallback support out of the box.

```python
# ═══════════════════════════════════════════════════════
# LCEL Basics — The Modern LangChain Way
# ═══════════════════════════════════════════════════════

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# Initialize LLM
llm = ChatOpenAI(model="gpt-4", temperature=0)

# Create a prompt template
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant that explains {topic} concepts."),
    ("human", "{question}")
])

# Create a chain using LCEL pipe syntax
chain = prompt | llm | StrOutputParser()

# Invoke the chain
result = chain.invoke({
    "topic": "machine learning",
    "question": "What is gradient descent?"
})
print(result)

# ─────────────────────────────────────────────────────
# LCEL gives you these for FREE:
# ─────────────────────────────────────────────────────

# Streaming
for chunk in chain.stream({"topic": "AI", "question": "What is a transformer?"}):
    print(chunk, end="", flush=True)

# Async
import asyncio
result = asyncio.run(chain.ainvoke({"topic": "AI", "question": "What is attention?"}))

# Batch processing
results = chain.batch([
    {"topic": "ML", "question": "What is overfitting?"},
    {"topic": "ML", "question": "What is regularization?"},
    {"topic": "ML", "question": "What is cross-validation?"}
])
```

### How LCEL Works — The Pipe Operator

```
┌─────────────────────────────────────────────────────────────────┐
│                    LCEL CHAIN FLOW                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Input Dict ──► Prompt ──► LLM ──► Output Parser ──► Result     │
│  {"topic": ..   Template    ChatOpenAI  StrOutputParser  String  │
│   "question":}                                                   │
│                                                                  │
│  Each component is a "Runnable" with:                            │
│  • .invoke()  — single input                                     │
│  • .stream()  — streaming output                                 │
│  • .batch()   — multiple inputs                                  │
│  • .ainvoke() — async single                                     │
│                                                                  │
│  The | operator chains: output of left → input of right          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-Step Chains

```python
# ═══════════════════════════════════════════════════════
# Complex Chain: Multiple Steps with Branching
# ═══════════════════════════════════════════════════════

from langchain_core.runnables import RunnablePassthrough, RunnableLambda, RunnableParallel
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

llm = ChatOpenAI(model="gpt-4", temperature=0)

# ─────────────────────────────────────────────────────
# Pattern 1: Sequential Chain (A → B → C)
# ─────────────────────────────────────────────────────

# Step 1: Generate an outline
outline_prompt = ChatPromptTemplate.from_template(
    "Create a brief outline for a blog post about: {topic}"
)

# Step 2: Write the post from the outline
write_prompt = ChatPromptTemplate.from_template(
    "Write a blog post based on this outline:\n{outline}"
)

# Step 3: Create a summary
summary_prompt = ChatPromptTemplate.from_template(
    "Write a 2-sentence summary of this blog post:\n{post}"
)

# Chain them together
sequential_chain = (
    outline_prompt | llm | StrOutputParser() |           # Step 1: topic → outline
    {"outline": RunnablePassthrough()} |                  # Pass output as "outline"
    write_prompt | llm | StrOutputParser() |              # Step 2: outline → post
    {"post": RunnablePassthrough()} |                     # Pass output as "post"
    summary_prompt | llm | StrOutputParser()              # Step 3: post → summary
)

result = sequential_chain.invoke({"topic": "AI in Healthcare"})


# ─────────────────────────────────────────────────────
# Pattern 2: Parallel Chain (A → [B, C] → D)
# ─────────────────────────────────────────────────────

# Run multiple prompts in parallel, combine results
pros_prompt = ChatPromptTemplate.from_template(
    "List 3 pros of {topic}"
)
cons_prompt = ChatPromptTemplate.from_template(
    "List 3 cons of {topic}"
)
synthesis_prompt = ChatPromptTemplate.from_template(
    "Given these pros:\n{pros}\n\nAnd these cons:\n{cons}\n\nWrite a balanced conclusion."
)

# Parallel step
parallel_chain = RunnableParallel(
    pros=pros_prompt | llm | StrOutputParser(),
    cons=cons_prompt | llm | StrOutputParser()
)

# Full chain with parallel + synthesis
full_chain = parallel_chain | synthesis_prompt | llm | StrOutputParser()

result = full_chain.invoke({"topic": "Remote Work"})


# ─────────────────────────────────────────────────────
# Pattern 3: Conditional Chain (Router)
# ─────────────────────────────────────────────────────

from langchain_core.runnables import RunnableBranch

# Different prompts for different types of questions
math_prompt = ChatPromptTemplate.from_template(
    "You are a math tutor. Solve: {question}"
)
code_prompt = ChatPromptTemplate.from_template(
    "You are a programming expert. Help with: {question}"
)
general_prompt = ChatPromptTemplate.from_template(
    "Answer this question: {question}"
)

# Classifier
classify_prompt = ChatPromptTemplate.from_template(
    "Classify this question as 'math', 'code', or 'general': {question}\nCategory:"
)
classifier = classify_prompt | llm | StrOutputParser()

# Router
def route(info):
    category = info["category"].strip().lower()
    if "math" in category:
        return math_prompt | llm | StrOutputParser()
    elif "code" in category:
        return code_prompt | llm | StrOutputParser()
    else:
        return general_prompt | llm | StrOutputParser()

# Full routing chain
routing_chain = (
    {"category": classifier, "question": RunnablePassthrough()} |
    RunnableLambda(route)
)
```

---

## Prompts and Output Parsers

### Prompt Templates

```python
# ═══════════════════════════════════════════════════════
# Prompt Engineering in LangChain
# ═══════════════════════════════════════════════════════

from langchain_core.prompts import (
    ChatPromptTemplate,
    FewShotChatMessagePromptTemplate,
    MessagesPlaceholder
)

# ─────────────────────────────────────────────────────
# Basic Chat Prompt
# ─────────────────────────────────────────────────────

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a {role}. Always respond in {language}."),
    ("human", "{user_input}")
])

# ─────────────────────────────────────────────────────
# Few-Shot Prompt (with examples)
# ─────────────────────────────────────────────────────

examples = [
    {"input": "happy", "output": "sad"},
    {"input": "tall", "output": "short"},
    {"input": "bright", "output": "dark"},
]

example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}")
])

few_shot_prompt = FewShotChatMessagePromptTemplate(
    example_prompt=example_prompt,
    examples=examples,
)

final_prompt = ChatPromptTemplate.from_messages([
    ("system", "You give the opposite of each word."),
    few_shot_prompt,
    ("human", "{input}")
])

# ─────────────────────────────────────────────────────
# Prompt with Conversation History
# ─────────────────────────────────────────────────────

prompt_with_history = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    MessagesPlaceholder(variable_name="chat_history"),  # Dynamic history
    ("human", "{input}")
])
```

### Output Parsers — Structured Outputs

```python
# ═══════════════════════════════════════════════════════
# Getting Structured Data from LLMs
# ═══════════════════════════════════════════════════════

from langchain_core.output_parsers import JsonOutputParser
from langchain_core.pydantic_v1 import BaseModel, Field
from typing import List

# ─────────────────────────────────────────────────────
# Method 1: Pydantic Output Parser (Type-safe)
# ─────────────────────────────────────────────────────

class MovieReview(BaseModel):
    """A structured movie review."""
    title: str = Field(description="The movie title")
    rating: float = Field(description="Rating from 0 to 10")
    pros: List[str] = Field(description="List of positive aspects")
    cons: List[str] = Field(description="List of negative aspects")
    recommendation: bool = Field(description="Whether to recommend")

# Create parser
parser = JsonOutputParser(pydantic_object=MovieReview)

# Include format instructions in prompt
prompt = ChatPromptTemplate.from_messages([
    ("system", "Analyze movie reviews and provide structured analysis."),
    ("human", "Review this movie: {movie}\n\n{format_instructions}")
])

chain = prompt | llm | parser

# Invoke — returns a validated Pydantic dict
result = chain.invoke({
    "movie": "Inception",
    "format_instructions": parser.get_format_instructions()
})
# result = {"title": "Inception", "rating": 9.2, "pros": [...], ...}


# ─────────────────────────────────────────────────────
# Method 2: with_structured_output (Cleanest, uses function calling)
# ─────────────────────────────────────────────────────

# This uses OpenAI's function calling under the hood
structured_llm = llm.with_structured_output(MovieReview)

result = structured_llm.invoke("Review the movie Inception")
# Returns a MovieReview Pydantic object directly!
print(result.title)    # "Inception"
print(result.rating)   # 9.2
print(result.pros)     # ["Mind-bending plot", ...]
```

---

## Memory and Conversation History

### Why Memory Matters

LLMs are stateless — each API call is independent. Memory systems maintain conversation context across turns.

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEMORY TYPES                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. BUFFER MEMORY — Store all messages                           │
│     Pro: Complete context. Con: Token limit issues.              │
│     Use: Short conversations                                     │
│                                                                  │
│  2. WINDOW MEMORY — Store last K messages                        │
│     Pro: Bounded size. Con: Loses old context.                   │
│     Use: General chatbots                                        │
│                                                                  │
│  3. SUMMARY MEMORY — LLM summarizes history                     │
│     Pro: Compact. Con: Loses details, extra LLM calls.           │
│     Use: Long conversations                                      │
│                                                                  │
│  4. ENTITY MEMORY — Track facts about entities                   │
│     Pro: Structured. Con: Complex to implement.                  │
│     Use: Personal assistants                                     │
│                                                                  │
│  5. VECTOR STORE MEMORY — Embed & retrieve relevant history      │
│     Pro: Scales to huge histories. Con: May miss recent context. │
│     Use: Long-running agents                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Code: Memory Implementation

```python
# ═══════════════════════════════════════════════════════
# Conversation Memory in LangChain
# ═══════════════════════════════════════════════════════

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage
from langchain_community.chat_message_histories import ChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

llm = ChatOpenAI(model="gpt-4", temperature=0.7)

# ─────────────────────────────────────────────────────
# Modern Approach: RunnableWithMessageHistory
# ─────────────────────────────────────────────────────

# Prompt with history placeholder
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful AI assistant. Be concise."),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}")
])

chain = prompt | llm

# Store for session histories
session_store = {}

def get_session_history(session_id: str):
    """Get or create history for a session."""
    if session_id not in session_store:
        session_store[session_id] = ChatMessageHistory()
    return session_store[session_id]

# Wrap chain with history management
chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history"
)

# Use it — history is automatically managed
config = {"configurable": {"session_id": "user_123"}}

# Turn 1
response1 = chain_with_history.invoke(
    {"input": "My name is Alice and I'm a data scientist."},
    config=config
)
print(response1.content)

# Turn 2 — The model remembers!
response2 = chain_with_history.invoke(
    {"input": "What's my name and profession?"},
    config=config
)
print(response2.content)  # "Your name is Alice and you're a data scientist."


# ─────────────────────────────────────────────────────
# Production: Persistent Memory with Redis/Database
# ─────────────────────────────────────────────────────

from langchain_community.chat_message_histories import RedisChatMessageHistory

def get_redis_history(session_id: str):
    """Get session history from Redis (persists across restarts)."""
    return RedisChatMessageHistory(
        session_id=session_id,
        url="redis://localhost:6379"
    )

# Same pattern, just swap the history store
chain_with_persistent_history = RunnableWithMessageHistory(
    chain,
    get_redis_history,
    input_messages_key="input",
    history_messages_key="history"
)


# ─────────────────────────────────────────────────────
# Window Memory: Keep only last N messages
# ─────────────────────────────────────────────────────

def get_windowed_history(session_id: str, window_size: int = 10):
    """Keep only the last N messages."""
    history = get_session_history(session_id)
    messages = history.messages
    if len(messages) > window_size:
        # Keep only recent messages
        history.clear()
        for msg in messages[-window_size:]:
            history.add_message(msg)
    return history
```

---

## Tools and Function Calling

### What Are Tools?

Tools give LLMs the ability to interact with the real world — search the web, run code, query databases, call APIs, etc. The LLM decides WHEN to use a tool and WHAT inputs to pass.

```
┌─────────────────────────────────────────────────────────────────┐
│                HOW TOOL CALLING WORKS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  User: "What's the weather in London?"                           │
│                                                                  │
│  ┌──────────┐    "I need weather data"     ┌──────────────────┐ │
│  │   LLM    │ ──────────────────────────►  │ Tool: get_weather │ │
│  │          │    Call: get_weather("London")│ (API call)        │ │
│  │          │ ◄──────────────────────────── │ Returns: 15°C    │ │
│  │          │    Result: "15°C, cloudy"     └──────────────────┘ │
│  │          │                                                    │
│  └────┬─────┘                                                    │
│       │                                                          │
│       ▼                                                          │
│  "The weather in London is 15°C and cloudy."                     │
│                                                                  │
│  Key Insight: The LLM doesn't execute tools.                     │
│  It DECIDES which tool to call and with what arguments.          │
│  The framework executes the tool and returns results.            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Code: Defining and Using Tools

```python
# ═══════════════════════════════════════════════════════
# Tools in LangChain
# ═══════════════════════════════════════════════════════

from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from typing import Optional
import requests

# ─────────────────────────────────────────────────────
# Define Custom Tools
# ─────────────────────────────────────────────────────

@tool
def search_web(query: str) -> str:
    """Search the web for current information. Use this for questions about recent events or facts you're unsure about."""
    # In production, use a real search API (Tavily, SerpAPI, etc.)
    # This is a simplified example
    from tavily import TavilyClient
    client = TavilyClient(api_key="your-key")
    results = client.search(query, max_results=3)
    return "\n".join([r["content"] for r in results["results"]])

@tool
def calculator(expression: str) -> str:
    """Calculate a mathematical expression. Input should be a valid Python math expression."""
    try:
        # Safe evaluation (only math operations)
        import ast
        import operator
        
        # Define allowed operations
        allowed_ops = {
            ast.Add: operator.add,
            ast.Sub: operator.sub,
            ast.Mult: operator.mul,
            ast.Div: operator.truediv,
            ast.Pow: operator.pow,
        }
        
        result = eval(expression)  # In production, use a safer evaluator
        return str(result)
    except Exception as e:
        return f"Error: {str(e)}"

@tool
def get_weather(city: str, units: str = "celsius") -> str:
    """Get current weather for a city. Returns temperature and conditions."""
    # Simplified example — use a real weather API
    # Example with OpenWeatherMap
    api_key = "your-api-key"
    url = f"http://api.openweathermap.org/data/2.5/weather?q={city}&appid={api_key}&units=metric"
    response = requests.get(url)
    data = response.json()
    return f"{data['main']['temp']}°C, {data['weather'][0]['description']}"

@tool
def query_database(sql_query: str) -> str:
    """Execute a read-only SQL query against the company database. Only SELECT queries are allowed."""
    # SECURITY: Only allow SELECT queries
    if not sql_query.strip().upper().startswith("SELECT"):
        return "Error: Only SELECT queries are allowed."
    
    import sqlite3
    conn = sqlite3.connect("company.db")
    cursor = conn.cursor()
    cursor.execute(sql_query)
    results = cursor.fetchall()
    conn.close()
    return str(results)


# ─────────────────────────────────────────────────────
# Bind Tools to LLM
# ─────────────────────────────────────────────────────

llm = ChatOpenAI(model="gpt-4", temperature=0)

# Give the LLM access to tools
tools = [search_web, calculator, get_weather, query_database]
llm_with_tools = llm.bind_tools(tools)

# The LLM can now decide to use tools
response = llm_with_tools.invoke("What's 15% of 2847?")
print(response.tool_calls)
# [{'name': 'calculator', 'args': {'expression': '2847 * 0.15'}, 'id': '...'}]

# The LLM chose the calculator tool! But it didn't execute it.
# We need an agent or chain to execute and return the result.
```

---

## AI Agents — The Big Picture

### What is an Agent?

An Agent is an LLM that can **decide**, **act**, and **observe** in a loop until a task is complete. Unlike a chain (which follows a fixed sequence), an agent dynamically chooses what to do next based on the current situation.

```
┌─────────────────────────────────────────────────────────────────┐
│                    AGENT LOOP (ReAct Pattern)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────────┐                           │
│                    │   User Query    │                           │
│                    └────────┬────────┘                           │
│                             │                                    │
│                             ▼                                    │
│              ┌──────────────────────────────┐                    │
│              │         THINK                 │                    │
│              │  "What should I do next?"     │◄────────┐         │
│              │  (LLM reasons about state)    │         │         │
│              └──────────────┬───────────────┘         │         │
│                             │                          │         │
│                     ┌───────┴────────┐                 │         │
│                     │                │                 │         │
│                     ▼                ▼                 │         │
│           ┌──────────────┐  ┌──────────────┐          │         │
│           │  USE TOOL    │  │ FINAL ANSWER │          │         │
│           │  (Act)       │  │ (Done!)      │          │         │
│           └──────┬───────┘  └──────────────┘          │         │
│                  │                                     │         │
│                  ▼                                     │         │
│           ┌──────────────┐                            │         │
│           │  OBSERVE     │                            │         │
│           │  (Get result │────────────────────────────┘         │
│           │   from tool) │                                      │
│           └──────────────┘                                      │
│                                                                  │
│  This loop continues until the agent decides it has             │
│  enough information to provide a final answer.                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Agent vs Chain

| Aspect | Chain | Agent |
|--------|-------|-------|
| **Control flow** | Fixed, predetermined | Dynamic, decided at runtime |
| **Steps** | Always runs same steps | May run 1 or 100 steps |
| **Determinism** | Highly deterministic | Non-deterministic |
| **Error handling** | Fail at the broken step | Can try alternative approaches |
| **Complexity** | Simpler to debug | Harder to predict/debug |
| **Use when** | You know the steps upfront | Steps depend on intermediate results |

### Types of Agents

```
┌─────────────────────────────────────────────────────────────────┐
│                    AGENT ARCHITECTURES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. ReAct (Reasoning + Acting)                                   │
│     └─ Think → Act → Observe → Repeat                           │
│     └─ Most common, general-purpose                              │
│                                                                  │
│  2. Plan-and-Execute                                             │
│     └─ Plan all steps → Execute each → Replan if needed         │
│     └─ Better for complex multi-step tasks                       │
│                                                                  │
│  3. Multi-Agent                                                  │
│     └─ Multiple specialized agents collaborate                   │
│     └─ E.g., Researcher + Writer + Editor                       │
│                                                                  │
│  4. Reflexion                                                    │
│     └─ Execute → Evaluate → Reflect → Improve                   │
│     └─ Self-correcting behavior                                  │
│                                                                  │
│  5. Tool-Calling Agent (OpenAI style)                            │
│     └─ LLM outputs tool calls natively                           │
│     └─ Most efficient with supporting models                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Building Agents with LangChain

### Simple ReAct Agent

```python
# ═══════════════════════════════════════════════════════
# Building a ReAct Agent with LangChain
# ═══════════════════════════════════════════════════════

from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# ─────────────────────────────────────────────────────
# Step 1: Define Tools
# ─────────────────────────────────────────────────────

@tool
def search(query: str) -> str:
    """Search for current information on the web."""
    # Simplified — use Tavily or SerpAPI in production
    return f"Search results for '{query}': [Simulated results about {query}]"

@tool
def calculate(expression: str) -> float:
    """Evaluate a mathematical expression and return the result."""
    return eval(expression)  # Use a safe evaluator in production

@tool
def get_current_date() -> str:
    """Get today's date."""
    from datetime import date
    return str(date.today())

tools = [search, calculate, get_current_date]

# ─────────────────────────────────────────────────────
# Step 2: Create Agent Prompt
# ─────────────────────────────────────────────────────

prompt = ChatPromptTemplate.from_messages([
    ("system", """You are a helpful AI assistant with access to tools.
Use tools when you need current information or calculations.
Always think step by step before acting.
If you're unsure, use the search tool to verify."""),
    MessagesPlaceholder(variable_name="chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad")  # Agent's working memory
])

# ─────────────────────────────────────────────────────
# Step 3: Create Agent
# ─────────────────────────────────────────────────────

llm = ChatOpenAI(model="gpt-4", temperature=0)

# Create the agent (decides what to do)
agent = create_tool_calling_agent(llm, tools, prompt)

# Create the executor (runs the agent loop)
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,        # Print the agent's thinking process
    max_iterations=10,   # Prevent infinite loops
    handle_parsing_errors=True  # Gracefully handle malformed outputs
)

# ─────────────────────────────────────────────────────
# Step 4: Run the Agent
# ─────────────────────────────────────────────────────

# Simple query (no tools needed)
result = agent_executor.invoke({"input": "What's 2+2?"})
print(result["output"])

# Complex query (requires tools)
result = agent_executor.invoke({
    "input": "What's the current date, and if I invest $10000 at 7% annual return, how much will I have in 5 years?"
})
print(result["output"])
# The agent will:
# 1. Call get_current_date()
# 2. Call calculate("10000 * (1.07 ** 5)")
# 3. Combine results into a natural answer


# ─────────────────────────────────────────────────────
# Agent with Memory (Conversational Agent)
# ─────────────────────────────────────────────────────

from langchain_community.chat_message_histories import ChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

message_history = ChatMessageHistory()

agent_with_memory = RunnableWithMessageHistory(
    agent_executor,
    lambda session_id: message_history,
    input_messages_key="input",
    history_messages_key="chat_history"
)

# Conversation with memory
config = {"configurable": {"session_id": "test"}}

response1 = agent_with_memory.invoke(
    {"input": "My budget is $5000 for a new laptop."},
    config=config
)

response2 = agent_with_memory.invoke(
    {"input": "Search for the best options within my budget."},
    config=config
)
# Agent remembers the $5000 budget from previous turn!
```

### RAG Agent (Agent + Retrieval)

```python
# ═══════════════════════════════════════════════════════
# RAG Agent: Agent that searches a knowledge base
# ═══════════════════════════════════════════════════════

from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain.tools.retriever import create_retriever_tool
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# Load your vector store
vectorstore = Chroma(
    persist_directory="./chroma_db",
    embedding_function=OpenAIEmbeddings()
)

# Create retriever tool
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 5}
)

retriever_tool = create_retriever_tool(
    retriever,
    name="knowledge_base_search",
    description="Search the company knowledge base for policies, procedures, and product information. Use this whenever the user asks about company-specific information."
)

# Combine with other tools
tools = [retriever_tool, search, calculate]

# Create agent
prompt = ChatPromptTemplate.from_messages([
    ("system", """You are a customer support agent for TechCorp.
    
Rules:
1. Always search the knowledge base first for company-specific questions.
2. Use web search for general knowledge questions.
3. If you can't find the answer, say so honestly.
4. Always cite your sources."""),
    MessagesPlaceholder(variable_name="chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad")
])

llm = ChatOpenAI(model="gpt-4", temperature=0)
agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# The agent now intelligently routes between internal KB and web search
result = executor.invoke({"input": "What's our return policy for opened electronics?"})
```

---

## LangGraph — Stateful Agent Workflows

### What is LangGraph?

LangGraph is a library for building **stateful, multi-step agent workflows** as graphs. It's the recommended way to build production agents in the LangChain ecosystem.

**Key difference from basic agents**: LangGraph gives you explicit control over the flow, state management, persistence, and human-in-the-loop patterns.

```
┌─────────────────────────────────────────────────────────────────┐
│              LANGGRAPH vs AGENTEXECUTOR                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  AgentExecutor:                                                  │
│  • Simple loop: Think → Act → Observe                           │
│  • Limited control over flow                                     │
│  • Hard to add custom logic between steps                        │
│  • No built-in persistence                                       │
│                                                                  │
│  LangGraph:                                                      │
│  • Explicit graph with nodes and edges                           │
│  • Full control over state and transitions                       │
│  • Built-in checkpointing and persistence                        │
│  • Human-in-the-loop at any point                                │
│  • Parallel execution paths                                      │
│  • Cycles and conditional branching                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### LangGraph Concepts

```
┌─────────────────────────────────────────────────────────────────┐
│              LANGGRAPH BUILDING BLOCKS                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STATE — The shared data that flows through the graph            │
│  ┌──────────────────────────────────────────────┐               │
│  │ {"messages": [...], "context": "...",         │               │
│  │  "next_step": "...", "results": {...}}        │               │
│  └──────────────────────────────────────────────┘               │
│                                                                  │
│  NODES — Functions that process/transform state                  │
│  [agent] [tool_executor] [evaluator] [human_review]             │
│                                                                  │
│  EDGES — Connections between nodes                               │
│  • Normal edge: A → B (always)                                   │
│  • Conditional edge: A → B or C (based on state)                 │
│                                                                  │
│  CHECKPOINTER — Saves state for persistence/resume              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Code: Building a LangGraph Agent

```python
# ═══════════════════════════════════════════════════════
# LangGraph: Production-Ready Agent
# ═══════════════════════════════════════════════════════

from typing import TypedDict, Annotated, Sequence
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage, ToolMessage
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver
import operator

# ─────────────────────────────────────────────────────
# Step 1: Define State
# ─────────────────────────────────────────────────────

class AgentState(TypedDict):
    """The state that flows through the graph."""
    messages: Annotated[Sequence[BaseMessage], operator.add]  # Append messages

# ─────────────────────────────────────────────────────
# Step 2: Define Tools
# ─────────────────────────────────────────────────────

@tool
def search_knowledge_base(query: str) -> str:
    """Search internal knowledge base for company information."""
    # Your retrieval logic here
    return f"Found: Information about {query}"

@tool
def calculate(expression: str) -> str:
    """Calculate a mathematical expression."""
    return str(eval(expression))

@tool
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email. Use with caution — this actually sends."""
    # In production, actually send the email
    return f"Email sent to {to} with subject '{subject}'"

tools = [search_knowledge_base, calculate, send_email]

# ─────────────────────────────────────────────────────
# Step 3: Define Nodes (processing functions)
# ─────────────────────────────────────────────────────

# LLM with tools bound
llm = ChatOpenAI(model="gpt-4", temperature=0).bind_tools(tools)

def agent_node(state: AgentState) -> dict:
    """The agent decides what to do next."""
    messages = state["messages"]
    response = llm.invoke(messages)
    return {"messages": [response]}

# Tool execution node (built-in helper)
tool_node = ToolNode(tools)

# ─────────────────────────────────────────────────────
# Step 4: Define Routing Logic
# ─────────────────────────────────────────────────────

def should_continue(state: AgentState) -> str:
    """Decide whether to continue to tools or end."""
    last_message = state["messages"][-1]
    
    # If the LLM made tool calls, execute them
    if last_message.tool_calls:
        return "tools"
    
    # Otherwise, we're done
    return "end"

# ─────────────────────────────────────────────────────
# Step 5: Build the Graph
# ─────────────────────────────────────────────────────

# Create the graph
workflow = StateGraph(AgentState)

# Add nodes
workflow.add_node("agent", agent_node)
workflow.add_node("tools", tool_node)

# Set entry point
workflow.set_entry_point("agent")

# Add edges
workflow.add_conditional_edges(
    "agent",                    # From node
    should_continue,            # Decision function
    {
        "tools": "tools",      # If "tools" → go to tools node
        "end": END             # If "end" → stop
    }
)
workflow.add_edge("tools", "agent")  # After tools → back to agent

# ─────────────────────────────────────────────────────
# Step 6: Compile with Memory (Checkpointing)
# ─────────────────────────────────────────────────────

# Memory saver allows conversation persistence
memory = MemorySaver()
app = workflow.compile(checkpointer=memory)

# ─────────────────────────────────────────────────────
# Step 7: Run the Agent
# ─────────────────────────────────────────────────────

# Thread config for conversation tracking
config = {"configurable": {"thread_id": "conversation_1"}}

# Turn 1
result = app.invoke(
    {"messages": [HumanMessage(content="What's 25 * 47?")]},
    config=config
)
print(result["messages"][-1].content)

# Turn 2 (remembers previous conversation due to checkpointing)
result = app.invoke(
    {"messages": [HumanMessage(content="Now add 100 to that result")]},
    config=config
)
print(result["messages"][-1].content)


# ─────────────────────────────────────────────────────
# Streaming (see agent's thought process in real-time)
# ─────────────────────────────────────────────────────

for event in app.stream(
    {"messages": [HumanMessage(content="Search for our refund policy and summarize it")]},
    config=config
):
    for node_name, output in event.items():
        print(f"\n--- {node_name} ---")
        if "messages" in output:
            for msg in output["messages"]:
                print(f"  {msg.type}: {msg.content[:100]}...")
```

### LangGraph: Human-in-the-Loop

```python
# ═══════════════════════════════════════════════════════
# Human-in-the-Loop: Require approval before actions
# ═══════════════════════════════════════════════════════

from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver

class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    requires_approval: bool

def agent_node(state: AgentState) -> dict:
    """Agent decides what to do."""
    response = llm.invoke(state["messages"])
    
    # Check if the action requires approval
    requires_approval = False
    if response.tool_calls:
        for call in response.tool_calls:
            if call["name"] in ["send_email", "delete_record", "make_payment"]:
                requires_approval = True
                break
    
    return {"messages": [response], "requires_approval": requires_approval}

def should_continue(state: AgentState) -> str:
    last_message = state["messages"][-1]
    
    if not last_message.tool_calls:
        return "end"
    
    if state.get("requires_approval"):
        return "human_review"  # Pause for human approval
    
    return "tools"

# Build graph with human review node
workflow = StateGraph(AgentState)
workflow.add_node("agent", agent_node)
workflow.add_node("tools", tool_node)
workflow.set_entry_point("agent")

workflow.add_conditional_edges(
    "agent",
    should_continue,
    {"tools": "tools", "human_review": "tools", "end": END}  
)
workflow.add_edge("tools", "agent")

# Compile with interrupt_before for human review
app = workflow.compile(
    checkpointer=MemorySaver(),
    interrupt_before=["tools"]  # Pause before executing tools
)

# Run — will pause at the tool node if approval needed
config = {"configurable": {"thread_id": "thread_1"}}
result = app.invoke(
    {"messages": [HumanMessage(content="Send an email to boss@company.com about the project update")]},
    config=config
)

# Check state and get approval
state = app.get_state(config)
print(f"Pending action: {state.values['messages'][-1].tool_calls}")

# If approved, continue execution
app.invoke(None, config=config)  # Resume from where it paused
```

### LangGraph: Multi-Agent System

```python
# ═══════════════════════════════════════════════════════
# Multi-Agent: Researcher + Writer working together
# ═══════════════════════════════════════════════════════

from typing import TypedDict, Annotated, Literal
from langchain_core.messages import BaseMessage, HumanMessage
from langgraph.graph import StateGraph, END
import operator

class MultiAgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    research_notes: str
    draft: str
    current_agent: str

# Specialized LLMs with different system prompts
researcher_llm = ChatOpenAI(model="gpt-4", temperature=0.3)
writer_llm = ChatOpenAI(model="gpt-4", temperature=0.7)
editor_llm = ChatOpenAI(model="gpt-4", temperature=0.2)

def researcher_node(state: MultiAgentState) -> dict:
    """Research agent: gathers information."""
    prompt = f"""You are a research assistant. 
    Task: {state['messages'][-1].content}
    
    Provide comprehensive research notes with key facts, statistics, and sources."""
    
    response = researcher_llm.invoke([HumanMessage(content=prompt)])
    return {
        "research_notes": response.content,
        "messages": [AIMessage(content=f"[Researcher]: Research complete.")],
        "current_agent": "writer"
    }

def writer_node(state: MultiAgentState) -> dict:
    """Writer agent: creates content from research."""
    prompt = f"""You are a professional writer.
    
    Based on these research notes, write a well-structured article:
    
    Research: {state['research_notes']}
    
    Original request: {state['messages'][0].content}"""
    
    response = writer_llm.invoke([HumanMessage(content=prompt)])
    return {
        "draft": response.content,
        "messages": [AIMessage(content=f"[Writer]: Draft complete.")],
        "current_agent": "editor"
    }

def editor_node(state: MultiAgentState) -> dict:
    """Editor agent: reviews and improves the draft."""
    prompt = f"""You are a meticulous editor. 
    Review and improve this draft. Fix any issues with clarity, accuracy, or flow.
    
    Draft: {state['draft']}"""
    
    response = editor_llm.invoke([HumanMessage(content=prompt)])
    return {
        "draft": response.content,
        "messages": [AIMessage(content=response.content)],
        "current_agent": "done"
    }

def route_agent(state: MultiAgentState) -> str:
    """Route to the next agent."""
    next_agent = state.get("current_agent", "researcher")
    if next_agent == "done":
        return "end"
    return next_agent

# Build multi-agent graph
workflow = StateGraph(MultiAgentState)
workflow.add_node("researcher", researcher_node)
workflow.add_node("writer", writer_node)
workflow.add_node("editor", editor_node)

workflow.set_entry_point("researcher")
workflow.add_edge("researcher", "writer")
workflow.add_edge("writer", "editor")
workflow.add_edge("editor", END)

app = workflow.compile()

# Run the multi-agent pipeline
result = app.invoke({
    "messages": [HumanMessage(content="Write an article about the impact of AI on healthcare in 2024")],
    "research_notes": "",
    "draft": "",
    "current_agent": "researcher"
})

print(result["draft"])  # Final edited article
```

---

## Production Patterns

### 1. Error Handling and Fallbacks

```python
# ═══════════════════════════════════════════════════════
# Production-Ready Error Handling
# ═══════════════════════════════════════════════════════

from langchain_core.runnables import RunnableWithFallbacks
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

# Primary model with fallback
primary = ChatOpenAI(model="gpt-4", temperature=0)
fallback = ChatAnthropic(model="claude-3-sonnet-20240229")

# If GPT-4 fails (rate limit, timeout, etc.), try Claude
llm_with_fallback = primary.with_fallbacks([fallback])

# Retry with exponential backoff
from langchain_core.runnables import RunnableConfig

config = RunnableConfig(
    max_retries=3,            # Retry up to 3 times
    retry_if_exception_type=(Exception,)  # Retry on any exception
)
```

### 2. Observability with LangSmith

```python
# ═══════════════════════════════════════════════════════
# Tracing and Monitoring with LangSmith
# ═══════════════════════════════════════════════════════

import os

# Enable LangSmith tracing (set env vars)
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-langsmith-api-key"
os.environ["LANGCHAIN_PROJECT"] = "my-production-app"

# Now ALL LangChain calls are automatically traced!
# You can see:
# - Input/output of every step
# - Latency per component
# - Token usage and cost
# - Error traces
# - Full conversation flows

# Custom metadata for traces
from langchain_core.runnables import RunnableConfig

config = RunnableConfig(
    metadata={
        "user_id": "user_123",
        "session_type": "support",
        "version": "2.1.0"
    },
    tags=["production", "customer-support"]
)

result = chain.invoke({"input": "question"}, config=config)
```

### 3. Rate Limiting and Caching

```python
# ═══════════════════════════════════════════════════════
# Caching: Don't repeat expensive LLM calls
# ═══════════════════════════════════════════════════════

from langchain_core.globals import set_llm_cache
from langchain_community.cache import InMemoryCache, RedisCache

# Simple in-memory cache
set_llm_cache(InMemoryCache())

# Production: Redis cache (persists across restarts)
# set_llm_cache(RedisCache(redis_url="redis://localhost:6379"))

# Now identical prompts return cached results instantly!
# First call: hits API (~2s)
result1 = llm.invoke("What is 2+2?")
# Second call: returns from cache (~0ms)
result2 = llm.invoke("What is 2+2?")
```

### 4. Streaming for User Experience

```python
# ═══════════════════════════════════════════════════════
# Streaming responses for better UX
# ═══════════════════════════════════════════════════════

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatOpenAI(model="gpt-4", temperature=0.7, streaming=True)
prompt = ChatPromptTemplate.from_template("Write a story about {topic}")
chain = prompt | llm | StrOutputParser()

# Stream tokens as they're generated
async def stream_response(topic: str):
    """Stream response for a web application."""
    async for chunk in chain.astream({"topic": topic}):
        yield chunk  # Send each token to the frontend

# FastAPI streaming endpoint example
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.get("/chat/stream")
async def chat_stream(question: str):
    return StreamingResponse(
        stream_response(question),
        media_type="text/plain"
    )
```

---

## Common Mistakes

### 1. Over-Engineering with LangChain

**Mistake**: Using LangChain for a simple one-off API call.
**Fix**: If you just need `openai.ChatCompletion.create()`, use the SDK directly. LangChain is for complex, multi-step workflows.

### 2. Not Handling Agent Loops

**Mistake**: Agent gets stuck in infinite tool-calling loops.
**Fix**: Always set `max_iterations` in AgentExecutor, implement timeout logic, and add "give up" conditions.

```python
# ALWAYS set a max iterations limit
executor = AgentExecutor(
    agent=agent,
    tools=tools,
    max_iterations=15,          # Hard stop
    max_execution_time=60,      # 60 second timeout
    early_stopping_method="generate"  # LLM generates final answer if stuck
)
```

### 3. Ignoring Token Costs

**Mistake**: Agent makes 20 tool calls with full context each time, costing $5 per query.
**Fix**: Monitor token usage, implement budgets, use smaller models for intermediate steps.

### 4. Poor Tool Descriptions

**Mistake**: Vague tool descriptions cause the LLM to use wrong tools.
**Fix**: Write clear, specific descriptions with examples of when to use each tool.

```python
# BAD — vague
@tool
def search(query: str) -> str:
    """Search for stuff."""
    ...

# GOOD — specific
@tool
def search_product_catalog(query: str) -> str:
    """Search the product catalog for items matching the query.
    Use this when the user asks about product availability, pricing, or specifications.
    Do NOT use this for general knowledge questions — use web_search for those.
    
    Args:
        query: Product name, category, or feature to search for.
    """
    ...
```

### 5. Not Testing Agent Behavior

**Mistake**: Deploying agents without testing edge cases.
**Fix**: Create evaluation datasets with expected tool usage patterns, edge cases, and adversarial inputs.

### 6. Monolithic Agent

**Mistake**: One agent with 20 tools trying to do everything.
**Fix**: Break into specialized sub-agents (LangGraph multi-agent) with 3-5 tools each.

### 7. Not Using Structured Output

**Mistake**: Parsing free-text LLM output with regex.
**Fix**: Use `with_structured_output()` or function calling for reliable structured data.

---

## Interview Questions

### Conceptual

**Q1: What is the difference between a chain and an agent?**
> A chain follows a fixed sequence of steps regardless of intermediate results. An agent dynamically decides what action to take next based on observations. Chains are deterministic; agents are non-deterministic. Use chains when you know the workflow upfront, agents when the path depends on intermediate results.

**Q2: Explain the ReAct pattern.**
> ReAct (Reasoning + Acting) interleaves reasoning (thinking about what to do) with acting (using tools) and observing (reading tool results). The loop is: Thought → Action → Observation → Thought → ... until the agent has enough information for a final answer. It was introduced by Yao et al. (2022).

**Q3: What is LangGraph and when would you use it over AgentExecutor?**
> LangGraph models agent workflows as state machines (graphs with nodes and edges). Use it when you need: explicit control over the agent loop, persistence/checkpointing, human-in-the-loop approvals, parallel execution paths, multi-agent coordination, or cyclic workflows. AgentExecutor is simpler but less flexible.

**Q4: How do you prevent an agent from taking dangerous actions?**
> (1) Don't give tools that can be dangerous, (2) Implement human-in-the-loop for risky actions (LangGraph interrupt_before), (3) Tools should validate inputs and have guardrails, (4) Use read-only tools where possible, (5) Set max_iterations/timeout, (6) Monitor with LangSmith and alert on anomalies.

**Q5: How does memory work in a chatbot built with LangChain?**
> Messages are stored in a message history (in-memory, Redis, database). On each turn, the history is injected into the prompt via MessagesPlaceholder. Strategies include: full buffer (all messages), window (last K), summary (LLM-generated summary), and vector store (embed and retrieve relevant history). RunnableWithMessageHistory manages this automatically.

### System Design

**Q6: Design an AI customer support system using LangChain/LangGraph.**
> Architecture: LangGraph workflow with states: classify_intent → route → {FAQ_agent, Technical_agent, Escalation_agent}. Each sub-agent has access to relevant tools (KB search, order lookup, ticket creation). Include human-in-the-loop for refunds > $100. Use Redis for conversation persistence, LangSmith for monitoring, with fallback to human agent after 3 failed attempts.

**Q7: How would you evaluate an agent's performance?**
> (1) Create test cases with expected tool usage sequences, (2) Measure: task completion rate, average steps to complete, cost per task, latency, (3) Use LLM-as-judge to evaluate answer quality, (4) Track tool call accuracy (did it use the right tool with right args?), (5) A/B test against simpler alternatives, (6) Monitor in production with LangSmith for degradation.

**Q8: Your agent is slow (10+ seconds per response). How do you optimize?**
> (1) Use streaming for perceived speed, (2) Cache frequent queries, (3) Use smaller/faster models for routing/classification steps, (4) Parallelize independent tool calls, (5) Pre-compute embeddings and keep vector DB warm, (6) Reduce context size (shorter prompts, summarize history), (7) Use function calling instead of text-based tool parsing, (8) Consider replacing agent with deterministic chain where possible.

---

## Quick Reference

### LangChain Component Cheat Sheet

```python
# ═══════════════════════════════════════════════════════
# LANGCHAIN QUICK REFERENCE
# ═══════════════════════════════════════════════════════

# --- Imports ---
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableParallel
from langchain_core.tools import tool
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_community.vectorstores import Chroma
from langgraph.graph import StateGraph, END

# --- Basic Chain ---
chain = prompt | llm | parser

# --- RAG Chain ---
rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt | llm | StrOutputParser()
)

# --- Agent ---
agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools)

# --- LangGraph ---
graph = StateGraph(State)
graph.add_node("name", function)
graph.add_edge("from", "to")
app = graph.compile(checkpointer=MemorySaver())
```

### Decision Matrix: What to Build

| Need | Solution |
|------|----------|
| Simple Q&A | Single chain (prompt → LLM → parser) |
| Chat with memory | Chain + RunnableWithMessageHistory |
| Q&A over documents | RAG chain (retriever + prompt + LLM) |
| Dynamic tool use | Agent (create_tool_calling_agent) |
| Complex workflow | LangGraph (explicit state machine) |
| Multi-agent | LangGraph with multiple agent nodes |
| Human approval needed | LangGraph with interrupt_before |
| Production monitoring | LangSmith tracing |

### Architecture Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│              COMMON PRODUCTION ARCHITECTURES                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Pattern 1: RAG Chatbot                                          │
│  User → Memory → Retriever → Prompt → LLM → Response            │
│                                                                  │
│  Pattern 2: Tool-Using Agent                                     │
│  User → Agent Loop(Think → Tool → Observe) → Response            │
│                                                                  │
│  Pattern 3: Multi-Agent Pipeline                                 │
│  User → Router → [Specialist A | B | C] → Synthesizer           │
│                                                                  │
│  Pattern 4: Human-in-the-Loop                                    │
│  User → Agent → [Risky Action? → Human Approval] → Execute      │
│                                                                  │
│  Pattern 5: Plan-Execute                                         │
│  User → Planner → [Step1, Step2, Step3] → Executor → Result     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Libraries & Versions

| Package | Purpose | Install |
|---------|---------|---------|
| `langchain` | Core framework | `pip install langchain` |
| `langchain-openai` | OpenAI integration | `pip install langchain-openai` |
| `langchain-anthropic` | Anthropic integration | `pip install langchain-anthropic` |
| `langchain-community` | Third-party integrations | `pip install langchain-community` |
| `langgraph` | Stateful agent graphs | `pip install langgraph` |
| `langsmith` | Tracing & evaluation | `pip install langsmith` |
| `chromadb` | Vector store | `pip install chromadb` |

---

*Previous Chapter: [05-RAG-Retrieval-Augmented-Generation](05-RAG-Retrieval-Augmented-Generation.md)*
*Next Chapter: [07-Diffusion-Models](07-Diffusion-Models.md)*
