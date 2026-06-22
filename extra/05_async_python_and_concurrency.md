# Async Programming in Python & How Python Handles Concurrent API Requests

---

## Table of Contents

1. [Fundamentals: Sync vs Async](#1-fundamentals-sync-vs-async)
2. [Python's GIL (Global Interpreter Lock)](#2-pythons-gil)
3. [Coroutines & The Event Loop](#3-coroutines--the-event-loop)
4. [async/await Syntax Deep Dive](#4-asyncawait-syntax-deep-dive)
5. [asyncio Module Complete Guide](#5-asyncio-module-complete-guide)
6. [Running Tasks Concurrently](#6-running-tasks-concurrently)
7. [Async Generators & Iterators](#7-async-generators--iterators)
8. [Async Context Managers](#8-async-context-managers)
9. [Synchronization Primitives](#9-synchronization-primitives)
10. [Async Queues](#10-async-queues)
11. [Async HTTP with aiohttp](#11-async-http-with-aiohttp)
12. [Async Database Access](#12-async-database-access)
13. [Threading vs Multiprocessing vs Asyncio](#13-threading-vs-multiprocessing-vs-asyncio)
14. [How Python Web Frameworks Handle Multiple Users](#14-how-python-web-frameworks-handle-multiple-users)
15. [Scenario 1: Flask (Sync WSGI)](#15-scenario-1-flask-sync-wsgi)
16. [Scenario 2: Django (Sync WSGI + Async Support)](#16-scenario-2-django-sync-wsgi--async-support)
17. [Scenario 3: FastAPI (Async ASGI)](#17-scenario-3-fastapi-async-asgi)
18. [Scenario 4: What Happens When 1000 Users Hit the Same API?](#18-scenario-4-1000-users-hit-the-same-api)
19. [Scenario 5: CPU-Bound vs I/O-Bound API Handlers](#19-scenario-5-cpu-bound-vs-io-bound-api-handlers)
20. [Scenario 6: Blocking Code Inside Async Handler](#20-scenario-6-blocking-code-inside-async-handler)
21. [Scenario 7: Database Connection Pooling](#21-scenario-7-database-connection-pooling)
22. [Scenario 8: Real-World Production Setup](#22-scenario-8-real-world-production-setup)
23. [Common Pitfalls & Best Practices](#23-common-pitfalls--best-practices)

---

## 1. Fundamentals: Sync vs Async

### Synchronous Execution (Default Python)

```python
import time

def task_1():
    print("Task 1 started")
    time.sleep(2)  # Blocks the entire thread for 2 seconds
    print("Task 1 finished")

def task_2():
    print("Task 2 started")
    time.sleep(2)
    print("Task 2 finished")

# Total time: ~4 seconds (sequential)
task_1()
task_2()
```

**Output:**
```
Task 1 started
(waits 2 sec)
Task 1 finished
Task 2 started
(waits 2 sec)
Task 2 finished
```

### Asynchronous Execution

```python
import asyncio

async def task_1():
    print("Task 1 started")
    await asyncio.sleep(2)  # Yields control back to event loop
    print("Task 1 finished")

async def task_2():
    print("Task 2 started")
    await asyncio.sleep(2)
    print("Task 2 finished")

async def main():
    # Total time: ~2 seconds (concurrent)
    await asyncio.gather(task_1(), task_2())

asyncio.run(main())
```

**Output:**
```
Task 1 started
Task 2 started
(waits 2 sec — both waiting simultaneously)
Task 1 finished
Task 2 finished
```

### Key Difference Visual

```
SYNCHRONOUS:
Thread: |---Task1---|---Task2---|  (Total: 4 sec)

ASYNCHRONOUS:
Thread: |---Task1---|              (Total: 2 sec)
        |---Task2---|
        (both run concurrently on SAME thread)
```

---

## 2. Python's GIL

### What is the GIL?

The **Global Interpreter Lock (GIL)** is a mutex in CPython that allows only **one thread to execute Python bytecode at a time**, even on multi-core CPUs.

```
CPU Core 1: [Thread 1 runs Python] [Thread 2 runs Python] [Thread 1 runs Python]
CPU Core 2: [IDLE]                 [IDLE]                  [IDLE]
CPU Core 3: [IDLE]                 [IDLE]                  [IDLE]
CPU Core 4: [IDLE]                 [IDLE]                  [IDLE]

^ Even with 4 cores, only 1 thread runs Python at a time!
```

### Why GIL Exists

- Simplifies CPython's memory management (reference counting)
- Makes C extensions easier to write
- Single-threaded programs run faster with GIL

### When GIL is Released

The GIL is released during **I/O operations**:

```python
import threading
import time

# GIL is released during time.sleep(), file I/O, network I/O
# So these CAN run in parallel:
def download_file(url):
    # GIL released during network I/O
    response = requests.get(url)  # Other threads can run here
    return response

# GIL is NOT released during CPU computation
# So these CANNOT run in parallel:
def compute_heavy(n):
    # GIL held — only this thread runs
    return sum(i * i for i in range(n))
```

### GIL Impact Summary

| Operation Type | Threading Helps? | Asyncio Helps? | Multiprocessing Helps? |
|---------------|-----------------|----------------|----------------------|
| I/O Bound (network, file, DB) | ✅ Yes | ✅ Yes | ✅ Yes (overkill) |
| CPU Bound (computation) | ❌ No (GIL) | ❌ No | ✅ Yes |
| Mixed | ⚠️ Partial | ⚠️ Partial | ✅ Yes |

---

## 3. Coroutines & The Event Loop

### What is a Coroutine?

A coroutine is a function that can **pause** and **resume** its execution. Defined with `async def`.

```python
import asyncio

# This is a coroutine FUNCTION
async def greet(name):
    print(f"Hello, {name}")
    await asyncio.sleep(1)      # Pause here, let others run
    print(f"Goodbye, {name}")

# Calling it returns a coroutine OBJECT (doesn't execute yet!)
coro = greet("Alice")
print(type(coro))  # <class 'coroutine'>

# To actually run it, you need an event loop:
asyncio.run(coro)
```

### What is the Event Loop?

The event loop is the **central scheduler** that manages and runs coroutines.

```
┌──────────────────────────────────────────┐
│              EVENT LOOP                   │
│                                          │
│  1. Pick a ready coroutine               │
│  2. Run it until it hits 'await'         │
│  3. Register what it's waiting for       │
│  4. Pick next ready coroutine            │
│  5. Repeat                               │
│                                          │
│  Ready Queue: [coro_A, coro_C]           │
│  Waiting:     [coro_B → network I/O]     │
│               [coro_D → sleep(2)]        │
└──────────────────────────────────────────┘
```

### Event Loop Lifecycle

```python
import asyncio

async def worker(name, delay):
    print(f"{name}: starting")
    await asyncio.sleep(delay)
    print(f"{name}: done")
    return f"{name} result"

async def main():
    # The event loop is already running here (started by asyncio.run)
    
    # Get reference to current event loop
    loop = asyncio.get_running_loop()
    print(f"Loop running: {loop.is_running()}")  # True
    
    results = await asyncio.gather(
        worker("A", 2),
        worker("B", 1),
        worker("C", 3),
    )
    print(results)  # ['A result', 'B result', 'C result']

# This creates the event loop, runs main(), then closes the loop
asyncio.run(main())
```

### How the Event Loop Schedules Coroutines Step by Step

```python
import asyncio

async def fetch_data(id, delay):
    print(f"[{id}] Start fetching")
    await asyncio.sleep(delay)        # Yields control to event loop
    print(f"[{id}] Done fetching")
    return f"Data-{id}"

async def main():
    task1 = asyncio.create_task(fetch_data(1, 2))
    task2 = asyncio.create_task(fetch_data(2, 1))
    task3 = asyncio.create_task(fetch_data(3, 3))
    
    # Event loop execution order:
    # t=0.0: task1 prints "Start", hits await sleep(2), PAUSED
    # t=0.0: task2 prints "Start", hits await sleep(1), PAUSED
    # t=0.0: task3 prints "Start", hits await sleep(3), PAUSED
    # t=1.0: task2 wakes up, prints "Done" → COMPLETED
    # t=2.0: task1 wakes up, prints "Done" → COMPLETED
    # t=3.0: task3 wakes up, prints "Done" → COMPLETED
    
    results = await asyncio.gather(task1, task2, task3)
    print(results)

asyncio.run(main())
```

---

## 4. async/await Syntax Deep Dive

### Rules of async/await

```python
import asyncio

# Rule 1: 'await' can ONLY be used inside 'async def'
async def valid():
    await asyncio.sleep(1)  # ✅ OK

def invalid():
    # await asyncio.sleep(1)  # ❌ SyntaxError!
    pass

# Rule 2: You can ONLY 'await' an awaitable (coroutine, Task, Future)
async def example():
    await asyncio.sleep(1)                # ✅ awaiting a coroutine
    task = asyncio.create_task(valid())
    await task                             # ✅ awaiting a Task
    # await 42                             # ❌ TypeError: int is not awaitable

# Rule 3: Calling async function returns coroutine, doesn't run it
async def greet():
    return "hello"

async def main():
    # WRONG: this doesn't run greet, just creates a coroutine object
    result = greet()        # ⚠️ RuntimeWarning: coroutine was never awaited
    
    # RIGHT: await it to run and get the result
    result = await greet()  # ✅ returns "hello"
    print(result)

asyncio.run(main())
```

### Awaitable Objects

```python
import asyncio

# 1. Coroutines (most common)
async def my_coroutine():
    return 42

# 2. Tasks (scheduled coroutines)
async def main():
    task = asyncio.create_task(my_coroutine())
    result = await task

# 3. Futures (low-level, rarely used directly)
async def main():
    loop = asyncio.get_running_loop()
    future = loop.create_future()
    future.set_result(42)
    result = await future

# 4. Objects with __await__ method (custom awaitables)
class MyAwaitable:
    def __await__(self):
        yield  # pause once
        return 42

async def main():
    result = await MyAwaitable()
    print(result)  # 42

asyncio.run(main())
```

### Async Return Values

```python
import asyncio

async def fetch_user(user_id: int) -> dict:
    await asyncio.sleep(0.5)  # simulate DB query
    return {"id": user_id, "name": f"User_{user_id}"}

async def fetch_orders(user_id: int) -> list:
    await asyncio.sleep(0.3)
    return [{"order_id": 1, "amount": 99.99}]

async def main():
    # Sequential (slow) — total ~0.8 sec
    user = await fetch_user(1)
    orders = await fetch_orders(1)
    
    # Concurrent (fast) — total ~0.5 sec
    user, orders = await asyncio.gather(
        fetch_user(1),
        fetch_orders(1)
    )
    
    print(user, orders)

asyncio.run(main())
```

---

## 5. asyncio Module Complete Guide

### Starting the Event Loop

```python
import asyncio

# Method 1: asyncio.run() — the standard way (Python 3.7+)
async def main():
    print("Hello")

asyncio.run(main())  # Creates loop, runs, closes

# Method 2: Low-level (rarely needed)
loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)
try:
    loop.run_until_complete(main())
finally:
    loop.close()
```

### asyncio.sleep()

```python
import asyncio

async def main():
    print("Start")
    await asyncio.sleep(2)      # Non-blocking sleep — event loop does other things
    print("After 2 seconds")
    
    # NEVER use time.sleep() in async code!
    # time.sleep(2)  # ❌ This BLOCKS the entire event loop!

asyncio.run(main())
```

### asyncio.create_task()

```python
import asyncio

async def background_job(name, seconds):
    print(f"{name}: started")
    await asyncio.sleep(seconds)
    print(f"{name}: finished")
    return f"{name} done"

async def main():
    # create_task schedules the coroutine to run concurrently
    task1 = asyncio.create_task(background_job("Job-A", 3))
    task2 = asyncio.create_task(background_job("Job-B", 1))
    
    # Tasks start running immediately after create_task
    # They don't wait for 'await'
    
    print("Tasks created, doing other work...")
    await asyncio.sleep(0.5)
    print("Other work done")
    
    # Now wait for both tasks to complete
    result1 = await task1
    result2 = await task2
    print(result1, result2)

asyncio.run(main())
```

### asyncio.gather()

```python
import asyncio

async def api_call(endpoint, delay):
    await asyncio.sleep(delay)
    return f"Response from {endpoint}"

async def main():
    # gather runs all coroutines concurrently and returns results IN ORDER
    results = await asyncio.gather(
        api_call("/users", 2),
        api_call("/orders", 1),
        api_call("/products", 3),
    )
    print(results)
    # ['Response from /users', 'Response from /orders', 'Response from /products']
    # Total time: ~3 seconds (not 6!)

    # With return_exceptions=True, exceptions don't cancel other tasks
    results = await asyncio.gather(
        api_call("/users", 1),
        asyncio.sleep(0),  # this will return None
        return_exceptions=True
    )

asyncio.run(main())
```

### asyncio.wait()

```python
import asyncio

async def task(name, delay):
    await asyncio.sleep(delay)
    if name == "bad":
        raise ValueError("Something went wrong")
    return f"{name} result"

async def main():
    tasks = [
        asyncio.create_task(task("A", 1)),
        asyncio.create_task(task("B", 2)),
        asyncio.create_task(task("C", 3)),
    ]
    
    # Wait for ALL tasks
    done, pending = await asyncio.wait(tasks, return_when=asyncio.ALL_COMPLETED)
    for t in done:
        print(t.result())
    
    # Wait for FIRST completed
    tasks2 = [
        asyncio.create_task(task("X", 3)),
        asyncio.create_task(task("Y", 1)),
    ]
    done, pending = await asyncio.wait(tasks2, return_when=asyncio.FIRST_COMPLETED)
    print(f"First done: {done.pop().result()}")  # Y result
    
    # Cancel remaining
    for t in pending:
        t.cancel()

asyncio.run(main())
```

### asyncio.wait_for() — Timeout

```python
import asyncio

async def slow_operation():
    await asyncio.sleep(10)
    return "Done"

async def main():
    try:
        result = await asyncio.wait_for(slow_operation(), timeout=3.0)
    except asyncio.TimeoutError:
        print("Operation timed out after 3 seconds!")

asyncio.run(main())
```

### asyncio.as_completed()

```python
import asyncio

async def fetch(url, delay):
    await asyncio.sleep(delay)
    return f"Data from {url}"

async def main():
    tasks = [
        fetch("url1", 3),
        fetch("url2", 1),
        fetch("url3", 2),
    ]
    
    # Process results as they complete (not in order)
    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(result)
    
    # Output order:
    # Data from url2  (completed first — 1 sec)
    # Data from url3  (completed second — 2 sec)
    # Data from url1  (completed last — 3 sec)

asyncio.run(main())
```

### Task Cancellation

```python
import asyncio

async def long_running():
    try:
        while True:
            print("Working...")
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print("Task was cancelled! Cleaning up...")
        # Perform cleanup here
        raise  # Re-raise to mark task as cancelled

async def main():
    task = asyncio.create_task(long_running())
    
    await asyncio.sleep(3)  # Let it run for 3 seconds
    
    task.cancel()  # Request cancellation
    
    try:
        await task
    except asyncio.CancelledError:
        print("Main: confirmed task is cancelled")

asyncio.run(main())
```

### TaskGroup (Python 3.11+)

```python
import asyncio

async def fetch_data(name, delay):
    await asyncio.sleep(delay)
    if name == "bad":
        raise ValueError(f"{name} failed!")
    return f"{name}: data"

async def main():
    # TaskGroup automatically cancels all tasks if any one fails
    try:
        async with asyncio.TaskGroup() as tg:
            task1 = tg.create_task(fetch_data("A", 1))
            task2 = tg.create_task(fetch_data("B", 2))
            task3 = tg.create_task(fetch_data("C", 3))
        
        # All tasks completed successfully
        print(task1.result(), task2.result(), task3.result())
    
    except* ValueError as eg:
        # ExceptionGroup handling (Python 3.11+)
        for exc in eg.exceptions:
            print(f"Caught: {exc}")

asyncio.run(main())
```

---

## 6. Running Tasks Concurrently

### Pattern 1: Fire and Forget

```python
import asyncio

async def send_email(to: str):
    await asyncio.sleep(2)  # simulate sending
    print(f"Email sent to {to}")

async def handle_signup(username: str):
    # Save user to DB
    print(f"User {username} saved")
    
    # Fire-and-forget: don't wait for email
    asyncio.create_task(send_email(f"{username}@example.com"))
    
    # Return immediately
    return {"status": "created"}

async def main():
    result = await handle_signup("alice")
    print(result)
    await asyncio.sleep(3)  # Keep loop alive for background task

asyncio.run(main())
```

### Pattern 2: Fan-Out / Fan-In

```python
import asyncio

async def fetch_price(product_id: int) -> dict:
    await asyncio.sleep(0.5)
    return {"id": product_id, "price": product_id * 10.0}

async def main():
    product_ids = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    
    # Fan-out: create all tasks
    tasks = [fetch_price(pid) for pid in product_ids]
    
    # Fan-in: gather all results
    results = await asyncio.gather(*tasks)
    
    total = sum(r["price"] for r in results)
    print(f"Total: ${total}")  # All fetched concurrently!

asyncio.run(main())
```

### Pattern 3: Semaphore — Limiting Concurrency

```python
import asyncio

async def fetch_url(sem: asyncio.Semaphore, url: str):
    async with sem:  # Only N tasks can enter at a time
        print(f"Fetching {url}")
        await asyncio.sleep(1)
        print(f"Done {url}")
        return f"Data from {url}"

async def main():
    sem = asyncio.Semaphore(3)  # Max 3 concurrent fetches
    
    urls = [f"https://api.example.com/page/{i}" for i in range(10)]
    
    tasks = [fetch_url(sem, url) for url in urls]
    results = await asyncio.gather(*tasks)
    # Only 3 URLs fetched at a time, but all 10 complete

asyncio.run(main())
```

### Pattern 4: Producer-Consumer

```python
import asyncio
import random

async def producer(queue: asyncio.Queue, name: str):
    for i in range(5):
        item = f"{name}-item-{i}"
        await queue.put(item)
        print(f"Produced: {item}")
        await asyncio.sleep(random.uniform(0.1, 0.5))
    await queue.put(None)  # Sentinel to signal done

async def consumer(queue: asyncio.Queue, name: str):
    while True:
        item = await queue.get()
        if item is None:
            break
        print(f"{name} consumed: {item}")
        await asyncio.sleep(random.uniform(0.2, 0.8))
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=5)
    
    await asyncio.gather(
        producer(queue, "P1"),
        consumer(queue, "C1"),
        consumer(queue, "C2"),
    )

asyncio.run(main())
```

---

## 7. Async Generators & Iterators

### Async Generator

```python
import asyncio

async def async_range(start, stop):
    """Async generator — yields values with async pauses"""
    for i in range(start, stop):
        await asyncio.sleep(0.5)  # Simulate async work
        yield i

async def fetch_pages(total_pages):
    """Simulate paginated API calls"""
    for page in range(1, total_pages + 1):
        await asyncio.sleep(0.3)
        yield {"page": page, "data": [f"item_{i}" for i in range(3)]}

async def main():
    # Consuming an async generator with 'async for'
    async for num in async_range(0, 5):
        print(f"Got: {num}")
    
    # Practical: paginated API
    async for page_data in fetch_pages(3):
        print(f"Page {page_data['page']}: {page_data['data']}")

asyncio.run(main())
```

### Async Iterator (Class-based)

```python
import asyncio

class AsyncCounter:
    def __init__(self, limit):
        self.limit = limit
        self.count = 0
    
    def __aiter__(self):
        return self
    
    async def __anext__(self):
        if self.count >= self.limit:
            raise StopAsyncIteration
        self.count += 1
        await asyncio.sleep(0.2)
        return self.count

async def main():
    async for num in AsyncCounter(5):
        print(num)  # 1, 2, 3, 4, 5

asyncio.run(main())
```

### Async Comprehensions

```python
import asyncio

async def get_value(x):
    await asyncio.sleep(0.1)
    return x * 2

async def async_range(n):
    for i in range(n):
        await asyncio.sleep(0.1)
        yield i

async def main():
    # Async list comprehension
    results = [await get_value(i) for i in range(5)]
    print(results)  # [0, 2, 4, 6, 8]
    
    # Async generator comprehension
    results = [x async for x in async_range(5)]
    print(results)  # [0, 1, 2, 3, 4]
    
    # Async with condition
    evens = [x async for x in async_range(10) if x % 2 == 0]
    print(evens)  # [0, 2, 4, 6, 8]

asyncio.run(main())
```

---

## 8. Async Context Managers

### Creating Async Context Managers

```python
import asyncio

# Method 1: Class-based
class AsyncDBConnection:
    def __init__(self, db_url):
        self.db_url = db_url
        self.connection = None
    
    async def __aenter__(self):
        print(f"Connecting to {self.db_url}...")
        await asyncio.sleep(0.5)  # Simulate connection
        self.connection = {"status": "connected"}
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print("Closing connection...")
        await asyncio.sleep(0.1)
        self.connection = None
        return False  # Don't suppress exceptions

    async def query(self, sql):
        await asyncio.sleep(0.2)
        return [{"id": 1, "name": "Alice"}]

# Method 2: Using contextlib
from contextlib import asynccontextmanager

@asynccontextmanager
async def async_db_connection(db_url):
    print(f"Connecting to {db_url}...")
    await asyncio.sleep(0.5)
    conn = {"status": "connected", "url": db_url}
    try:
        yield conn
    finally:
        print("Closing connection...")
        await asyncio.sleep(0.1)

async def main():
    # Using class-based
    async with AsyncDBConnection("postgresql://localhost/mydb") as db:
        results = await db.query("SELECT * FROM users")
        print(results)
    
    # Using decorator-based
    async with async_db_connection("postgresql://localhost/mydb") as conn:
        print(f"Connected: {conn}")

asyncio.run(main())
```

---

## 9. Synchronization Primitives

### asyncio.Lock

```python
import asyncio

# Shared resource without lock — RACE CONDITION
counter = 0

async def unsafe_increment(name):
    global counter
    temp = counter
    await asyncio.sleep(0.01)  # Simulate some async work
    counter = temp + 1
    print(f"{name}: counter = {counter}")

# With lock — SAFE
lock = asyncio.Lock()

async def safe_increment(name):
    global counter
    async with lock:  # Only one coroutine at a time
        temp = counter
        await asyncio.sleep(0.01)
        counter = temp + 1
        print(f"{name}: counter = {counter}")

async def main():
    global counter
    
    # Unsafe — counter might not reach 10
    counter = 0
    await asyncio.gather(*[unsafe_increment(f"T{i}") for i in range(10)])
    print(f"Unsafe final: {counter}")  # Could be less than 10!
    
    # Safe — counter will always be 10
    counter = 0
    await asyncio.gather(*[safe_increment(f"T{i}") for i in range(10)])
    print(f"Safe final: {counter}")  # Always 10

asyncio.run(main())
```

### asyncio.Event

```python
import asyncio

async def waiter(event: asyncio.Event, name: str):
    print(f"{name}: waiting for event...")
    await event.wait()
    print(f"{name}: event received! Proceeding.")

async def trigger(event: asyncio.Event):
    print("Trigger: preparing...")
    await asyncio.sleep(2)
    print("Trigger: setting event!")
    event.set()  # All waiters wake up

async def main():
    event = asyncio.Event()
    
    await asyncio.gather(
        waiter(event, "Worker-1"),
        waiter(event, "Worker-2"),
        waiter(event, "Worker-3"),
        trigger(event),
    )

asyncio.run(main())
```

### asyncio.Semaphore

```python
import asyncio

async def limited_resource(sem: asyncio.Semaphore, worker_id: int):
    print(f"Worker {worker_id}: waiting for semaphore")
    async with sem:
        print(f"Worker {worker_id}: acquired! Working...")
        await asyncio.sleep(2)
        print(f"Worker {worker_id}: released")

async def main():
    sem = asyncio.Semaphore(2)  # Only 2 workers at a time
    
    await asyncio.gather(*[limited_resource(sem, i) for i in range(6)])
    # Workers 0,1 run → then 2,3 → then 4,5

asyncio.run(main())
```

### asyncio.Condition

```python
import asyncio

async def consumer(condition: asyncio.Condition, name: str, shared_data: list):
    async with condition:
        while not shared_data:
            print(f"{name}: waiting for data...")
            await condition.wait()
        item = shared_data.pop(0)
        print(f"{name}: consumed {item}")

async def producer(condition: asyncio.Condition, shared_data: list):
    await asyncio.sleep(1)
    async with condition:
        shared_data.append("item-1")
        shared_data.append("item-2")
        print("Producer: added items, notifying all")
        condition.notify_all()

async def main():
    condition = asyncio.Condition()
    shared_data = []
    
    await asyncio.gather(
        consumer(condition, "C1", shared_data),
        consumer(condition, "C2", shared_data),
        producer(condition, shared_data),
    )

asyncio.run(main())
```

---

## 10. Async Queues

```python
import asyncio
import random

async def producer(queue: asyncio.Queue, name: str, count: int):
    for i in range(count):
        item = f"{name}-{i}"
        await queue.put(item)  # Blocks if queue is full
        print(f"[{name}] Produced: {item} | Queue size: {queue.qsize()}")
        await asyncio.sleep(random.uniform(0.1, 0.3))

async def consumer(queue: asyncio.Queue, name: str):
    while True:
        item = await queue.get()  # Blocks if queue is empty
        if item is None:
            break
        print(f"[{name}] Processing: {item}")
        await asyncio.sleep(random.uniform(0.2, 0.5))
        queue.task_done()
        print(f"[{name}] Done: {item}")

async def main():
    queue = asyncio.Queue(maxsize=5)  # Bounded queue
    
    # Start producers and consumers
    producers = [
        asyncio.create_task(producer(queue, "P1", 5)),
        asyncio.create_task(producer(queue, "P2", 5)),
    ]
    consumers = [
        asyncio.create_task(consumer(queue, "C1")),
        asyncio.create_task(consumer(queue, "C2")),
        asyncio.create_task(consumer(queue, "C3")),
    ]
    
    # Wait for all items to be produced
    await asyncio.gather(*producers)
    
    # Wait until all items are processed
    await queue.join()
    
    # Stop consumers
    for _ in consumers:
        await queue.put(None)
    await asyncio.gather(*consumers)

asyncio.run(main())
```

### Priority Queue

```python
import asyncio

async def main():
    pq = asyncio.PriorityQueue()
    
    # Lower number = higher priority
    await pq.put((3, "Low priority task"))
    await pq.put((1, "High priority task"))
    await pq.put((2, "Medium priority task"))
    
    while not pq.empty():
        priority, item = await pq.get()
        print(f"Priority {priority}: {item}")

asyncio.run(main())
# High priority task
# Medium priority task
# Low priority task
```

---

## 11. Async HTTP with aiohttp

### Async HTTP Client

```python
import asyncio
import aiohttp

# pip install aiohttp

async def fetch_url(session: aiohttp.ClientSession, url: str) -> dict:
    async with session.get(url) as response:
        data = await response.json()
        return {"url": url, "status": response.status, "data": data}

async def main():
    async with aiohttp.ClientSession() as session:
        # Single request
        async with session.get("https://jsonplaceholder.typicode.com/posts/1") as resp:
            data = await resp.json()
            print(data["title"])
        
        # Multiple concurrent requests
        urls = [
            f"https://jsonplaceholder.typicode.com/posts/{i}"
            for i in range(1, 11)
        ]
        
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        
        for r in results:
            print(f"{r['url']}: {r['status']}")

asyncio.run(main())
```

### Async HTTP with Rate Limiting

```python
import asyncio
import aiohttp

async def fetch_with_limit(
    session: aiohttp.ClientSession,
    url: str,
    semaphore: asyncio.Semaphore
):
    async with semaphore:
        async with session.get(url) as response:
            return await response.json()

async def main():
    semaphore = asyncio.Semaphore(5)  # Max 5 concurrent requests
    
    async with aiohttp.ClientSession() as session:
        urls = [f"https://api.example.com/data/{i}" for i in range(100)]
        tasks = [fetch_with_limit(session, url, semaphore) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        successful = [r for r in results if not isinstance(r, Exception)]
        failed = [r for r in results if isinstance(r, Exception)]
        print(f"Success: {len(successful)}, Failed: {len(failed)}")

asyncio.run(main())
```

---

## 12. Async Database Access

### With asyncpg (PostgreSQL)

```python
import asyncio
import asyncpg

# pip install asyncpg

async def main():
    # Connection pool
    pool = await asyncpg.create_pool(
        host="localhost",
        database="mydb",
        user="user",
        password="password",
        min_size=5,
        max_size=20,
    )
    
    async with pool.acquire() as conn:
        # Single query
        row = await conn.fetchrow("SELECT * FROM users WHERE id = $1", 1)
        print(dict(row))
        
        # Multiple rows
        rows = await conn.fetch("SELECT * FROM users LIMIT 10")
        for row in rows:
            print(dict(row))
        
        # Insert
        await conn.execute(
            "INSERT INTO users(name, email) VALUES($1, $2)",
            "Alice", "alice@example.com"
        )
        
        # Transaction
        async with conn.transaction():
            await conn.execute("UPDATE accounts SET balance = balance - 100 WHERE id = $1", 1)
            await conn.execute("UPDATE accounts SET balance = balance + 100 WHERE id = $1", 2)
    
    await pool.close()

asyncio.run(main())
```

### With SQLAlchemy Async (2.0+)

```python
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import select

# pip install sqlalchemy[asyncio] aiosqlite

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]
    email: Mapped[str]

async def main():
    engine = create_async_engine("sqlite+aiosqlite:///mydb.db", echo=True)
    
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    
    async_session = async_sessionmaker(engine, class_=AsyncSession)
    
    # Insert
    async with async_session() as session:
        user = User(name="Alice", email="alice@example.com")
        session.add(user)
        await session.commit()
    
    # Query
    async with async_session() as session:
        result = await session.execute(select(User).where(User.name == "Alice"))
        user = result.scalar_one()
        print(f"{user.name}: {user.email}")
    
    await engine.dispose()

asyncio.run(main())
```

---

## 13. Threading vs Multiprocessing vs Asyncio

### Side-by-Side Comparison

```python
import asyncio
import threading
import multiprocessing
import time
import requests

# ═══════════════════════════════════════════
# SCENARIO: Fetch 10 URLs
# ═══════════════════════════════════════════

urls = [f"https://jsonplaceholder.typicode.com/posts/{i}" for i in range(1, 11)]

# --- 1. SYNCHRONOUS (Baseline) ---
def sync_fetch():
    start = time.time()
    results = []
    for url in urls:
        resp = requests.get(url)
        results.append(resp.status_code)
    print(f"Sync: {time.time() - start:.2f}s")
    return results

# --- 2. THREADING ---
def threaded_fetch():
    start = time.time()
    results = []
    lock = threading.Lock()
    
    def fetch(url):
        resp = requests.get(url)
        with lock:
            results.append(resp.status_code)
    
    threads = [threading.Thread(target=fetch, args=(url,)) for url in urls]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    
    print(f"Threading: {time.time() - start:.2f}s")
    return results

# --- 3. MULTIPROCESSING ---
def mp_fetch_one(url):
    resp = requests.get(url)
    return resp.status_code

def mp_fetch():
    start = time.time()
    with multiprocessing.Pool(10) as pool:
        results = pool.map(mp_fetch_one, urls)
    print(f"Multiprocessing: {time.time() - start:.2f}s")
    return results

# --- 4. ASYNCIO ---
import aiohttp

async def async_fetch():
    start = time.time()
    async with aiohttp.ClientSession() as session:
        tasks = []
        for url in urls:
            tasks.append(session.get(url))
        responses = await asyncio.gather(*tasks)
        results = [r.status for r in responses]
    print(f"Asyncio: {time.time() - start:.2f}s")
    return results

# Results (approximate):
# Sync:            5.0s  (sequential)
# Threading:       0.7s  (concurrent, 10 threads)
# Multiprocessing: 1.0s  (parallel, 10 processes — more overhead)
# Asyncio:         0.5s  (concurrent, 1 thread — least overhead)
```

### When to Use What

```
┌─────────────────────────────────────────────────────────┐
│                    DECISION TREE                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Is the task I/O-bound?                                 │
│  ├── YES → Use asyncio (best) or threading              │
│  │         Examples: HTTP calls, DB queries, file I/O   │
│  │                                                      │
│  └── NO (CPU-bound?)                                    │
│      ├── YES → Use multiprocessing                      │
│      │         Examples: data processing, ML training   │
│      │                                                  │
│      └── Mixed → Use multiprocessing + asyncio          │
│                  (run async event loop per process)      │
│                                                         │
│  Need to integrate with sync library?                   │
│  ├── YES → Use threading or run_in_executor()           │
│  └── NO  → Use asyncio                                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### run_in_executor — Bridge Between Sync and Async

```python
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def blocking_io_operation():
    """Simulates a blocking I/O call (e.g., sync DB driver)"""
    time.sleep(2)
    return "IO result"

def cpu_heavy_computation(n):
    """Simulates CPU-heavy work"""
    return sum(i * i for i in range(n))

async def main():
    loop = asyncio.get_running_loop()
    
    # Run blocking I/O in a thread pool (doesn't block event loop)
    result = await loop.run_in_executor(
        ThreadPoolExecutor(max_workers=4),
        blocking_io_operation
    )
    print(result)
    
    # Run CPU-heavy work in a process pool (bypasses GIL)
    result = await loop.run_in_executor(
        ProcessPoolExecutor(max_workers=4),
        cpu_heavy_computation,
        10_000_000
    )
    print(result)
    
    # Using default executor (thread pool)
    result = await loop.run_in_executor(None, blocking_io_operation)
    print(result)

asyncio.run(main())
```

---

## 14. How Python Web Frameworks Handle Multiple Users

### The Big Picture

```
                    MULTIPLE USERS
                   ┌────┐ ┌────┐ ┌────┐
                   │U1  │ │U2  │ │U3  │
                   └──┬─┘ └──┬─┘ └──┬─┘
                      │      │      │
                      ▼      ▼      ▼
              ┌──────────────────────────┐
              │     REVERSE PROXY        │
              │     (Nginx / Traefik)    │
              └──────────┬───────────────┘
                         │
              ┌──────────▼───────────────┐
              │     PYTHON APP SERVER    │
              │                          │
              │  ┌─ Gunicorn (WSGI) ──┐  │
              │  │  Worker 1 (sync)   │  │
              │  │  Worker 2 (sync)   │  │
              │  │  Worker 3 (sync)   │  │
              │  │  Worker 4 (sync)   │  │
              │  └────────────────────┘  │
              │                          │
              │  ┌─ Uvicorn (ASGI) ───┐  │
              │  │  Worker 1 (async)  │  │
              │  │   └─ Event Loop    │  │
              │  │      ├─ Req A      │  │
              │  │      ├─ Req B      │  │
              │  │      └─ Req C      │  │
              │  │  Worker 2 (async)  │  │
              │  └────────────────────┘  │
              └──────────────────────────┘
```

### WSGI vs ASGI

```
WSGI (Web Server Gateway Interface) — SYNCHRONOUS
═══════════════════════════════════════════════════
- One request per thread/process
- Frameworks: Flask, Django (traditional)
- Servers: Gunicorn, uWSGI
- Each request blocks until complete

    Thread 1: |----Request A----|
    Thread 2: |----Request B----|
    Thread 3: |----Request C----|
    Thread 4: (idle, waiting for next request)

ASGI (Asynchronous Server Gateway Interface) — ASYNC
═══════════════════════════════════════════════════
- Many requests per thread (via event loop)
- Frameworks: FastAPI, Django (channels), Starlette
- Servers: Uvicorn, Hypercorn, Daphne
- Requests yield during I/O

    Thread 1: |--Req A--|        |--Req A--|
              |--Req B--|--Req B--|
              |--Req C--|              |--Req C--|
    (All on same thread, interleaved at await points)
```

---

## 15. Scenario 1: Flask (Sync WSGI)

### How Flask Handles Multiple Requests

```python
from flask import Flask, jsonify
import time

app = Flask(__name__)

@app.route("/api/users/<int:user_id>")
def get_user(user_id):
    # This entire function BLOCKS the worker thread
    time.sleep(2)  # Simulate DB query
    return jsonify({"id": user_id, "name": f"User {user_id}"})

@app.route("/api/heavy")
def heavy_computation():
    # CPU-bound: blocks the worker thread
    result = sum(i * i for i in range(10_000_000))
    return jsonify({"result": result})
```

### What happens when 5 users hit `/api/users` simultaneously?

```
With Gunicorn: gunicorn app:app --workers 4 --threads 2

Total capacity: 4 workers × 2 threads = 8 concurrent requests

User 1 → Worker 1, Thread 1 → Blocked for 2s → Response
User 2 → Worker 1, Thread 2 → Blocked for 2s → Response
User 3 → Worker 2, Thread 1 → Blocked for 2s → Response
User 4 → Worker 2, Thread 2 → Blocked for 2s → Response
User 5 → Worker 3, Thread 1 → Blocked for 2s → Response

All 5 served in ~2 seconds (parallel across workers/threads)

If 10+ users hit simultaneously:
- 8 served immediately
- 2 wait in queue until a thread is free
- Total: some users wait ~4 seconds
```

### Flask with Gunicorn Configuration

```bash
# Sync workers (default) — each worker = 1 process with 1 thread
gunicorn app:app --workers 4
# Can handle: 4 concurrent requests

# Sync workers with threads
gunicorn app:app --workers 4 --threads 4
# Can handle: 16 concurrent requests

# Async workers with gevent (monkey-patched async)
gunicorn app:app --workers 4 --worker-class gevent --worker-connections 1000
# Can handle: 4000 concurrent connections (async I/O via greenlets)
```

---

## 16. Scenario 2: Django (Sync WSGI + Async Support)

### Traditional Django (Sync)

```python
# views.py
from django.http import JsonResponse
import time

def get_user(request, user_id):
    # Synchronous — blocks the worker
    time.sleep(1)  # Simulate DB query
    return JsonResponse({"id": user_id, "name": f"User {user_id}"})
```

### Django Async Views (4.1+)

```python
# views.py
import asyncio
from django.http import JsonResponse

async def get_user(request, user_id):
    # Async — doesn't block the worker
    await asyncio.sleep(1)  # Simulate async DB query
    return JsonResponse({"id": user_id, "name": f"User {user_id}"})

async def get_dashboard(request):
    # Fetch multiple things concurrently
    user_data, orders, notifications = await asyncio.gather(
        fetch_user(request.user.id),
        fetch_orders(request.user.id),
        fetch_notifications(request.user.id),
    )
    return JsonResponse({
        "user": user_data,
        "orders": orders,
        "notifications": notifications,
    })

async def fetch_user(user_id):
    await asyncio.sleep(0.5)
    return {"id": user_id, "name": "Alice"}

async def fetch_orders(user_id):
    await asyncio.sleep(0.3)
    return [{"id": 1, "total": 99.99}]

async def fetch_notifications(user_id):
    await asyncio.sleep(0.2)
    return [{"msg": "New order!"}]
```

### Django — Mixing Sync and Async

```python
from django.http import JsonResponse
from asgiref.sync import sync_to_async, async_to_sync

# Django ORM is synchronous — wrap it for async views
from myapp.models import User

async def get_user_async(request, user_id):
    # sync_to_async wraps Django ORM call to run in thread pool
    user = await sync_to_async(User.objects.get)(id=user_id)
    return JsonResponse({"name": user.name, "email": user.email})

# Or use the decorator
@sync_to_async
def get_user_from_db(user_id):
    return User.objects.get(id=user_id)

async def get_user_view(request, user_id):
    user = await get_user_from_db(user_id)
    return JsonResponse({"name": user.name})
```

### Django Deployment

```bash
# Sync (traditional)
gunicorn myproject.wsgi:application --workers 4

# Async (with ASGI)
uvicorn myproject.asgi:application --workers 4
# or
daphne myproject.asgi:application
```

---

## 17. Scenario 3: FastAPI (Async ASGI)

### FastAPI — The Async-First Framework

```python
from fastapi import FastAPI
import asyncio
import httpx

app = FastAPI()

# ─── ASYNC ENDPOINT ───
@app.get("/api/users/{user_id}")
async def get_user(user_id: int):
    # Non-blocking — event loop handles thousands of these concurrently
    await asyncio.sleep(1)  # Simulate async DB query
    return {"id": user_id, "name": f"User {user_id}"}

# ─── SYNC ENDPOINT ───
@app.get("/api/sync-users/{user_id}")
def get_user_sync(user_id: int):
    # FastAPI AUTOMATICALLY runs this in a thread pool!
    # So it doesn't block the event loop
    import time
    time.sleep(1)  # This is OK — runs in separate thread
    return {"id": user_id, "name": f"User {user_id}"}
```

### FastAPI — How It Handles Requests Internally

```
FastAPI runs on Uvicorn (ASGI server)

┌─────────────────────────────────────────────┐
│ Uvicorn Worker (1 process, 1 event loop)    │
│                                             │
│  Event Loop                                 │
│  ├── Request 1 (async def) → runs directly  │
│  │   └── await db.query() → YIELDS          │
│  │       └── event loop runs other tasks    │
│  ├── Request 2 (async def) → runs directly  │
│  │   └── await sleep(1) → YIELDS            │
│  ├── Request 3 (def — sync!) → thread pool  │
│  │   └── ThreadPoolExecutor.submit()        │
│  └── Request 4 (async def) → runs directly  │
│                                             │
│  Thread Pool (for sync endpoints)           │
│  ├── Thread 1: Request 3 running            │
│  ├── Thread 2: (idle)                       │
│  └── Thread 3: (idle)                       │
└─────────────────────────────────────────────┘

KEY INSIGHT:
- async def endpoints → run on event loop (lightweight)
- def endpoints → run in thread pool (heavier but still non-blocking)
- Both can handle many concurrent requests!
```

### FastAPI — Concurrent External API Calls

```python
from fastapi import FastAPI
import httpx
import asyncio

app = FastAPI()

@app.get("/api/dashboard")
async def get_dashboard():
    async with httpx.AsyncClient() as client:
        # Fetch 3 APIs concurrently — total time = slowest one
        user_task = client.get("https://api.example.com/user/1")
        orders_task = client.get("https://api.example.com/orders?user=1")
        stats_task = client.get("https://api.example.com/stats?user=1")
        
        user_resp, orders_resp, stats_resp = await asyncio.gather(
            user_task, orders_task, stats_task
        )
        
        return {
            "user": user_resp.json(),
            "orders": orders_resp.json(),
            "stats": stats_resp.json(),
        }
```

### FastAPI — Dependencies with Async

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy import select

app = FastAPI()

engine = create_async_engine("sqlite+aiosqlite:///app.db")
async_session = async_sessionmaker(engine, class_=AsyncSession)

async def get_db():
    """Dependency that provides a DB session"""
    async with async_session() as session:
        yield session  # Async generator dependency

@app.get("/api/users")
async def list_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    users = result.scalars().all()
    return [{"id": u.id, "name": u.name} for u in users]

@app.get("/api/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return {"id": user.id, "name": user.name}
```

---

## 18. Scenario 4: What Happens When 1000 Users Hit the Same API?

### Flask (Sync) — The Worst Case

```python
# Flask app
from flask import Flask, jsonify
import time

app = Flask(__name__)

@app.route("/api/data")
def get_data():
    time.sleep(0.5)  # Simulate DB query (500ms)
    return jsonify({"data": "result"})
```

```
Deployment: gunicorn app:app --workers 4 --threads 4
Max concurrent: 16 requests at a time

1000 requests hitting /api/data:
- Batch 1: 16 requests served (0.5s each)
- Batch 2: 16 requests served
- ...
- Total batches: 1000/16 = 63 batches
- Total time: 63 × 0.5s ≈ 31.5 seconds
- Last user waits: ~31 seconds! 😱

Memory: 4 workers × ~30MB = ~120MB
```

### FastAPI (Async) — The Best Case

```python
# FastAPI app
from fastapi import FastAPI
import asyncio

app = FastAPI()

@app.get("/api/data")
async def get_data():
    await asyncio.sleep(0.5)  # Simulate async DB query (500ms)
    return {"data": "result"}
```

```
Deployment: uvicorn app:app --workers 4
Each worker has 1 event loop that handles thousands of connections

1000 requests hitting /api/data:
- All 1000 coroutines start immediately
- All 1000 await sleep(0.5) concurrently
- All 1000 complete after ~0.5 seconds
- Every user gets response in ~0.5 seconds! 🎉

Memory: 4 workers × ~25MB + coroutines (~KB each) ≈ ~110MB
```

### Comparison Table (1000 concurrent requests, 500ms DB query)

```
┌───────────────────────┬─────────┬──────────┬──────────┐
│ Setup                 │ Avg     │ P99      │ Memory   │
│                       │ Latency │ Latency  │          │
├───────────────────────┼─────────┼──────────┼──────────┤
│ Flask + Gunicorn      │ ~15s    │ ~31s     │ ~120MB   │
│ (4w × 4t = 16 conc)  │         │          │          │
├───────────────────────┼─────────┼──────────┼──────────┤
│ Flask + Gevent        │ ~0.6s   │ ~1.2s    │ ~150MB   │
│ (4w × 1000 greenlets) │         │          │          │
├───────────────────────┼─────────┼──────────┼──────────┤
│ Django + Gunicorn     │ ~15s    │ ~31s     │ ~200MB   │
│ (4w × 4t = 16 conc)  │         │          │          │
├───────────────────────┼─────────┼──────────┼──────────┤
│ Django + Uvicorn      │ ~0.6s   │ ~1s      │ ~160MB   │
│ (async views)         │         │          │          │
├───────────────────────┼─────────┼──────────┼──────────┤
│ FastAPI + Uvicorn     │ ~0.5s   │ ~0.7s    │ ~110MB   │
│ (4 workers, async)    │         │          │          │
└───────────────────────┴─────────┴──────────┴──────────┘
```

---

## 19. Scenario 5: CPU-Bound vs I/O-Bound API Handlers

### I/O-Bound Handler (Async is Perfect)

```python
from fastapi import FastAPI
import httpx
import asyncio

app = FastAPI()

@app.get("/api/aggregate")
async def aggregate_data():
    """
    I/O-Bound: Fetches data from 3 external APIs.
    Total I/O wait: ~2 seconds. CPU work: ~0ms.
    Async handles this beautifully.
    """
    async with httpx.AsyncClient() as client:
        results = await asyncio.gather(
            client.get("https://api1.example.com/data"),   # 1.0s
            client.get("https://api2.example.com/data"),   # 0.8s
            client.get("https://api3.example.com/data"),   # 2.0s
        )
    
    # Total time: ~2.0s (max of all three, not sum)
    return {"results": [r.json() for r in results]}
```

### CPU-Bound Handler (Async Does NOT Help!)

```python
from fastapi import FastAPI
import asyncio
from concurrent.futures import ProcessPoolExecutor

app = FastAPI()
process_pool = ProcessPoolExecutor(max_workers=4)

# ❌ BAD: CPU-heavy work in async handler blocks the event loop!
@app.get("/api/compute-bad")
async def compute_bad():
    # This blocks the entire event loop for 5 seconds
    # NO other requests can be processed during this time!
    result = sum(i * i for i in range(50_000_000))
    return {"result": result}

# ✅ GOOD: Offload CPU-heavy work to process pool
@app.get("/api/compute-good")
async def compute_good():
    loop = asyncio.get_running_loop()
    
    # Run in a separate process (bypasses GIL)
    result = await loop.run_in_executor(
        process_pool,
        cpu_intensive_task,
        50_000_000
    )
    return {"result": result}

def cpu_intensive_task(n):
    return sum(i * i for i in range(n))
```

### Mixed Handler

```python
from fastapi import FastAPI
import asyncio
from concurrent.futures import ProcessPoolExecutor

app = FastAPI()
process_pool = ProcessPoolExecutor(max_workers=4)

@app.get("/api/report")
async def generate_report(user_id: int):
    """Mix of I/O and CPU work"""
    
    # Step 1: I/O bound — fetch data from DB (async, non-blocking)
    raw_data = await fetch_from_database(user_id)
    
    # Step 2: CPU bound — process data (offload to process pool)
    loop = asyncio.get_running_loop()
    processed = await loop.run_in_executor(
        process_pool,
        heavy_data_processing,
        raw_data
    )
    
    # Step 3: I/O bound — save results (async, non-blocking)
    await save_to_database(user_id, processed)
    
    return {"status": "done", "data": processed}
```

---

## 20. Scenario 6: Blocking Code Inside Async Handler

### The Problem

```python
from fastapi import FastAPI
import asyncio
import time
import requests  # ← This is a SYNC library!

app = FastAPI()

# ❌ DISASTROUS: Sync HTTP call inside async handler
@app.get("/api/data")
async def get_data():
    # requests.get() BLOCKS the event loop!
    # While this runs, NO other request can be processed!
    response = requests.get("https://slow-api.example.com/data")  # 3 seconds
    return response.json()

# ❌ DISASTROUS: time.sleep inside async handler
@app.get("/api/slow")
async def slow_endpoint():
    time.sleep(5)  # Blocks the ENTIRE event loop for 5 seconds!
    return {"status": "done"}
```

```
What happens with 10 concurrent requests to /api/data:

Event Loop: |---Req1 (3s blocking)---|---Req2 (3s blocking)---|---...
            ALL other requests frozen while one is blocking!

Total time for 10 requests: 30 seconds! (sequential, not concurrent)
```

### The Solutions

```python
from fastapi import FastAPI
import asyncio
import httpx
from concurrent.futures import ThreadPoolExecutor

app = FastAPI()
thread_pool = ThreadPoolExecutor(max_workers=10)

# ✅ Solution 1: Use async HTTP library
@app.get("/api/data-v1")
async def get_data_v1():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://slow-api.example.com/data")
        return response.json()

# ✅ Solution 2: Run sync code in thread pool
@app.get("/api/data-v2")
async def get_data_v2():
    import requests
    loop = asyncio.get_running_loop()
    response = await loop.run_in_executor(
        thread_pool,
        requests.get,
        "https://slow-api.example.com/data"
    )
    return response.json()

# ✅ Solution 3: Use def (not async def) — FastAPI auto-threads it
@app.get("/api/data-v3")
def get_data_v3():  # NOTE: plain 'def', not 'async def'
    import requests
    # FastAPI automatically runs this in a thread pool!
    response = requests.get("https://slow-api.example.com/data")
    return response.json()
```

### Decision: When to Use `async def` vs `def` in FastAPI

```
┌─────────────────────────────────────────────────────┐
│         async def  vs  def  in FastAPI               │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Use 'async def' when:                              │
│  ✅ All I/O uses async libraries (httpx, asyncpg)   │
│  ✅ You use 'await' inside the function              │
│  ✅ No blocking calls at all                        │
│                                                     │
│  Use 'def' (sync) when:                             │
│  ✅ You use sync libraries (requests, psycopg2)     │
│  ✅ You use sync ORM (Django ORM, sync SQLAlchemy)  │
│  ✅ You call blocking functions                     │
│  → FastAPI auto-runs it in a thread pool            │
│                                                     │
│  NEVER do this:                                     │
│  ❌ async def + requests.get() (blocks event loop!) │
│  ❌ async def + time.sleep() (blocks event loop!)   │
│  ❌ async def + sync DB driver (blocks event loop!) │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 21. Scenario 7: Database Connection Pooling

### Why Connection Pooling Matters

```
WITHOUT Pool (100 users × 1 query each):
──────────────────────────────────────────
User 1 → Open Connection → Query → Close Connection
User 2 → Open Connection → Query → Close Connection
...
User 100 → Open Connection → Query → Close Connection

Problem: 100 TCP handshakes, 100 authentications = SLOW + DB overloaded

WITH Pool (100 users, pool size 20):
──────────────────────────────────────────
Pool: [Conn1, Conn2, ..., Conn20]  (pre-opened)

User 1 → Borrow Conn1 → Query → Return Conn1
User 2 → Borrow Conn2 → Query → Return Conn2
...
User 21 → WAIT for free connection...
User 1 done → Conn1 returned to pool
User 21 → Borrow Conn1 → Query → Return Conn1

Benefits: Reuse connections, limit DB load, faster response
```

### Async Connection Pool with FastAPI

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from contextlib import asynccontextmanager

# Create engine with connection pool
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/mydb",
    pool_size=20,          # Maintain 20 connections
    max_overflow=10,       # Allow 10 extra under load (total max: 30)
    pool_timeout=30,       # Wait 30s for a connection before error
    pool_recycle=1800,     # Recycle connections every 30 minutes
    pool_pre_ping=True,    # Verify connection is alive before using
)

async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: pool is created by engine
    yield
    # Shutdown: close all connections
    await engine.dispose()

app = FastAPI(lifespan=lifespan)

async def get_db() -> AsyncSession:
    async with async_session() as session:
        yield session

@app.get("/api/users")
async def list_users(db: AsyncSession = Depends(get_db)):
    # Borrows a connection from the pool
    result = await db.execute(select(User))
    # Connection is returned to pool when session closes
    return result.scalars().all()
```

### Pool Behavior Under Load

```
100 concurrent requests, pool_size=20, max_overflow=10

Requests 1-20:  → Get connection immediately from pool
Requests 21-30: → max_overflow allows 10 more connections (new TCP)
Requests 31+:   → WAIT in queue for pool_timeout seconds
                   └─ If timeout: raise TimeoutError (503 to user)
                   └─ If connection freed: proceed

After load subsides:
- 20 connections stay in pool (pool_size)
- 10 overflow connections close after idle timeout
```

---

## 22. Scenario 8: Real-World Production Setup

### Full Production Architecture

```
                        Internet
                           │
                    ┌──────▼──────┐
                    │  Load       │
                    │  Balancer   │  (AWS ALB / Nginx)
                    └──────┬──────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
     ┌──────▼──────┐ ┌────▼──────┐ ┌─────▼─────┐
     │  Server 1   │ │ Server 2  │ │ Server 3  │
     │  Uvicorn    │ │ Uvicorn   │ │ Uvicorn   │
     │  4 workers  │ │ 4 workers │ │ 4 workers │
     │  FastAPI    │ │ FastAPI   │ │ FastAPI   │
     └──────┬──────┘ └────┬──────┘ └─────┬─────┘
            │              │              │
            └──────────────┼──────────────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
     ┌──────▼──────┐ ┌────▼──────┐ ┌─────▼─────┐
     │ PostgreSQL  │ │   Redis   │ │  Celery   │
     │ (Primary)   │ │  (Cache)  │ │ (Workers) │
     │ + Replicas  │ │           │ │           │
     └─────────────┘ └───────────┘ └───────────┘
```

### Complete FastAPI Production App

```python
# app/main.py
from fastapi import FastAPI, Depends, HTTPException
from contextlib import asynccontextmanager
import asyncio
import redis.asyncio as aioredis
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from concurrent.futures import ProcessPoolExecutor

# ─── Configuration ───
DATABASE_URL = "postgresql+asyncpg://user:pass@db:5432/myapp"
REDIS_URL = "redis://redis:6379/0"

# ─── Engine & Pool ───
engine = create_async_engine(DATABASE_URL, pool_size=20, max_overflow=10)
SessionLocal = async_sessionmaker(engine, class_=AsyncSession)

# ─── Process Pool for CPU work ───
cpu_pool = ProcessPoolExecutor(max_workers=4)

# ─── App Lifecycle ───
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    app.state.redis = aioredis.from_url(REDIS_URL)
    yield
    # Shutdown
    await app.state.redis.close()
    await engine.dispose()
    cpu_pool.shutdown(wait=False)

app = FastAPI(lifespan=lifespan)

# ─── Dependencies ───
async def get_db():
    async with SessionLocal() as session:
        yield session

async def get_redis():
    return app.state.redis

# ─── API Endpoints ───

@app.get("/api/users/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
    redis = Depends(get_redis),
):
    # 1. Check cache first
    cached = await redis.get(f"user:{user_id}")
    if cached:
        return {"source": "cache", "data": cached.decode()}
    
    # 2. Query database
    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    # 3. Cache the result
    await redis.setex(f"user:{user_id}", 300, user.name)  # Cache for 5 min
    
    return {"source": "database", "data": user.name}

@app.get("/api/analytics")
async def get_analytics(db: AsyncSession = Depends(get_db)):
    # Fetch multiple queries concurrently
    users_count, orders_total, active_sessions = await asyncio.gather(
        db.scalar(select(func.count()).select_from(User)),
        db.scalar(select(func.sum(Order.total))),
        db.scalar(select(func.count()).select_from(Session).where(Session.active == True)),
    )
    
    return {
        "users": users_count,
        "revenue": float(orders_total or 0),
        "active_sessions": active_sessions,
    }

@app.post("/api/reports/generate")
async def generate_report(report_id: int, db: AsyncSession = Depends(get_db)):
    # 1. Fetch data (I/O-bound → async)
    data = await db.execute(select(SalesData).where(SalesData.report_id == report_id))
    rows = data.scalars().all()
    
    # 2. Process data (CPU-bound → process pool)
    loop = asyncio.get_running_loop()
    report = await loop.run_in_executor(
        cpu_pool,
        process_report_data,
        [row.to_dict() for row in rows]
    )
    
    # 3. Save result (I/O-bound → async)
    await save_report(db, report_id, report)
    
    return {"status": "generated", "report_id": report_id}
```

### Deployment Commands

```bash
# Development
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Production (single server)
uvicorn app.main:app \
    --host 0.0.0.0 \
    --port 8000 \
    --workers 4 \
    --loop uvloop \
    --http httptools \
    --access-log \
    --log-level info

# Production with Gunicorn (process manager)
gunicorn app.main:app \
    --worker-class uvicorn.workers.UvicornWorker \
    --workers 4 \
    --bind 0.0.0.0:8000 \
    --timeout 120 \
    --graceful-timeout 30 \
    --keep-alive 5
```

### Docker Production Setup

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Run with Gunicorn + Uvicorn workers
CMD ["gunicorn", "app.main:app", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--workers", "4", \
     "--bind", "0.0.0.0:8000"]
```

### Nginx Reverse Proxy

```nginx
upstream fastapi_backend {
    least_conn;                    # Load balance by fewest connections
    server app1:8000 max_fails=3;
    server app2:8000 max_fails=3;
    server app3:8000 max_fails=3;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://fastapi_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
    }
}
```

---

## 23. Common Pitfalls & Best Practices

### Pitfall 1: Forgetting to Await

```python
# ❌ WRONG — coroutine never executes!
async def main():
    result = fetch_data()  # Returns coroutine object, doesn't run!
    print(result)  # <coroutine object fetch_data at 0x...>

# ✅ RIGHT
async def main():
    result = await fetch_data()
    print(result)
```

### Pitfall 2: Blocking the Event Loop

```python
# ❌ WRONG
async def handler():
    time.sleep(5)       # Blocks entire event loop!
    requests.get(url)   # Blocks entire event loop!
    open("big.csv").read()  # Blocks entire event loop!

# ✅ RIGHT
async def handler():
    await asyncio.sleep(5)
    async with httpx.AsyncClient() as c:
        await c.get(url)
    async with aiofiles.open("big.csv") as f:
        data = await f.read()
```

### Pitfall 3: Creating Too Many Concurrent Connections

```python
# ❌ WRONG — overwhelms the database
async def main():
    tasks = [query_db(i) for i in range(10000)]
    await asyncio.gather(*tasks)  # 10,000 simultaneous DB connections!

# ✅ RIGHT — limit concurrency
async def main():
    sem = asyncio.Semaphore(50)  # Max 50 concurrent
    
    async def limited_query(i):
        async with sem:
            return await query_db(i)
    
    tasks = [limited_query(i) for i in range(10000)]
    await asyncio.gather(*tasks)
```

### Pitfall 4: Not Handling Cancellation

```python
# ❌ WRONG — resource leak on cancellation
async def process():
    conn = await connect_to_db()
    result = await conn.query("SELECT ...")  # If cancelled here...
    await conn.close()  # ...this never runs!

# ✅ RIGHT — use try/finally or async with
async def process():
    async with connect_to_db() as conn:  # Auto-closes on cancel
        result = await conn.query("SELECT ...")
        return result

# Or with try/finally
async def process():
    conn = await connect_to_db()
    try:
        result = await conn.query("SELECT ...")
        return result
    finally:
        await conn.close()  # Always runs, even on cancel
```

### Pitfall 5: Shared Mutable State

```python
# ❌ WRONG — race condition (even in asyncio!)
results = []

async def fetch_and_store(url):
    data = await fetch(url)
    results.append(data)  # Safe for append, but...
    if len(results) > 10:
        results.clear()   # ← RACE CONDITION!

# ✅ RIGHT — use asyncio.Lock for read-modify-write
lock = asyncio.Lock()
results = []

async def fetch_and_store(url):
    data = await fetch(url)
    async with lock:
        results.append(data)
        if len(results) > 10:
            results.clear()
```

### Best Practices Summary

```
1. Use async all the way down
   - Don't mix sync I/O in async handlers
   - Use async DB drivers (asyncpg, aiosqlite)
   - Use async HTTP clients (httpx, aiohttp)

2. Limit concurrency
   - Use Semaphore for external API calls
   - Use connection pools for databases
   - Don't create unbounded tasks

3. Offload CPU work
   - Use ProcessPoolExecutor for CPU-heavy tasks
   - Never do heavy computation in async handlers

4. Handle errors properly
   - Use asyncio.gather(return_exceptions=True) or TaskGroup
   - Always handle CancelledError in long-running tasks
   - Use timeouts: asyncio.wait_for()

5. Use proper async patterns
   - async with for resource management
   - async for for streaming data
   - TaskGroup (3.11+) for structured concurrency

6. Deploy correctly
   - Use Uvicorn + Gunicorn for production
   - Set appropriate worker count: 2 × CPU_CORES + 1
   - Use connection pooling for databases
   - Put behind Nginx for static files and rate limiting
```

### Quick Reference: Sync → Async Library Mapping

```
┌─────────────────────┬─────────────────────────────┐
│ Sync Library        │ Async Alternative            │
├─────────────────────┼─────────────────────────────┤
│ requests            │ httpx / aiohttp              │
│ psycopg2            │ asyncpg / psycopg3 (async)   │
│ pymongo             │ motor                        │
│ redis-py            │ redis.asyncio (aioredis)     │
│ sqlite3             │ aiosqlite                    │
│ open() (file I/O)   │ aiofiles                     │
│ time.sleep()        │ asyncio.sleep()              │
│ threading.Lock      │ asyncio.Lock                 │
│ queue.Queue         │ asyncio.Queue                │
│ subprocess          │ asyncio.create_subprocess    │
│ SQLAlchemy (sync)   │ SQLAlchemy async (2.0+)      │
│ Django ORM          │ Django async ORM / tortoise   │
│ Flask               │ FastAPI / Starlette          │
│ Celery              │ arq / taskiq                 │
└─────────────────────┴─────────────────────────────┘
```

---

## Summary Cheatsheet

```python
import asyncio

# 1. Define coroutine
async def my_task(name, delay):
    await asyncio.sleep(delay)
    return f"{name} done"

# 2. Run concurrently
async def main():
    results = await asyncio.gather(
        my_task("A", 1),
        my_task("B", 2),
        my_task("C", 3),
    )
    # Total: 3s (not 6s)

# 3. Start the event loop
asyncio.run(main())
```

```
Key Takeaways:
━━━━━━━━━━━━━
• async/await is for I/O-bound concurrency on a SINGLE thread
• The event loop switches between coroutines at 'await' points
• Use multiprocessing for CPU-bound work
• FastAPI: async def for async code, def for sync code
• NEVER block the event loop with sync I/O in async handlers
• Use connection pools, semaphores, and proper error handling
• Deploy with Uvicorn + Gunicorn behind Nginx
```
