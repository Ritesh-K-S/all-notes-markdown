# LangGraph Python - Complete Notes

> **Version**: LangGraph 1.1.x (latest stable as of April 2026)
> **Python Requirement**: Python 3.10+
> **Official Docs**: https://docs.langchain.com/oss/python/langgraph/overview

---

## Table of Contents

1. [Introduction & Architecture](#1-introduction--architecture)
2. [Installation & Setup](#2-installation--setup)
3. [Core Concepts Overview](#3-core-concepts-overview)
4. [Graph API — StateGraph](#4-graph-api--stategraph)
5. [State & State Schema](#5-state--state-schema)
6. [Reducers](#6-reducers)
7. [Working with Messages](#7-working-with-messages)
8. [Nodes](#8-nodes)
9. [Edges](#9-edges)
10. [Command — Control Flow + State Updates](#10-command--control-flow--state-updates)
11. [Send — Dynamic Fan-out / Map-Reduce](#11-send--dynamic-fan-out--map-reduce)
12. [Compiling a Graph](#12-compiling-a-graph)
13. [Invoking & Running Graphs](#13-invoking--running-graphs)
14. [Functional API (@entrypoint & @task)](#14-functional-api-entrypoint--task)
15. [Persistence & Checkpointing](#15-persistence--checkpointing)
16. [Threads & State Management](#16-threads--state-management)
17. [Short-Term Memory (Conversation Memory)](#17-short-term-memory-conversation-memory)
18. [Long-Term Memory (Memory Store)](#18-long-term-memory-memory-store)
19. [Streaming](#19-streaming)
20. [Interrupts & Human-in-the-Loop](#20-interrupts--human-in-the-loop)
21. [Durable Execution](#21-durable-execution)
22. [Subgraphs](#22-subgraphs)
23. [Runtime Context & Configuration](#23-runtime-context--configuration)
24. [Recursion Limit & Step Counter](#24-recursion-limit--step-counter)
25. [Node Caching](#25-node-caching)
26. [Graph Migrations](#26-graph-migrations)
27. [Visualization](#27-visualization)
28. [LangSmith Integration (Tracing & Observability)](#28-langsmith-integration-tracing--observability)
29. [Deployment & Production](#29-deployment--production)
30. [Common Patterns & Best Practices](#30-common-patterns--best-practices)
31. [Choosing Graph API vs Functional API](#31-choosing-graph-api-vs-functional-api)

---

## 1. Introduction & Architecture

### What is LangGraph?

LangGraph is a **low-level orchestration framework** for building, managing, and deploying **long-running, stateful agents**. It does NOT abstract prompts or architecture — it provides the underlying infrastructure for agent workflows.

### Key Characteristics

- Models agent workflows as **graphs** (nodes + edges)
- Provides **durable execution** — agents persist through failures
- Built-in **human-in-the-loop** support
- Comprehensive **memory** (short-term + long-term)
- **Streaming** support for real-time output
- Production-ready deployment

### Relationship with LangChain

| Component | Purpose |
|-----------|---------|
| **LangGraph** | Low-level agent orchestration (graphs, state, persistence) |
| **LangChain** | High-level framework — models, tools, agents (uses LangGraph under the hood) |
| **LangSmith** | Tracing, debugging, evaluation, deployment |

> LangGraph can be used **without** LangChain. It is standalone.

### Inspiration

- Inspired by Google's **Pregel** and **Apache Beam**
- Public interface draws from **NetworkX**

---

## 2. Installation & Setup

### Basic Installation

```bash
pip install -U langgraph
```

### With Checkpointer Backends

```bash
# SQLite checkpointer
pip install langgraph-checkpoint-sqlite

# Postgres checkpointer (production)
pip install langgraph-checkpoint-postgres

# Azure Cosmos DB
pip install langgraph-checkpoint-cosmosdb
```

### With LangChain (commonly used together)

```bash
pip install langchain langchain-openai langchain-anthropic
```

### Verify Installation

```python
from langgraph.graph import StateGraph, START, END, MessagesState

def mock_llm(state: MessagesState):
    return {"messages": [{"role": "ai", "content": "hello world"}]}

graph = StateGraph(MessagesState)
graph.add_node(mock_llm)
graph.add_edge(START, "mock_llm")
graph.add_edge("mock_llm", END)
graph = graph.compile()

result = graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
print(result)
```

---

## 3. Core Concepts Overview

LangGraph models agent workflows as **graphs** with three key components:

| Component | Description |
|-----------|-------------|
| **State** | Shared data structure — the current snapshot of your application |
| **Nodes** | Functions that encode agent logic — receive state, return updated state |
| **Edges** | Functions that determine which Node executes next |

### Execution Model — Super-Steps

- Uses **message passing** (inspired by Pregel)
- Executes in discrete **super-steps** (iterations over graph nodes)
- Nodes in same super-step run **in parallel**
- Nodes in different super-steps run **sequentially**
- A node is **active** when it receives a message; otherwise it's **inactive**
- Execution terminates when all nodes are inactive and no messages are in transit

> **Key insight**: Nodes do the work, edges tell what to do next.

---

## 4. Graph API — StateGraph

### StateGraph

`StateGraph` is the main class for defining graph-based workflows.

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

class State(TypedDict):
    text: str

def node_a(state: State) -> dict:
    return {"text": state["text"] + "a"}

def node_b(state: State) -> dict:
    return {"text": state["text"] + "b"}

# Build the graph
graph = StateGraph(State)
graph.add_node("node_a", node_a)
graph.add_node("node_b", node_b)
graph.add_edge(START, "node_a")
graph.add_edge("node_a", "node_b")
graph.add_edge("node_b", END)

# Compile and run
compiled = graph.compile()
result = compiled.invoke({"text": ""})
print(result)  # {'text': 'ab'}
```

### Workflow

1. Define the **State** schema
2. Add **Nodes** (functions)
3. Add **Edges** (connections between nodes)
4. **Compile** the graph
5. **Invoke** or **stream** execution

---

## 5. State & State Schema

### Definition

State is the shared data structure passed between all nodes and edges. Nodes emit **updates** to the state.

### Schema Options

| Schema Type | Use Case |
|-------------|----------|
| `TypedDict` | Default, most performant |
| `dataclass` | When you need default values |
| `Pydantic BaseModel` | When you need recursive data validation (slower) |

### TypedDict State (Most Common)

```python
from typing_extensions import TypedDict

class State(TypedDict):
    foo: int
    bar: list[str]
```

### Dataclass State

```python
from dataclasses import dataclass, field

@dataclass
class State:
    foo: int = 0
    bar: list[str] = field(default_factory=list)
```

### Pydantic State

```python
from pydantic import BaseModel

class State(BaseModel):
    foo: int
    bar: list[str]
```

### Node Return — Partial Updates

Nodes do NOT need to return the full state — only the keys they want to update:

```python
def node_a(state: State):
    # Only updates 'foo', 'bar' remains unchanged
    return {"foo": 2}
```

### Multiple Schemas (Input/Output/Private)

You can define separate schemas for input, output, and internal state:

```python
class InputState(TypedDict):
    user_input: str

class OutputState(TypedDict):
    graph_output: str

class OverallState(TypedDict):
    foo: str
    user_input: str
    graph_output: str

class PrivateState(TypedDict):
    bar: str  # Only used internally between nodes

def node_1(state: InputState) -> OverallState:
    return {"foo": state["user_input"] + " name"}

def node_2(state: OverallState) -> PrivateState:
    return {"bar": state["foo"] + " is"}

def node_3(state: PrivateState) -> OutputState:
    return {"graph_output": state["bar"] + " Lance"}

builder = StateGraph(
    OverallState,
    input_schema=InputState,
    output_schema=OutputState
)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", "node_3")
builder.add_edge("node_3", END)

graph = builder.compile()
result = graph.invoke({"user_input": "My"})
# {'graph_output': 'My name is Lance'}
```

**Key rules**:
1. A node can **write** to any state channel in the graph state (union of all schemas)
2. Nodes can declare **additional state channels** as long as their schema is defined

---

## 6. Reducers

### What Are Reducers?

Reducers define **how updates from nodes are applied** to each state key. Each key has its own independent reducer function.

### Default Behavior (No Reducer)

Without a reducer, updates **overwrite** the existing value:

```python
class State(TypedDict):
    foo: int
    bar: list[str]

# If node returns {"bar": ["bye"]}, bar becomes ["bye"] (replaced, not appended)
```

### Using Annotated Reducers

```python
from typing import Annotated
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]  # <-- add reducer

# If state is {"bar": ["hi"]} and node returns {"bar": ["bye"]}
# Result: {"bar": ["hi", "bye"]}  (lists are concatenated)
```

### Common Reducer: `operator.add`

Used for accumulating lists:

```python
from typing import Annotated
import operator

class State(TypedDict):
    messages: Annotated[list, operator.add]
```

### Custom Reducer Functions

Any function `(existing, new) -> merged` can be a reducer:

```python
def keep_last_n(existing: list, new: list) -> list:
    combined = existing + new
    return combined[-10:]  # Keep only last 10

class State(TypedDict):
    history: Annotated[list, keep_last_n]
```

### Overwrite (Bypass Reducer)

Use `Overwrite` type to bypass a reducer and directly replace a value:

```python
from langgraph.types import Overwrite

# To bypass reducer: return Overwrite(value)
```

---

## 7. Working with Messages

### Why Messages?

Most LLM providers use a chat interface accepting message lists. LangChain message types:

| Type | Purpose |
|------|---------|
| `HumanMessage` | User input |
| `AIMessage` | LLM response |
| `SystemMessage` | System instruction |
| `ToolMessage` | Tool call result |
| `AnyMessage` | Union of all message types |

### Using `add_messages` Reducer

`add_messages` is a prebuilt reducer that:
- **Appends** new messages
- **Updates** existing messages by matching message IDs
- **Deserializes** dict messages into LangChain Message objects

```python
from langchain.messages import AnyMessage
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict

class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

### Prebuilt `MessagesState`

For convenience, LangGraph provides a prebuilt state with a `messages` key:

```python
from langgraph.graph import MessagesState

# Equivalent to:
# class MessagesState(TypedDict):
#     messages: Annotated[list[AnyMessage], add_messages]

# Extend with additional fields:
class State(MessagesState):
    documents: list[str]
    summary: str
```

### Message Input Formats

Both formats are supported:

```python
# LangChain message objects
{"messages": [HumanMessage(content="hello")]}

# Dict format (auto-deserialized)
{"messages": [{"type": "human", "content": "hello"}]}
```

### Accessing Message Attributes

Use **dot notation** since messages are LangChain objects:

```python
state["messages"][-1].content   # ✅ Correct
state["messages"][-1]["content"] # ❌ Incorrect
```

---

## 8. Nodes

### Definition

Nodes are **Python functions** (sync or async) that accept:

1. `state` — the current graph state
2. `config` (optional) — `RunnableConfig` with `thread_id`, tracing info, etc.
3. `runtime` (optional) — `Runtime` object with `store`, `stream_writer`, `execution_info`, etc.

```python
from langgraph.graph import StateGraph
from langgraph.runtime import Runtime
from dataclasses import dataclass
from typing_extensions import TypedDict

class State(TypedDict):
    input: str
    results: str

@dataclass
class Context:
    user_id: str

builder = StateGraph(State)

# Simple node
def plain_node(state: State):
    return state

# Node with runtime access
def node_with_runtime(state: State, runtime: Runtime[Context]):
    print("User:", runtime.context.user_id)
    return {"results": f"Hello, {state['input']}!"}

# Node with execution info
def node_with_execution_info(state: State, runtime: Runtime):
    print("Thread:", runtime.execution_info.thread_id)
    return {"results": f"Hello, {state['input']}!"}

builder.add_node("plain_node", plain_node)
builder.add_node("node_with_runtime", node_with_runtime)
```

### Auto-Naming

If no name is provided, the function name is used:

```python
builder.add_node(my_node)
# Can reference as "my_node" in edges
```

### Special Nodes

| Node | Import | Purpose |
|------|--------|---------|
| `START` | `from langgraph.graph import START` | Entry point — first node to execute |
| `END` | `from langgraph.graph import END` | Terminal node — graph is complete |

```python
from langgraph.graph import START, END

graph.add_edge(START, "node_a")  # node_a runs first
graph.add_edge("node_b", END)   # graph ends after node_b
```

---

## 9. Edges

### Edge Types

| Type | Description |
|------|-------------|
| **Normal edge** | Fixed transition from A to B |
| **Conditional edge** | Calls a function to determine next node(s) |
| **Entry point** | First node via `START` |
| **Conditional entry point** | Dynamic first node via `START` + routing function |

### Normal Edges

```python
graph.add_edge("node_a", "node_b")
```

### Conditional Edges

```python
from typing import Literal
from langgraph.graph import END

def routing_function(state: State) -> Literal["node_b", "node_c"]:
    if state["condition"]:
        return "node_b"
    return "node_c"

graph.add_conditional_edges("node_a", routing_function)
```

With explicit mapping:

```python
graph.add_conditional_edges(
    "node_a",
    routing_function,
    {True: "node_b", False: "node_c"}
)
```

### Entry Point

```python
from langgraph.graph import START
graph.add_edge(START, "node_a")
```

### Conditional Entry Point

```python
graph.add_conditional_edges(START, routing_function)
```

### Parallel Execution

A node with **multiple outgoing edges** causes all destination nodes to run **in parallel** (same super-step):

```python
graph.add_edge(START, "node_a")
graph.add_edge(START, "node_b")
# Both node_a and node_b run in parallel
```

---

## 10. Command — Control Flow + State Updates

### What is Command?

`Command` combines **state updates** + **routing** in a single return value. It accepts:

| Parameter | Purpose |
|-----------|---------|
| `update` | Apply state updates |
| `goto` | Navigate to specific node(s) |
| `graph` | Target a parent graph (from subgraphs) |
| `resume` | Resume after an interrupt |

### Return from Nodes — `update` and `goto`

```python
from langgraph.types import Command
from typing import Literal

def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        update={"foo": "bar"},     # state update
        goto="my_other_node"       # control flow
    )
```

### Dynamic Routing (replaces conditional edges)

```python
def my_node(state: State) -> Command[Literal["node_b", "node_c"]]:
    if state["foo"] == "bar":
        return Command(update={"foo": "baz"}, goto="node_b")
    else:
        return Command(goto="node_c")
```

> **Important**: You MUST add `Command[Literal["node_names"]]` as the return type annotation. This is required for graph rendering.

> **Warning**: `Command` only adds **dynamic** edges. Static edges defined with `add_edge` still execute.

### Navigate to Parent Graph (from Subgraphs)

```python
def my_node(state: State) -> Command[Literal["other_subgraph"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",
        graph=Command.PARENT
    )
```

### Return from Tools

```python
from langchain.tools import tool
from langgraph.types import Command

@tool
def my_tool(query: str) -> Command:
    result = lookup(query)
    return Command(
        update={"info": result},
        goto="next_node"
    )
```

### As Input — `resume` (for Interrupts)

```python
from langgraph.types import Command

# Resume after an interrupt
result = graph.invoke(Command(resume="yes"), config)
```

> **Warning**: `Command(resume=...)` is the ONLY Command pattern for input to `invoke`/`stream`. Do NOT use `Command(update=...)` for multi-turn conversations — use a plain dict instead.

---

## 11. Send — Dynamic Fan-out / Map-Reduce

### What is Send?

`Send` creates **dynamic edges** at runtime — useful when the number of edges is unknown ahead of time.

```python
from langgraph.types import Send
from typing_extensions import TypedDict

class OverallState(TypedDict):
    subjects: list[str]
    jokes: list[str]

class JokeState(TypedDict):
    subject: str

def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state['subjects']]

def generate_joke(state: JokeState):
    return {"jokes": [f"Joke about {state['subject']}"]}

graph.add_conditional_edges("collect_subjects", continue_to_jokes)
```

**Key points**:
- First arg: node name to send to
- Second arg: state to pass to that node
- Multiple `Send` objects = parallel execution

---

## 12. Compiling a Graph

### Why Compile?

- Performs basic structural checks (no orphaned nodes, etc.)
- Where you specify **checkpointers**, **breakpoints**, **cache**, **store**

```python
# Basic compile
graph = builder.compile()

# With checkpointer (persistence)
from langgraph.checkpoint.memory import InMemorySaver
graph = builder.compile(checkpointer=InMemorySaver())

# With breakpoints (debugging)
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["node_a"],
    interrupt_after=["node_b"]
)

# With store (long-term memory)
graph = builder.compile(
    checkpointer=checkpointer,
    store=my_store
)

# With cache
from langgraph.cache.memory import InMemoryCache
graph = builder.compile(cache=InMemoryCache())
```

> **Warning**: You MUST compile before using the graph.

---

## 13. Invoking & Running Graphs

### invoke (Synchronous)

```python
result = graph.invoke({"text": "hello"})
```

### ainvoke (Async)

```python
result = await graph.ainvoke({"text": "hello"})
```

### With Config (thread_id for persistence)

```python
config = {"configurable": {"thread_id": "thread-1"}}
result = graph.invoke({"text": "hello"}, config=config)
```

### With Context

```python
from dataclasses import dataclass

@dataclass
class Context:
    user_id: str

result = graph.invoke(
    {"text": "hello"},
    config=config,
    context=Context(user_id="user-1")
)
```

### v2 Invoke Format (LangGraph >= 1.1)

```python
from langgraph.types import GraphOutput

result = graph.invoke(inputs, version="v2")
# Returns GraphOutput with .value and .interrupts

result.value       # your output
result.interrupts  # tuple[Interrupt, ...], empty if none
```

---

## 14. Functional API (@entrypoint & @task)

### Overview

An alternative to the Graph API for adding LangGraph features (persistence, memory, human-in-the-loop, streaming) with **minimal code changes**.

### Two Building Blocks

| Component | Purpose |
|-----------|---------|
| `@entrypoint` | Starting point of a workflow; manages execution flow |
| `@task` | Discrete unit of work; runs asynchronously; results are checkpointed |

### Example

```python
import time
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import interrupt

@task
def write_essay(topic: str) -> str:
    """Write an essay about the given topic."""
    time.sleep(1)
    return f"An essay about topic: {topic}"

@entrypoint(checkpointer=InMemorySaver())
def workflow(topic: str) -> dict:
    """A simple workflow that writes an essay and asks for a review."""
    essay = write_essay("cat").result()
    is_approved = interrupt({
        "essay": essay,
        "action": "Please approve/reject the essay",
    })
    return {"essay": essay, "is_approved": is_approved}
```

### @entrypoint Details

```python
from langgraph.func import entrypoint

@entrypoint(checkpointer=checkpointer)
def my_workflow(some_input: dict) -> int:
    ...
    return result
```

**Injectable parameters** (auto-injected at runtime):

| Parameter | Purpose |
|-----------|---------|
| `previous` | State from previous checkpoint (short-term memory) |
| `store` | `BaseStore` instance (long-term memory) |
| `writer` | `StreamWriter` for async Python < 3.11 |
| `config` | `RunnableConfig` |

### Short-Term Memory with `previous`

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> int:
    previous = previous or 0
    return number + previous

config = {"configurable": {"thread_id": "1"}}
my_workflow.invoke(1, config)  # 1
my_workflow.invoke(2, config)  # 3 (previous was 1)
```

### entrypoint.final

Decouples return value from checkpoint value:

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> entrypoint.final[int, int]:
    previous = previous or 0
    return entrypoint.final(value=previous, save=2 * number)

config = {"configurable": {"thread_id": "1"}}
my_workflow.invoke(3, config)  # 0 (previous was None)
my_workflow.invoke(1, config)  # 6 (previous was 3 * 2)
```

### @task Details

```python
from langgraph.func import task

@task()
def slow_computation(input_value):
    # Long-running operation
    return result
```

**Tasks MUST be called from within** an entrypoint, another task, or a state graph node.

**Calling a task returns a future**:

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(some_input: int) -> int:
    future = slow_computation(some_input)
    return future.result()  # Wait synchronously
```

### When to Use Tasks

- **Checkpointing**: Save long-running results
- **Human-in-the-loop**: Wrap randomness (API calls) for correct resumption
- **Parallel execution**: Run I/O-bound tasks concurrently
- **Observability**: Track progress via LangSmith
- **Retryable work**: Handle failures with retry logic

### Serialization Rules

1. `@entrypoint` inputs and outputs must be **JSON-serializable**
2. `@task` outputs must be **JSON-serializable**

### Determinism

- Randomness MUST be inside tasks for correct resume behavior
- Tasks/subgraph results are persisted and replayed on resume

---

## 15. Persistence & Checkpointing

### What is Persistence?

LangGraph saves a **snapshot** (checkpoint) of graph state at every super-step boundary.

### Why Persistence?

| Feature | Why Needed |
|---------|-----------|
| **Human-in-the-loop** | Inspect, interrupt, approve graph steps |
| **Memory** | Retain conversation history between interactions |
| **Time travel** | Replay/debug prior graph executions |
| **Fault tolerance** | Resume from last successful step on failure |
| **Pending writes** | Failed node doesn't lose other nodes' results |

### Checkpointer Libraries

| Library | Class | Use Case |
|---------|-------|----------|
| `langgraph-checkpoint` | `InMemorySaver` | Development/testing (included with langgraph) |
| `langgraph-checkpoint-sqlite` | `SqliteSaver` / `AsyncSqliteSaver` | Local workflows |
| `langgraph-checkpoint-postgres` | `PostgresSaver` / `AsyncPostgresSaver` | **Production** |
| `langgraph-checkpoint-cosmosdb` | `CosmosDBSaver` / `AsyncCosmosDBSaver` | Azure production |

### Using a Checkpointer

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "1"}}
result = graph.invoke({"foo": "bar"}, config)
```

### Checkpoints & Super-Steps

For `START -> A -> B -> END`, checkpoints are created:
1. Empty checkpoint (next: `START`)
2. After user input (next: `node_a`)
3. After `node_a` (next: `node_b`)
4. After `node_b` (next: none — complete)

### Serializer

Default: `JsonPlusSerializer` (uses ormsgpack + JSON).

```python
# Pickle fallback for unsupported types (e.g., Pandas DataFrames)
from langgraph.checkpoint.serde.jsonplus import JsonPlusSerializer

graph.compile(
    checkpointer=InMemorySaver(serde=JsonPlusSerializer(pickle_fallback=True))
)
```

### Encryption

```python
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

serde = EncryptedSerializer.from_pycryptodome_aes()  # reads LANGGRAPH_AES_KEY env var
checkpointer = SqliteSaver(sqlite3.connect("checkpoint.db"), serde=serde)
```

---

## 16. Threads & State Management

### Threads

A **thread** is a unique identifier for a sequence of runs. State is persisted per-thread.

```python
config = {"configurable": {"thread_id": "my-conversation-1"}}
```

### Get State

```python
config = {"configurable": {"thread_id": "1"}}
snapshot = graph.get_state(config)
```

Returns a `StateSnapshot`:

| Field | Type | Description |
|-------|------|-------------|
| `values` | `dict` | State channel values at this checkpoint |
| `next` | `tuple[str, ...]` | Nodes to execute next (empty = complete) |
| `config` | `dict` | thread_id, checkpoint_ns, checkpoint_id |
| `metadata` | `dict` | source, writes, step |
| `created_at` | `str` | ISO 8601 timestamp |
| `parent_config` | `dict | None` | Config of previous checkpoint |
| `tasks` | `tuple[PregelTask, ...]` | Tasks to execute at this step |

### Get State History

```python
config = {"configurable": {"thread_id": "1"}}
history = list(graph.get_state_history(config))
# Returns list of StateSnapshot, most recent first
```

### Find Specific Checkpoints

```python
history = list(graph.get_state_history(config))

# Before node_b executed
before_b = next(s for s in history if s.next == ("node_b",))

# By step number
step_2 = next(s for s in history if s.metadata["step"] == 2)

# Where interrupt occurred
interrupted = next(
    s for s in history
    if s.tasks and any(t.interrupts for t in s.tasks)
)
```

### Update State

```python
# Creates a NEW checkpoint (does not modify original)
graph.update_state(config, values={"foo": "updated"})

# With as_node to control which node executes next
graph.update_state(config, values={"foo": "updated"}, as_node="node_a")
```

> Updates go through **reducers** — channels with reducers accumulate values.

---

## 17. Short-Term Memory (Conversation Memory)

Short-term memory = conversation history within a **thread**. Enabled via checkpointers.

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, MessagesState, START, END

def chatbot(state: MessagesState):
    # Use state["messages"] — contains full conversation history
    response = model.invoke(state["messages"])
    return {"messages": [response]}

builder = StateGraph(MessagesState)
builder.add_node("chatbot", chatbot)
builder.add_edge(START, "chatbot")
builder.add_edge("chatbot", END)

graph = builder.compile(checkpointer=InMemorySaver())

# First message
config = {"configurable": {"thread_id": "chat-1"}}
graph.invoke({"messages": [{"role": "user", "content": "Hi, I'm Bob"}]}, config)

# Follow-up (same thread_id = memory retained)
graph.invoke({"messages": [{"role": "user", "content": "What's my name?"}]}, config)
# AI remembers: "Your name is Bob"
```

---

## 18. Long-Term Memory (Memory Store)

Memory that persists **across threads** (e.g., user preferences across conversations).

### InMemoryStore — Basics

```python
from langgraph.store.memory import InMemoryStore
import uuid

store = InMemoryStore()

user_id = "1"
namespace = (user_id, "memories")

# Save a memory
memory_id = str(uuid.uuid4())
store.put(namespace, memory_id, {"food_preference": "I like pizza"})

# Retrieve memories
memories = store.search(namespace)
print(memories[-1].dict())
# {'value': {'food_preference': 'I like pizza'}, 'key': '...', 'namespace': ['1', 'memories'], ...}
```

### Memory Item Attributes

| Attribute | Description |
|-----------|-------------|
| `value` | The memory (dict) |
| `key` | Unique key within namespace |
| `namespace` | Tuple of strings |
| `created_at` | Timestamp |
| `updated_at` | Timestamp |

### Semantic Search

```python
from langchain.embeddings import init_embeddings

store = InMemoryStore(
    index={
        "embed": init_embeddings("openai:text-embedding-3-small"),
        "dims": 1536,
        "fields": ["food_preference", "$"]
    }
)

# Search by meaning
memories = store.search(
    namespace,
    query="What does the user like to eat?",
    limit=3
)
```

### Using Store in LangGraph

```python
from dataclasses import dataclass
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.runtime import Runtime

@dataclass
class Context:
    user_id: str

checkpointer = InMemorySaver()
builder = StateGraph(MessagesState, context_schema=Context)
# ... add nodes and edges ...
graph = builder.compile(checkpointer=checkpointer, store=store)

# Access store in nodes via Runtime
async def update_memory(state: MessagesState, runtime: Runtime[Context]):
    user_id = runtime.context.user_id
    namespace = (user_id, "memories")
    memory_id = str(uuid.uuid4())
    await runtime.store.aput(namespace, memory_id, {"memory": "some info"})

async def call_model(state: MessagesState, runtime: Runtime[Context]):
    user_id = runtime.context.user_id
    namespace = (user_id, "memories")
    memories = await runtime.store.asearch(
        namespace,
        query=state["messages"][-1].content,
        limit=3
    )
    info = "\n".join([d.value["memory"] for d in memories])
    # Use memories in model call...

# Invoke with context
config = {"configurable": {"thread_id": "1"}}
graph.invoke(
    {"messages": [{"role": "user", "content": "hi"}]},
    config,
    context=Context(user_id="1")
)
```

> `InMemoryStore` is for dev/testing. Use `PostgresStore` or `RedisStore` in production.

---

## 19. Streaming

### Overview

LangGraph streams real-time updates via `stream()` (sync) and `astream()` (async).

### Basic Usage

```python
for chunk in graph.stream(
    {"topic": "ice cream"},
    stream_mode=["updates", "custom"],
    version="v2",
):
    if chunk["type"] == "updates":
        for node_name, state in chunk["data"].items():
            print(f"Node {node_name}: {state}")
    elif chunk["type"] == "custom":
        print(f"Custom: {chunk['data']}")
```

### v2 Format (LangGraph >= 1.1)

All chunks are `StreamPart` dicts with a consistent shape:

```python
{
    "type": "values" | "updates" | "messages" | "custom" | "checkpoints" | "tasks" | "debug",
    "ns": (),      # namespace tuple, populated for subgraph events
    "data": ...,   # the actual payload
}
```

### Stream Modes

| Mode | Description |
|------|-------------|
| `values` | Full state after each step |
| `updates` | Only changed keys from each node |
| `messages` | (LLM token, metadata) from LLM calls |
| `custom` | Custom data emitted via `get_stream_writer` |
| `checkpoints` | Checkpoint events (requires checkpointer) |
| `tasks` | Task start/finish events (requires checkpointer) |
| `debug` | All info — combines checkpoints + tasks + metadata |

### Streaming State Updates

```python
for chunk in graph.stream({"topic": "ice cream"}, stream_mode="updates", version="v2"):
    if chunk["type"] == "updates":
        for node_name, state in chunk["data"].items():
            print(f"Node `{node_name}`: {state}")
```

### Streaming LLM Tokens

```python
for chunk in graph.stream({"topic": "ice cream"}, stream_mode="messages", version="v2"):
    if chunk["type"] == "messages":
        message_chunk, metadata = chunk["data"]
        if message_chunk.content:
            print(message_chunk.content, end="", flush=True)
```

**Filter by node**:

```python
if metadata["langgraph_node"] == "some_node_name":
    print(msg.content)
```

**Filter by tag**:

```python
model_1 = init_chat_model("gpt-4.1-mini", tags=['joke'])
# Then filter: if metadata["tags"] == ["joke"]:
```

**Omit from stream**: Use `nostream` tag on the model.

### Custom Data Streaming

```python
from langgraph.config import get_stream_writer

def my_node(state: State):
    writer = get_stream_writer()
    writer({"progress": "50%"})
    # ... processing ...
    writer({"progress": "100%"})
    return {"answer": "done"}

# Consume:
for chunk in graph.stream(inputs, stream_mode="custom", version="v2"):
    if chunk["type"] == "custom":
        print(chunk["data"])
```

### Streaming from Subgraphs

```python
for chunk in graph.stream(
    {"foo": "foo"},
    subgraphs=True,
    stream_mode="updates",
    version="v2",
):
    print(chunk["ns"])  # () for root, ("node_name:<task_id>",) for subgraph
```

### Multiple Modes at Once

```python
for chunk in graph.stream(inputs, stream_mode=["updates", "custom"], version="v2"):
    if chunk["type"] == "updates":
        ...
    elif chunk["type"] == "custom":
        ...
```

---

## 20. Interrupts & Human-in-the-Loop

### Overview

Interrupts pause graph execution at specific points and wait for external input.

### Requirements

1. A **checkpointer** for persistence
2. A **thread_id** in config
3. Call `interrupt()` where you want to pause (payload must be JSON-serializable)

### Pause with `interrupt()`

```python
from langgraph.types import interrupt

def approval_node(state: State):
    approved = interrupt("Do you approve this action?")
    return {"approved": approved}
```

**What happens**:
1. Execution suspends at the `interrupt()` call
2. State is saved via checkpointer
3. Value is returned to caller (under `__interrupt__`)
4. Graph waits indefinitely until resumed
5. Resume value becomes the return value of `interrupt()`

### Resume with `Command(resume=...)`

```python
from langgraph.types import Command

config = {"configurable": {"thread_id": "thread-1"}}

# Initial run — hits interrupt
result = graph.invoke({"input": "data"}, config=config)
print(result.interrupts)  # (Interrupt(value='Do you approve?'),)

# Resume with user's response
graph.invoke(Command(resume=True), config=config)
```

### Common Patterns

#### Approve or Reject

```python
from typing import Literal
from langgraph.types import interrupt, Command

def approval_node(state: State) -> Command[Literal["proceed", "cancel"]]:
    is_approved = interrupt({
        "question": "Do you want to proceed?",
        "details": state["action_details"]
    })
    if is_approved:
        return Command(goto="proceed")
    else:
        return Command(goto="cancel")

# Resume: graph.invoke(Command(resume=True), config)   # approve
# Resume: graph.invoke(Command(resume=False), config)  # reject
```

#### Review and Edit State

```python
def review_node(state: State):
    edited = interrupt({
        "instruction": "Review and edit this content",
        "content": state["generated_text"]
    })
    return {"generated_text": edited}

# Resume: graph.invoke(Command(resume="The edited text"), config)
```

#### Interrupt in Tools

```python
@tool
def send_email(to: str, subject: str, body: str):
    """Send an email to a recipient."""
    response = interrupt({
        "action": "send_email",
        "to": to, "subject": subject, "body": body,
        "message": "Approve sending this email?"
    })
    if response.get("action") == "approve":
        return f"Email sent to {to}"
    return "Email cancelled"
```

#### Validating Human Input (Loop)

```python
def get_age_node(state: State):
    prompt = "What is your age?"
    while True:
        answer = interrupt(prompt)
        if isinstance(answer, int) and answer > 0:
            break
        prompt = f"'{answer}' is not a valid age. Enter a positive number."
    return {"age": answer}
```

#### Handling Multiple Parallel Interrupts

```python
# When parallel branches interrupt simultaneously
interrupted_result = graph.invoke({"vals": []}, config)
# interrupted_result["__interrupt__"] contains multiple Interrupts

resume_map = {
    i.id: f"answer for {i.value}"
    for i in interrupted_result["__interrupt__"]
}
result = graph.invoke(Command(resume=resume_map), config)
```

### Rules of Interrupts

| Rule | Why |
|------|-----|
| **Do NOT wrap `interrupt` in try/except** | `interrupt` works by throwing a special exception |
| **Do NOT reorder interrupts within a node** | Matching is strictly index-based |
| **Use JSON-serializable payloads only** | Complex objects (functions, classes) cannot be serialized |
| **Side effects before `interrupt` must be idempotent** | Node restarts from the beginning on resume |

### Static Interrupts (Debugging)

```python
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["node_a"],     # Pause before node_a
    interrupt_after=["node_b"],      # Pause after node_b
)

graph.invoke(inputs, config)    # Runs until breakpoint
graph.invoke(None, config)      # Resume from breakpoint
```

> Static interrupts are for debugging only. Use `interrupt()` for production human-in-the-loop.

---

## 21. Durable Execution

### What is Durable Execution?

A technique where a workflow saves progress at key points, allowing it to **pause and resume exactly where it left off** — even after failures.

### Requirements

1. Enable **persistence** via checkpointer
2. Specify a **thread_id**
3. Wrap non-deterministic operations in **tasks** or **nodes**

### Determinism & Consistent Replay

When resuming, code does NOT resume from the same line — it replays from an appropriate starting point. You MUST:

- **Avoid repeating work**: Wrap each side-effect in a separate task
- **Encapsulate non-deterministic ops**: Random numbers, time, API calls → inside tasks
- **Use idempotent operations**: Database writes should use upsert, not insert

### Durability Modes

```python
graph.stream({"input": "test"}, durability="sync")
```

| Mode | Behavior | Performance | Durability |
|------|----------|-------------|------------|
| `"exit"` | Persist only on graph exit | Best | Lowest |
| `"async"` | Persist asynchronously while next step runs | Good | Medium |
| `"sync"` | Persist synchronously before next step | Overhead | Highest |

### Using Tasks in Nodes (Graph API)

```python
from langgraph.func import task

@task
def call_api(url: str) -> str:
    return requests.get(url).text[:100]

def my_node(state: State):
    result = call_api(state["url"]).result()
    return {"result": result}
```

### Resuming Workflows

- **After interrupt**: `graph.invoke(Command(resume=value), config)`
- **After error**: `graph.invoke(None, config)` (same thread_id, assumes error resolved)

### Starting Points for Resume

- **StateGraph**: Beginning of the node where execution stopped
- **Subgraphs**: Parent node that called the subgraph; within subgraph → the specific node
- **Functional API**: Beginning of the `@entrypoint`

---

## 22. Subgraphs

### What are Subgraphs?

A subgraph is a graph used as a **node** in another graph. Useful for:
- **Multi-agent systems**
- **Reusing** node sets across graphs
- **Distributed development** (separate teams own separate subgraphs)

### Two Communication Patterns

| Pattern | When to Use | How |
|---------|-------------|-----|
| **Call inside a node** | Different state schemas (no shared keys) | Write wrapper function to transform state |
| **Add as a node** | Shared state keys | Pass compiled subgraph directly to `add_node` |

### Pattern 1: Call Inside a Node (Different Schemas)

```python
class SubgraphState(TypedDict):
    bar: str

def subgraph_node_1(state: SubgraphState):
    return {"bar": "hi! " + state["bar"]}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# Parent graph
class State(TypedDict):
    foo: str

def call_subgraph(state: State):
    subgraph_output = subgraph.invoke({"bar": state["foo"]})
    return {"foo": subgraph_output["bar"]}

builder = StateGraph(State)
builder.add_node("node_1", call_subgraph)
builder.add_edge(START, "node_1")
graph = builder.compile()
```

### Pattern 2: Add as a Node (Shared Schema)

```python
class State(TypedDict):
    foo: str

# Subgraph
def subgraph_node_1(state: State):
    return {"foo": "hi! " + state["foo"]}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# Parent graph — pass compiled subgraph directly
builder = StateGraph(State)
builder.add_node("node_1", subgraph)  # <-- Direct!
builder.add_edge(START, "node_1")
graph = builder.compile()
```

### Subgraph Persistence Modes

| Mode | `checkpointer=` | Behavior |
|------|-----------------|----------|
| **Per-invocation** (default) | `None` | Each call starts fresh; inherits parent's checkpointer |
| **Per-thread** | `True` | State accumulates across calls on same thread |
| **Stateless** | `False` | No checkpointing — like a plain function |

```python
subgraph = subgraph_builder.compile(checkpointer=None)   # Per-invocation
subgraph = subgraph_builder.compile(checkpointer=True)    # Per-thread
subgraph = subgraph_builder.compile(checkpointer=False)   # Stateless
```

### Capabilities by Mode

| Feature | `None` (per-invocation) | `True` (per-thread) | `False` (stateless) |
|---------|------------------------|--------------------|--------------------|
| Interrupts (HITL) | ✅ | ✅ | ❌ |
| Multi-turn memory | ❌ | ✅ | ❌ |
| Parallel calls (same subgraph) | ✅ | ❌ | ✅ |
| State inspection | Current invocation only | ✅ | ❌ |

### Viewing Subgraph State

```python
state = graph.get_state(config, subgraphs=True)
subgraph_state = state.tasks[0].state
```

### Streaming from Subgraphs

```python
for chunk in graph.stream(inputs, subgraphs=True, stream_mode="updates", version="v2"):
    print(chunk["ns"])  # () = root, ("node_name:<task_id>",) = subgraph
```

---

## 23. Runtime Context & Configuration

### Context Schema

Pass runtime information to nodes that is NOT part of graph state:

```python
from dataclasses import dataclass
from langgraph.graph import StateGraph
from langgraph.runtime import Runtime

@dataclass
class ContextSchema:
    llm_provider: str = "openai"

graph = StateGraph(State, context_schema=ContextSchema)

# Pass context at invocation
graph.invoke(inputs, context={"llm_provider": "anthropic"})

# Access in nodes
def node_a(state: State, runtime: Runtime[ContextSchema]):
    llm = get_llm(runtime.context.llm_provider)
```

### RunnableConfig

Access thread_id, tags, and metadata:

```python
from langchain_core.runnables import RunnableConfig

def my_node(state: State, config: RunnableConfig):
    thread_id = config["configurable"]["thread_id"]
    checkpoint_ns = config["configurable"]["checkpoint_ns"]
```

### Available Metadata

```python
def inspect_metadata(state: dict, config: RunnableConfig):
    metadata = config["metadata"]
    print(f"Step: {metadata['langgraph_step']}")
    print(f"Node: {metadata['langgraph_node']}")
    print(f"Triggers: {metadata['langgraph_triggers']}")
    print(f"Path: {metadata['langgraph_path']}")
    print(f"Checkpoint NS: {metadata['langgraph_checkpoint_ns']}")
```

---

## 24. Recursion Limit & Step Counter

### Default Recursion Limit

Default: **1000 super-steps** (since v1.0.6).

### Setting Recursion Limit

```python
# Set via config (NOT inside configurable!)
graph.invoke(inputs, config={"recursion_limit": 5})
```

### Proactive Handling with `RemainingSteps`

```python
from typing import Annotated
from langgraph.managed import RemainingSteps
from typing_extensions import TypedDict

class State(TypedDict):
    messages: Annotated[list, lambda x, y: x + y]
    remaining_steps: RemainingSteps  # Auto-populated by LangGraph

def reasoning_node(state: State):
    remaining = state["remaining_steps"]
    if remaining <= 2:
        return {"messages": ["Wrapping up..."]}
    return {"messages": ["thinking..."]}
```

### Reactive Handling (Catch Error)

```python
from langgraph.errors import GraphRecursionError

try:
    result = graph.invoke(inputs, config={"recursion_limit": 10})
except GraphRecursionError:
    result = {"messages": ["Recursion limit exceeded"]}
```

### Proactive vs Reactive

| Approach | When | Where | Outcome |
|----------|------|-------|---------|
| **Proactive** (RemainingSteps) | Before limit | Inside graph | Graceful degradation |
| **Reactive** (try/catch) | After limit | Outside graph | Exception handling |

---

## 25. Node Caching

### What is Node Caching?

Cache node results based on input to avoid re-computation.

```python
from langgraph.graph import StateGraph
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy
from typing_extensions import TypedDict

class State(TypedDict):
    x: int
    result: int

def expensive_node(state: State) -> dict:
    import time
    time.sleep(2)  # expensive computation
    return {"result": state["x"] * 2}

builder = StateGraph(State)
builder.add_node(
    "expensive_node",
    expensive_node,
    cache_policy=CachePolicy(ttl=3)  # Cache for 3 seconds
)
builder.set_entry_point("expensive_node")
builder.set_finish_point("expensive_node")

graph = builder.compile(cache=InMemoryCache())

# First run: 2 seconds
print(graph.invoke({"x": 5}, stream_mode='updates'))
# [{'expensive_node': {'result': 10}}]

# Second run: instant (cached)
print(graph.invoke({"x": 5}, stream_mode='updates'))
# [{'expensive_node': {'result': 10}, '__metadata__': {'cached': True}}]
```

### CachePolicy Options

| Parameter | Description |
|-----------|-------------|
| `key_func` | Function to generate cache key from node input (default: hash via pickle) |
| `ttl` | Time to live in seconds (None = never expire) |

---

## 26. Graph Migrations

### Supported Changes

| Scenario | Thread State | Support |
|----------|-------------|---------|
| Finished threads | All topology changes (add/remove/rename nodes, edges) | ✅ |
| Interrupted threads | All changes except renaming/removing current node | ⚠️ |
| State keys — add/remove | Full compatibility | ✅ |
| State keys — rename | Loses saved state for renamed key | ⚠️ |
| State keys — type change (incompatible) | May cause issues in existing threads | ⚠️ |

---

## 27. Visualization

LangGraph has built-in graph visualization:

```python
from IPython.display import Image, display

# Get Mermaid PNG
display(Image(graph.get_graph(xray=True).draw_mermaid_png()))

# Get Mermaid diagram as string
print(graph.get_graph().draw_mermaid())
```

> **Note**: The Functional API does NOT support visualization (graph is dynamic at runtime).

---

## 28. LangSmith Integration (Tracing & Observability)

### Enable Tracing

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=your_key
```

### What LangSmith Provides

- Full execution traces of agent runs
- State transitions at each step
- LLM call details (inputs, outputs, tokens)
- Debugging for complex multi-step agents
- Evaluation of agent performance

---

## 29. Deployment & Production

### Checkpointer Selection

| Stage | Checkpointer |
|-------|-------------|
| Development | `InMemorySaver` |
| Testing | `SqliteSaver` |
| Production | `PostgresSaver` or `CosmosDBSaver` |

### Store Selection

| Stage | Store |
|-------|-------|
| Development | `InMemoryStore` |
| Production | `PostgresStore` or `RedisStore` |

### LangGraph Server (Agent Server)

For production deployment, use the LangGraph Server / LangSmith platform:
- Handles checkpoint infrastructure automatically
- Handles stores automatically
- Provides scalable infrastructure for stateful, long-running workflows
- Supports visual prototyping via **LangGraph Studio**

### Configuration for Deployment

```json
{
    "store": {
        "index": {
            "embed": "openai:text-embeddings-3-small",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

---

## 30. Common Patterns & Best Practices

### ReAct Agent (Tool-Calling Loop)

```python
from langchain.chat_models import init_chat_model
from langchain.tools import tool
from langchain.messages import AnyMessage, SystemMessage, HumanMessage, ToolMessage
from langgraph.graph import StateGraph, MessagesState, START, END
from typing import Annotated, Literal

model = init_chat_model("claude-sonnet-4-6", temperature=0)

@tool
def multiply(a: int, b: int) -> int:
    """Multiply a and b."""
    return a * b

@tool
def add(a: int, b: int) -> int:
    """Add a and b."""
    return a + b

tools = [add, multiply]
tools_by_name = {t.name: t for t in tools}
model_with_tools = model.bind_tools(tools)

class State(MessagesState):
    llm_calls: int

def llm_call(state: State):
    return {
        "messages": [
            model_with_tools.invoke(
                [SystemMessage(content="You are a helpful assistant.")]
                + state["messages"]
            )
        ],
        "llm_calls": state.get("llm_calls", 0) + 1
    }

def tool_node(state: State):
    results = []
    for tc in state["messages"][-1].tool_calls:
        tool = tools_by_name[tc["name"]]
        result = tool.invoke(tc["args"])
        results.append(ToolMessage(content=result, tool_call_id=tc["id"]))
    return {"messages": results}

def should_continue(state: State) -> Literal["tool_node", "__end__"]:
    if state["messages"][-1].tool_calls:
        return "tool_node"
    return END

builder = StateGraph(State)
builder.add_node("llm_call", llm_call)
builder.add_node("tool_node", tool_node)
builder.add_edge(START, "llm_call")
builder.add_conditional_edges("llm_call", should_continue, ["tool_node", END])
builder.add_edge("tool_node", "llm_call")

agent = builder.compile()
```

### Chatbot with Memory

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "conv-1"}}
graph.invoke({"messages": [HumanMessage(content="Hi!")]}, config)
graph.invoke({"messages": [HumanMessage(content="Follow up")]}, config)
```

### Best Practices

1. **Always use checkpointers** for production workflows
2. **Wrap side effects in tasks** for durable execution
3. **Use `add_messages` reducer** for message lists
4. **Use `MessagesState`** as a convenient base state
5. **Favor `interrupt()` over static breakpoints** for human-in-the-loop
6. **Keep interrupt payloads JSON-serializable**
7. **Ensure idempotency** for operations before interrupts
8. **Use `RemainingSteps`** to handle recursion limits gracefully
9. **Use v2 streaming format** for consistent output structure
10. **Use subgraphs** for modular, reusable agent components

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Non-idempotent side effects before `interrupt` | Move side effects after interrupt, or use upsert |
| Wrapping `interrupt` in try/except | Separate interrupt from error-prone code |
| Non-deterministic control flow | Wrap randomness in tasks |
| Using `Command(update=...)` as input | Use plain dict for multi-turn input |
| Forgetting to compile before use | Always call `.compile()` |
| Missing thread_id with checkpointer | Always provide `{"configurable": {"thread_id": "..."}}` |

---

## 31. Choosing Graph API vs Functional API

| Aspect | Graph API | Functional API |
|--------|-----------|----------------|
| **Control flow** | Explicit nodes + edges | Standard Python (`if`, `for`, functions) |
| **State management** | Declare `State` + reducers | Scoped to function (no shared state) |
| **Checkpointing** | New checkpoint per super-step | Results saved to existing checkpoint |
| **Visualization** | ✅ Built-in graph rendering | ❌ Dynamic — no visualization |
| **Code style** | Declarative (graph paradigm) | Imperative (write regular code) |
| **Amount of code** | More boilerplate | Less code typically |

### When to Use Graph API

- You want visual debugging of your workflow
- You need explicit parallel execution control
- Your team prefers declarative workflow definitions
- You're building complex multi-agent architectures

### When to Use Functional API

- You want minimal changes to existing code
- You prefer standard Python control flow
- You're building simpler, more linear workflows
- Visualization is not important

### They Can Be Mixed

Both APIs share the same underlying runtime and can be used together in the same application. Tasks can be called from within Graph API nodes.

---

## Quick Reference — Key Imports

```python
# Graph building
from langgraph.graph import StateGraph, START, END, MessagesState

# Message handling
from langgraph.graph.message import add_messages

# Types & primitives
from langgraph.types import Command, Send, interrupt, CachePolicy
from langgraph.types import Overwrite, RemainingSteps  # managed values

# Functional API
from langgraph.func import entrypoint, task

# Persistence
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore

# Caching
from langgraph.cache.memory import InMemoryCache

# Runtime
from langgraph.runtime import Runtime

# Config/streaming
from langgraph.config import get_stream_writer

# Errors
from langgraph.errors import GraphRecursionError

# Managed values
from langgraph.managed import RemainingSteps

# LangChain (commonly used with LangGraph)
from langchain.chat_models import init_chat_model
from langchain.tools import tool
from langchain.messages import HumanMessage, AIMessage, SystemMessage, ToolMessage, AnyMessage
from langchain_core.runnables import RunnableConfig
```

---

*Notes created April 2026 — based on LangGraph v1.1.6 stable release.*
