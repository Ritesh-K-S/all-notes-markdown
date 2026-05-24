# Chapter 3.12: Concurrency Control & Locking — Preventing Double Execution

> **Level**: ⭐⭐⭐ Advanced  
> **What you'll learn**: How to ensure a function, task, or agent runs ONLY ONCE at a time — even when multiple users or requests try to trigger it simultaneously. Covers in-process locks, cross-process locks, and the exact patterns you need for single-server applications.

---

## 🧠 Real-Life Analogy: The Recording Studio

```
    Imagine a recording studio with ONE microphone booth:
    
    🎙️ Singer A enters the booth → RED light turns ON
    🎙️ Singer B arrives → sees RED light → WAITS outside
    🎙️ Singer C arrives → sees RED light → WAITS outside
    
    Singer A finishes → RED light turns OFF → GREEN light ON
    Singer B enters → RED light turns ON again
    
    NOBODY can enter while someone is recording!
    
    
    Now imagine 3 booths (Agent 1, Agent 2, Agent 3):
    
    SCENARIO 1: Per-Booth Lock (Function-Level Lock)
    ─────────────────────────────────────────────────
    🎙️ Booth 1: Singer A recording → Booth 1 LOCKED
    🎙️ Booth 2: Singer B can enter → Booth 2 OPEN ✅
    🎙️ Booth 3: Singer C can enter → Booth 3 OPEN ✅
    
    Another singer wants Booth 1? WAIT! ❌
    But Booth 2 and 3 are independent.
    
    
    SCENARIO 2: Studio-Wide Lock (Global Lock)
    ─────────────────────────────────────────────────
    🎙️ Singer A enters ANY booth → ENTIRE STUDIO LOCKED 🔒
    🎙️ Singer B arrives → "Sorry, studio in use" ❌
    🎙️ Singer C arrives → "Sorry, studio in use" ❌
    
    Nobody can use ANY booth until Singer A finishes!
    This is a GLOBAL LOCK across all booths.
```

---

## 📖 The Core Problem: Why Do We Need Execution Guards?

```
    THE PROBLEM:
    ═══════════════════════════════════════════════════
    
    You have a Python application with 3 AI Agents:
    
    ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
    │   Agent 1      │  │   Agent 2      │  │   Agent 3      │
    │  (Data Sync)   │  │  (Report Gen)  │  │  (Email Send)  │
    │                │  │                │  │                │
    │  Takes 5 min   │  │  Takes 10 min  │  │  Takes 2 min   │
    └────────────────┘  └────────────────┘  └────────────────┘
    
    
    WITHOUT any lock:
    ─────────────────
    User A: Triggers Agent 1 at 10:00 AM
    User B: Triggers Agent 1 at 10:01 AM (Agent 1 still running!)
    
    Now TWO copies of Agent 1 are running simultaneously!
    
    Problems:
    • Duplicate data synced (corrupted database)
    • Double emails sent to customers
    • Race conditions on shared files
    • Resource exhaustion (memory, API rate limits)
    • Inconsistent reports with partial data
    
    
    WHAT WE WANT:
    ─────────────
    
    Case 1: Function-Level Lock (per agent)
    ┌──────────────────────────────────────────────────────┐
    │  User A triggers Agent 1 → RUNNING ✅               │
    │  User B triggers Agent 1 → REJECTED ❌ (already     │
    │                              running)                │
    │  User C triggers Agent 2 → RUNNING ✅ (different    │
    │                              agent, no conflict)     │
    └──────────────────────────────────────────────────────┘
    
    Case 2: Global Lock (all agents)
    ┌──────────────────────────────────────────────────────┐
    │  User A triggers Agent 1 → RUNNING ✅               │
    │  User B triggers Agent 1 → REJECTED ❌              │
    │  User C triggers Agent 2 → REJECTED ❌ (global      │
    │                              lock active!)           │
    │  User D triggers Agent 3 → REJECTED ❌              │
    │                                                      │
    │  Agent 1 finishes → lock released                    │
    │  User C triggers Agent 2 → RUNNING ✅               │
    └──────────────────────────────────────────────────────┘
```

---

## 🔧 Approach 1: In-Process Locks (Single Process, Multiple Threads)

When your application runs as a **single process** (e.g., one Flask/FastAPI server), use Python's built-in threading locks.

### threading.Lock — The Simplest Mutex

```
    HOW threading.Lock WORKS:
    ═══════════════════════════════════════════════════
    
    Lock has 2 states: LOCKED or UNLOCKED
    
    Thread 1: lock.acquire() → state = LOCKED ✅
    Thread 2: lock.acquire() → BLOCKS (waits...) ⏳
    Thread 3: lock.acquire() → BLOCKS (waits...) ⏳
    
    Thread 1: lock.release() → state = UNLOCKED
    Thread 2: (wakes up!) → lock.acquire() → LOCKED ✅
    Thread 3: still waiting... ⏳
    
    
    IMPORTANT: Lock = BLOCKING wait (threads queue up)
    
    For "reject immediately" behavior, use lock.acquire(blocking=False):
    
    Thread 1: lock.acquire() → True ✅
    Thread 2: lock.acquire(blocking=False) → False ❌ (instant reject!)
```

### Python Code: Function-Level Lock (Per Agent)

```python
"""
Function-Level Lock: Each agent has its OWN lock.
Agent 1 running doesn't block Agent 2 or Agent 3.
Only prevents the SAME agent from running twice.
"""
import threading
import time
from functools import wraps

# ── Each agent gets its own lock ──
agent_locks = {
    "agent_1": threading.Lock(),
    "agent_2": threading.Lock(),
    "agent_3": threading.Lock(),
}

def exclusive_execution(agent_name: str):
    """Decorator: ensures only one execution of this agent at a time."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            lock = agent_locks[agent_name]
            
            # Try to acquire WITHOUT blocking
            acquired = lock.acquire(blocking=False)
            if not acquired:
                print(f"❌ {agent_name} is already running! Skipping.")
                return None  # or raise an exception
            
            try:
                print(f"✅ {agent_name} started by {kwargs.get('user', 'unknown')}")
                return func(*args, **kwargs)
            finally:
                lock.release()
                print(f"🔓 {agent_name} finished, lock released.")
        
        return wrapper
    return decorator


# ── Define agents ──

@exclusive_execution("agent_1")
def agent_data_sync(user="unknown"):
    """Agent 1: Syncs data from external API. Takes ~5 minutes."""
    print("  Syncing data from external API...")
    time.sleep(5)  # Simulating long work
    print("  Data sync complete!")
    return {"status": "success", "records": 1500}


@exclusive_execution("agent_2")
def agent_report_gen(user="unknown"):
    """Agent 2: Generates analytics report. Takes ~10 minutes."""
    print("  Generating analytics report...")
    time.sleep(10)
    print("  Report generated!")
    return {"status": "success", "report": "Q4_2024.pdf"}


@exclusive_execution("agent_3")
def agent_email_sender(user="unknown"):
    """Agent 3: Sends batch emails. Takes ~2 minutes."""
    print("  Sending batch emails...")
    time.sleep(2)
    print("  Emails sent!")
    return {"status": "success", "emails_sent": 350}


# ── Simulate concurrent execution ──

t1 = threading.Thread(target=agent_data_sync, kwargs={"user": "Alice"})
t2 = threading.Thread(target=agent_data_sync, kwargs={"user": "Bob"})    # Same agent!
t3 = threading.Thread(target=agent_report_gen, kwargs={"user": "Charlie"})  # Different agent

t1.start()
time.sleep(0.1)  # Small delay so t1 acquires first
t2.start()       # Will be REJECTED (agent_1 already running)
t3.start()       # Will SUCCEED (different agent)

t1.join(); t2.join(); t3.join()

# OUTPUT:
# ✅ agent_1 started by Alice
#   Syncing data from external API...
# ❌ agent_1 is already running! Skipping.        ← Bob rejected!
# ✅ agent_2 started by Charlie                    ← Different agent, OK!
#   Generating analytics report...
#   Data sync complete!
# 🔓 agent_1 finished, lock released.
#   Report generated!
# 🔓 agent_2 finished, lock released.
```

### Python Code: Global Lock (All Agents)

```python
"""
Global Lock: If ANY agent is running, NO other agent can start.
One lock for all agents — only one agent runs at a time.
"""
import threading
import time
from functools import wraps

# ── ONE lock for ALL agents ──
global_agent_lock = threading.Lock()

def global_exclusive_execution(func):
    """Decorator: ensures only one agent (of any type) runs at a time."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        acquired = global_agent_lock.acquire(blocking=False)
        if not acquired:
            print(f"❌ Another agent is already running! "
                  f"Cannot start {func.__name__}.")
            return None
        
        try:
            print(f"✅ {func.__name__} started (global lock acquired)")
            return func(*args, **kwargs)
        finally:
            global_agent_lock.release()
            print(f"🔓 {func.__name__} finished (global lock released)")
    
    return wrapper


@global_exclusive_execution
def agent_data_sync():
    print("  Syncing data...")
    time.sleep(5)

@global_exclusive_execution
def agent_report_gen():
    print("  Generating report...")
    time.sleep(10)

@global_exclusive_execution
def agent_email_sender():
    print("  Sending emails...")
    time.sleep(2)


# ── Simulate ──

t1 = threading.Thread(target=agent_data_sync)
t2 = threading.Thread(target=agent_report_gen)     # Different agent, STILL BLOCKED!
t3 = threading.Thread(target=agent_email_sender)    # Different agent, STILL BLOCKED!

t1.start()
time.sleep(0.1)
t2.start()  # ❌ Rejected! Global lock held by agent_data_sync
t3.start()  # ❌ Rejected! Global lock held by agent_data_sync

t1.join(); t2.join(); t3.join()

# OUTPUT:
# ✅ agent_data_sync started (global lock acquired)
#   Syncing data...
# ❌ Another agent is already running! Cannot start agent_report_gen.
# ❌ Another agent is already running! Cannot start agent_email_sender.
#   (Data sync finishes)
# 🔓 agent_data_sync finished (global lock released)
```

---

## 🔧 Approach 2: asyncio.Lock (For Async Applications)

If your application uses `async/await` (FastAPI, aiohttp), use `asyncio.Lock` instead of `threading.Lock`.

```python
"""
Async version: For FastAPI, aiohttp, or any async framework.
asyncio.Lock works within a SINGLE event loop (single process).
"""
import asyncio
from functools import wraps

# ── Per-agent locks (async version) ──
agent_locks = {
    "data_sync": asyncio.Lock(),
    "report_gen": asyncio.Lock(),
    "email_sender": asyncio.Lock(),
}

def async_exclusive(agent_name: str):
    """Decorator: async version of exclusive execution."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            lock = agent_locks[agent_name]
            
            if lock.locked():
                return {"error": f"{agent_name} is already running!"}
            
            async with lock:
                return await func(*args, **kwargs)
        
        return wrapper
    return decorator


@async_exclusive("data_sync")
async def agent_data_sync():
    await asyncio.sleep(5)
    return {"status": "success"}


# ── FastAPI example ──
# from fastapi import FastAPI
# app = FastAPI()
#
# @app.post("/run/agent1")
# async def run_agent1():
#     result = await agent_data_sync()
#     if "error" in result:
#         raise HTTPException(status_code=409, detail=result["error"])
#     return result


# ── Global lock (async version) ──
global_lock = asyncio.Lock()

def async_global_exclusive(func):
    """Only one agent of ANY type can run at a time (async)."""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        if global_lock.locked():
            return {"error": "Another agent is already running!"}
        
        async with global_lock:
            return await func(*args, **kwargs)
    
    return wrapper
```

---

## 🔧 Approach 3: Cross-Process Locks (Multiple Workers / Gunicorn)

`threading.Lock` and `asyncio.Lock` only work within a **single process**. If your app runs with multiple workers (Gunicorn with 4 workers, or multiple instances), each worker is a separate process with its own memory — they **don't share locks**.

```
    THE MULTI-WORKER PROBLEM:
    ═══════════════════════════════════════════════════
    
    Gunicorn with 4 workers:
    
    ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
    │ Worker 1  │  │ Worker 2  │  │ Worker 3  │  │ Worker 4  │
    │           │  │           │  │           │  │           │
    │ Own Lock  │  │ Own Lock  │  │ Own Lock  │  │ Own Lock  │
    │ (memory)  │  │ (memory)  │  │ (memory)  │  │ (memory)  │
    └───────────┘  └───────────┘  └───────────┘  └───────────┘
         │              │              │              │
         └──────────────┼──────────────┼──────────────┘
                        │
        Each worker has its OWN threading.Lock!
        Worker 1's lock does NOT block Worker 2!
        
        Request 1 → Worker 1 → acquires lock → Agent 1 running
        Request 2 → Worker 3 → acquires lock → Agent 1 ALSO running! 💀
        
        threading.Lock is USELESS here!
```

### Solution 3A: File-Based Lock (OS-Level)

```python
"""
File-based locking: Works across multiple processes on the SAME machine.
Uses OS-level file locks (fcntl on Linux/Mac, msvcrt on Windows).
"""
import os
import sys
import time
from contextlib import contextmanager
from functools import wraps

class FileLock:
    """Cross-process lock using file system.
    Works across Gunicorn workers, multiple scripts, etc.
    """
    
    def __init__(self, lock_file: str):
        self.lock_file = lock_file
        self._fd = None
    
    def acquire(self, blocking: bool = False) -> bool:
        """Try to acquire the lock."""
        try:
            # Create lock file if it doesn't exist
            self._fd = open(self.lock_file, 'w')
            
            if sys.platform == 'win32':
                # Windows
                import msvcrt
                try:
                    msvcrt.locking(self._fd.fileno(), msvcrt.LK_NBLCK, 1)
                    return True
                except IOError:
                    self._fd.close()
                    return False
            else:
                # Linux / macOS
                import fcntl
                flags = fcntl.LOCK_EX
                if not blocking:
                    flags |= fcntl.LOCK_NB
                try:
                    fcntl.flock(self._fd.fileno(), flags)
                    # Write PID for debugging
                    self._fd.write(str(os.getpid()))
                    self._fd.flush()
                    return True
                except BlockingIOError:
                    self._fd.close()
                    return False
        except Exception:
            if self._fd:
                self._fd.close()
            return False
    
    def release(self):
        """Release the lock."""
        if self._fd:
            try:
                if sys.platform == 'win32':
                    import msvcrt
                    msvcrt.locking(self._fd.fileno(), msvcrt.LK_UNLCK, 1)
                else:
                    import fcntl
                    fcntl.flock(self._fd.fileno(), fcntl.LOCK_UN)
            finally:
                self._fd.close()
                # Clean up lock file
                try:
                    os.remove(self.lock_file)
                except OSError:
                    pass


def file_lock_exclusive(lock_path: str):
    """Decorator: cross-process exclusive execution using file locks."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            lock = FileLock(lock_path)
            
            if not lock.acquire(blocking=False):
                print(f"❌ {func.__name__} is already running "
                      f"(lock file: {lock_path})")
                return None
            
            try:
                return func(*args, **kwargs)
            finally:
                lock.release()
        
        return wrapper
    return decorator


# ── Usage ──

@file_lock_exclusive("/tmp/agent_data_sync.lock")
def agent_data_sync():
    print("Syncing data...")
    time.sleep(300)  # 5 minutes

@file_lock_exclusive("/tmp/agent_report_gen.lock")
def agent_report_gen():
    print("Generating report...")
    time.sleep(600)  # 10 minutes

# For GLOBAL lock — all agents use the SAME lock file:
@file_lock_exclusive("/tmp/all_agents.lock")
def agent_data_sync_global():
    print("Syncing data (global lock)...")
    time.sleep(300)

@file_lock_exclusive("/tmp/all_agents.lock")
def agent_report_gen_global():
    print("Generating report (global lock)...")
    time.sleep(600)
```

### Solution 3B: Database-Based Lock

```python
"""
Database-based locking: Works across multiple servers.
Uses a database row as a lock — works even with distributed deployments.
"""
import datetime
import uuid
from contextlib import contextmanager

# Using SQLAlchemy (works with PostgreSQL, MySQL, SQLite)
from sqlalchemy import create_engine, Column, String, DateTime, Boolean
from sqlalchemy.orm import declarative_base, Session
from sqlalchemy.dialects.postgresql import insert as pg_insert

Base = declarative_base()

class ExecutionLock(Base):
    __tablename__ = "execution_locks"
    
    lock_name = Column(String, primary_key=True)    # "agent_1" or "global"
    owner_id = Column(String, nullable=False)         # UUID of lock holder
    acquired_at = Column(DateTime, nullable=False)
    expires_at = Column(DateTime, nullable=False)      # Auto-expire (deadlock safety)
    is_locked = Column(Boolean, default=True)


class DBLockManager:
    """Database-based lock manager for cross-process/cross-server locking."""
    
    def __init__(self, db_url: str):
        self.engine = create_engine(db_url)
        Base.metadata.create_all(self.engine)
    
    def acquire(self, lock_name: str, ttl_seconds: int = 600) -> str | None:
        """Try to acquire a lock. Returns owner_id if successful, None if not."""
        owner_id = str(uuid.uuid4())
        now = datetime.datetime.utcnow()
        expires_at = now + datetime.timedelta(seconds=ttl_seconds)
        
        with Session(self.engine) as session:
            # Check if lock exists and is still valid
            existing = session.query(ExecutionLock).filter_by(
                lock_name=lock_name, is_locked=True
            ).first()
            
            if existing:
                if existing.expires_at > now:
                    # Lock is held and not expired
                    return None
                else:
                    # Lock expired — clean it up
                    existing.is_locked = False
                    session.commit()
            
            # Acquire the lock
            lock = ExecutionLock(
                lock_name=lock_name,
                owner_id=owner_id,
                acquired_at=now,
                expires_at=expires_at,
                is_locked=True,
            )
            session.merge(lock)
            session.commit()
            return owner_id
    
    def release(self, lock_name: str, owner_id: str) -> bool:
        """Release a lock. Only the owner can release it."""
        with Session(self.engine) as session:
            lock = session.query(ExecutionLock).filter_by(
                lock_name=lock_name, owner_id=owner_id, is_locked=True
            ).first()
            
            if lock:
                lock.is_locked = False
                session.commit()
                return True
            return False


# ── Usage with decorator ──

db_lock_manager = DBLockManager("postgresql://localhost/myapp")

def db_exclusive(lock_name: str, ttl: int = 600):
    """Decorator: DB-based exclusive execution."""
    def decorator(func):
        def wrapper(*args, **kwargs):
            owner_id = db_lock_manager.acquire(lock_name, ttl)
            if not owner_id:
                return {"error": f"{lock_name} is already running!"}
            
            try:
                return func(*args, **kwargs)
            finally:
                db_lock_manager.release(lock_name, owner_id)
        return wrapper
    return decorator


@db_exclusive("agent_data_sync", ttl=600)
def agent_data_sync():
    """This lock works across multiple servers!"""
    print("Syncing data...")
```

### Solution 3C: Redis-Based Lock (Single Server, Fast)

```python
"""
Redis-based locking: Fast, works across processes.
Best when you already have Redis in your stack.
For multi-server distributed locks, see Chapter 13.8.
"""
import redis
import uuid
import time
from functools import wraps

redis_client = redis.Redis(host="localhost", port=6379, db=0)

def redis_exclusive(lock_name: str, ttl_seconds: int = 600):
    """Decorator: Redis-based exclusive execution."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            lock_key = f"exec_lock:{lock_name}"
            owner_id = str(uuid.uuid4())
            
            # Try to acquire (NX = only if not exists, EX = auto-expire)
            acquired = redis_client.set(
                lock_key, owner_id, nx=True, ex=ttl_seconds
            )
            
            if not acquired:
                return {"error": f"{lock_name} is already running!"}
            
            try:
                return func(*args, **kwargs)
            finally:
                # Release only if we are the owner (Lua for atomicity)
                lua_script = """
                if redis.call("GET", KEYS[1]) == ARGV[1] then
                    return redis.call("DEL", KEYS[1])
                else
                    return 0
                end
                """
                redis_client.eval(lua_script, 1, lock_key, owner_id)
        
        return wrapper
    return decorator


# ── Per-agent lock ──
@redis_exclusive("agent_data_sync")
def agent_data_sync():
    print("Syncing data...")
    time.sleep(300)

# ── Global lock (all agents use same lock name) ──
@redis_exclusive("global_agent_lock")
def agent_data_sync_global():
    print("Syncing data (global)...")

@redis_exclusive("global_agent_lock")
def agent_report_gen_global():
    print("Generating report (global)...")
```

---

## 📊 Complete Comparison: Which Lock to Use?

```
    ┌──────────────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
    │                      │ threading    │ File Lock    │ Redis Lock   │ DB Lock      │
    │                      │ .Lock()      │ (OS-level)   │              │              │
    ├──────────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
    │ Scope                │ Single       │ Single       │ Single       │ Multiple     │
    │                      │ process      │ machine      │ machine /    │ machines     │
    │                      │              │              │ cluster      │              │
    ├──────────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
    │ Speed                │ ~1 μs        │ ~100 μs      │ ~1 ms        │ ~5-10 ms     │
    ├──────────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
    │ Works across         │ ❌ No        │ ✅ Yes       │ ✅ Yes       │ ✅ Yes       │
    │ processes?           │              │              │              │              │
    ├──────────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
    │ Works across         │ ❌ No        │ ❌ No        │ ✅ Yes       │ ✅ Yes       │
    │ servers?             │              │              │ (if shared   │              │
    │                      │              │              │  Redis)      │              │
    ├──────────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
    │ Auto-expire          │ ❌ No        │ ❌ No        │ ✅ Yes       │ ✅ Yes       │
    │ (TTL)?               │ (manual)     │ (OS handles  │ (built-in)   │ (manual      │
    │                      │              │  on crash)   │              │  logic)      │
    ├──────────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
    │ Survives crash?      │ ❌ Lost      │ ✅ OS        │ ✅ TTL       │ ✅ TTL       │
    │                      │              │ releases     │ expires      │ expires      │
    ├──────────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
    │ Extra dependency?    │ None         │ None         │ Redis        │ Database     │
    ├──────────────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
    │ Best for             │ Single app,  │ Multi-worker │ Fast locks,  │ Already have │
    │                      │ simple case  │ same machine │ real-time    │ DB, need     │
    │                      │              │ (Gunicorn)   │ apps         │ audit trail  │
    └──────────────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
    
    
    DECISION TREE:
    ══════════════
    
    Is your app a SINGLE process?
    ├── YES → Use threading.Lock (simplest)
    │         or asyncio.Lock (if async)
    │
    └── NO → Multiple processes on SAME machine?
             ├── YES → Use File Lock (no extra dependency)
             │         or Redis Lock (if you have Redis)
             │
             └── NO → Multiple SERVERS?
                      ├── YES → Use Redis Lock or DB Lock
                      │         For strong guarantees:
                      │         → See Chapter 13.8 (Distributed Locking)
                      │
                      └── Use DB Lock (works everywhere)
```

---

## 🏗️ Full Production Example: Agent Execution Manager

A complete, production-ready implementation combining per-agent and global lock modes:

```python
"""
Production-ready Agent Execution Manager.
Supports both per-agent locks and global locks.
Uses Redis for cross-process safety.

Usage:
    manager = AgentExecutionManager(redis_url="redis://localhost:6379")

    # Per-agent lock (only blocks same agent)
    @manager.exclusive(mode="per_agent")
    def agent_data_sync():
        ...

    # Global lock (blocks ALL agents)
    @manager.exclusive(mode="global")
    def agent_critical_migration():
        ...
"""
import redis
import uuid
import time
import logging
from functools import wraps
from typing import Literal
from dataclasses import dataclass
from datetime import datetime

logger = logging.getLogger(__name__)


@dataclass
class LockInfo:
    agent_name: str
    owner_id: str
    started_at: str
    mode: str


class AgentExecutionManager:
    """Manages exclusive execution of agents with per-agent or global locks."""
    
    GLOBAL_LOCK_KEY = "agent_exec:__global__"
    LOCK_PREFIX = "agent_exec:"
    
    def __init__(self, redis_url: str = "redis://localhost:6379",
                 default_ttl: int = 3600):
        self.redis = redis.from_url(redis_url)
        self.default_ttl = default_ttl
        
        # Lua script: acquire lock only if global lock AND agent lock are free
        self._acquire_script = self.redis.register_script("""
            local lock_key = KEYS[1]
            local global_key = KEYS[2]
            local owner_id = ARGV[1]
            local ttl = tonumber(ARGV[2])
            local mode = ARGV[3]
            local agent_info = ARGV[4]
            
            -- Check if global lock is held (by someone else)
            local global_owner = redis.call("GET", global_key)
            if global_owner and global_owner ~= owner_id then
                return "BLOCKED_BY_GLOBAL"
            end
            
            -- If mode is "global", check if ANY agent lock is held
            if mode == "global" then
                local keys = redis.call("KEYS", "agent_exec:*")
                for _, k in ipairs(keys) do
                    if k ~= global_key and k ~= lock_key then
                        local v = redis.call("GET", k)
                        if v then
                            return "BLOCKED_BY_AGENT:" .. k
                        end
                    end
                end
            end
            
            -- Try to acquire the agent-specific lock
            local result = redis.call("SET", lock_key, agent_info, "NX", "EX", ttl)
            if not result then
                return "ALREADY_RUNNING"
            end
            
            -- If global mode, also set the global lock
            if mode == "global" then
                redis.call("SET", global_key, agent_info, "NX", "EX", ttl)
            end
            
            return "ACQUIRED"
        """)
        
        # Lua script: release lock only if we own it
        self._release_script = self.redis.register_script("""
            local lock_key = KEYS[1]
            local global_key = KEYS[2]
            local owner_id = ARGV[1]
            
            local lock_val = redis.call("GET", lock_key)
            if lock_val then
                -- Check owner_id is in the stored value
                if string.find(lock_val, owner_id) then
                    redis.call("DEL", lock_key)
                    -- Also release global lock if we hold it
                    local global_val = redis.call("GET", global_key)
                    if global_val and string.find(global_val, owner_id) then
                        redis.call("DEL", global_key)
                    end
                    return 1
                end
            end
            return 0
        """)
    
    def exclusive(self, mode: Literal["per_agent", "global"] = "per_agent",
                  ttl: int | None = None):
        """
        Decorator for exclusive agent execution.
        
        Args:
            mode: "per_agent" — only blocks same agent from running again
                  "global"    — blocks ALL agents while this one runs
            ttl:  Max lock duration in seconds (auto-expires for safety)
        """
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                agent_name = func.__name__
                owner_id = str(uuid.uuid4())
                lock_key = f"{self.LOCK_PREFIX}{agent_name}"
                lock_ttl = ttl or self.default_ttl
                
                agent_info = f"{owner_id}|{agent_name}|{datetime.utcnow().isoformat()}"
                
                # Try to acquire
                result = self._acquire_script(
                    keys=[lock_key, self.GLOBAL_LOCK_KEY],
                    args=[owner_id, lock_ttl, mode, agent_info]
                )
                result = result.decode() if isinstance(result, bytes) else result
                
                if result != "ACQUIRED":
                    logger.warning(f"Cannot start {agent_name}: {result}")
                    return {
                        "error": f"Agent '{agent_name}' cannot start",
                        "reason": result,
                        "agent": agent_name,
                    }
                
                logger.info(f"Agent '{agent_name}' started (mode={mode}, "
                           f"owner={owner_id[:8]})")
                
                try:
                    return func(*args, **kwargs)
                finally:
                    self._release_script(
                        keys=[lock_key, self.GLOBAL_LOCK_KEY],
                        args=[owner_id]
                    )
                    logger.info(f"Agent '{agent_name}' finished, lock released")
            
            return wrapper
        return decorator
    
    def get_running_agents(self) -> list[LockInfo]:
        """Get list of currently running agents (for admin dashboard)."""
        agents = []
        for key in self.redis.scan_iter(f"{self.LOCK_PREFIX}*"):
            if key == self.GLOBAL_LOCK_KEY.encode():
                continue
            val = self.redis.get(key)
            if val:
                parts = val.decode().split("|")
                if len(parts) >= 3:
                    agents.append(LockInfo(
                        agent_name=parts[1],
                        owner_id=parts[0],
                        started_at=parts[2],
                        mode="check_global",
                    ))
        return agents
    
    def force_release(self, agent_name: str):
        """Admin: force-release a stuck agent lock."""
        lock_key = f"{self.LOCK_PREFIX}{agent_name}"
        self.redis.delete(lock_key)
        self.redis.delete(self.GLOBAL_LOCK_KEY)
        logger.warning(f"Force-released lock for {agent_name}")


# ══════════════════════════════════════════════════════
# USAGE EXAMPLE
# ══════════════════════════════════════════════════════

manager = AgentExecutionManager(redis_url="redis://localhost:6379")

# ── Per-agent lock: Only blocks duplicate runs of the same agent ──

@manager.exclusive(mode="per_agent")
def agent_data_sync():
    """If agent_data_sync is running, another call is rejected.
    But agent_report_gen can run simultaneously."""
    time.sleep(300)
    return {"synced": 1500}


@manager.exclusive(mode="per_agent")
def agent_report_gen():
    """Independent from agent_data_sync."""
    time.sleep(600)
    return {"report": "generated"}


# ── Global lock: If this runs, NO other agent can start ──

@manager.exclusive(mode="global", ttl=7200)
def agent_critical_migration():
    """Database migration — nothing else should run!"""
    time.sleep(3600)
    return {"migrated": True}
```

---

## 📊 Visualization: Lock Modes Side by Side

```
    PER-AGENT LOCK (mode="per_agent"):
    ═══════════════════════════════════════════════════
    
    Timeline ──────────────────────────────────────────▶
    
    Agent 1: ████████████████████░░░░░░░░░░░░░░░░░░░░
             ↑ start            ↑ end
    
    Agent 1: ❌ ❌ ❌ ❌ ❌ ❌ ❌ ████████████████████
             blocked while ↑     ↑ now allowed!
             Agent 1 runs        starts after first finishes
    
    Agent 2: ░░░████████████████░░░░░░░░░░░░░░░░░░░░░
                ↑ can start!     ↑ end
                (different agent, not blocked)
    
    Agent 3: ░░░░░░████████░░░░░░░░░░░░░░░░░░░░░░░░░
                   ↑ can start!  ↑ end
                   (independent)
    
    
    GLOBAL LOCK (mode="global"):
    ═══════════════════════════════════════════════════
    
    Timeline ──────────────────────────────────────────▶
    
    Agent 1: ████████████████████░░░░░░░░░░░░░░░░░░░░
             ↑ start (GLOBAL)    ↑ end → all unlocked
    
    Agent 2: ❌ ❌ ❌ ❌ ❌ ❌ ❌ ░░████████████████░░
             blocked!            starts AFTER Agent 1 finishes
    
    Agent 3: ❌ ❌ ❌ ❌ ❌ ❌ ❌ ░░░░░░░░░░░░░░░░░░░░
             blocked!            can start after Agent 2
                                 (if Agent 2 is also global)
    
    
    KEY DIFFERENCE:
    ┌─────────────────────────────────────────────────┐
    │ Per-Agent: "I'm using Booth 1, you can use 2"  │
    │ Global:    "I'm in the studio, NOBODY comes in" │
    └─────────────────────────────────────────────────┘
```

---

## ⚠️ Common Pitfalls & How to Avoid Them

```
    PITFALL 1: DEADLOCK (Forgot to Release)
    ═══════════════════════════════════════════════════
    
    ❌ BAD:
    lock.acquire()
    do_work()         # What if this CRASHES?
    lock.release()    # Never reached! Lock stuck forever!
    
    ✅ GOOD:
    lock.acquire()
    try:
        do_work()
    finally:
        lock.release()  # ALWAYS runs, even on crash!
    
    ✅ BEST (Python context manager):
    with lock:
        do_work()     # Auto-released when block exits!
    
    
    PITFALL 2: NO TTL (Lock Lives Forever After Crash)
    ═══════════════════════════════════════════════════
    
    Process acquires Redis lock → process CRASHES → lock stays!
    Nobody can acquire it → system stuck forever!
    
    ✅ SOLUTION: Always set TTL (auto-expiry):
    redis.set("lock:agent1", owner, NX=True, EX=3600)
    │                                         │
    │                                         └── Expires in 1 hour
    └── Even if process crashes, lock auto-releases!
    
    
    PITFALL 3: WRONG SCOPE (threading.Lock with Gunicorn)
    ═══════════════════════════════════════════════════
    
    threading.Lock only works within ONE process!
    Gunicorn spawns MULTIPLE processes (workers).
    Each worker has its OWN threading.Lock — they don't know
    about each other!
    
    ✅ SOLUTION: Use file lock, Redis lock, or DB lock.
    
    
    PITFALL 4: BLOCKING vs NON-BLOCKING
    ═══════════════════════════════════════════════════
    
    lock.acquire()                    # BLOCKS forever! Thread hangs!
    lock.acquire(blocking=False)      # Returns False immediately ✅
    lock.acquire(timeout=5)           # Waits max 5 seconds, then False
    
    For agents: use NON-BLOCKING (reject immediately).
    For queued work: use BLOCKING with TIMEOUT.
    
    
    PITFALL 5: NOT HANDLING THE "REJECTED" CASE
    ═══════════════════════════════════════════════════
    
    What should happen when execution is rejected?
    
    Option A: Return error to the user
    → "Agent is already running, please try later"
    
    Option B: Queue it for later execution
    → Add to a task queue (Celery, RQ) — runs when lock is free
    
    Option C: Return the status of the running execution
    → "Agent started at 10:00 AM, estimated completion: 10:05 AM"
```

---

## 🔗 Connection to Other Chapters

```
    THIS CHAPTER (3.12) — Application-Level Locks:
    Single server, same process or multiple workers.
    Python threading.Lock, asyncio.Lock, file locks, Redis/DB locks.
    
         │
         │ When you scale to MULTIPLE SERVERS:
         ▼
    
    CHAPTER 13.8 — Distributed Locking:
    Multiple servers, network partitions, consensus.
    Redlock algorithm, ZooKeeper, etcd, fencing tokens.
    Much harder problem! (see CAP theorem, Chapter 13.2)
    
         │
         │ For limiting concurrent ACCESS (not blocking entirely):
         ▼
    
    CHAPTER 12.4 — Bulkhead Pattern:
    Limit how many requests can access a service simultaneously.
    Uses semaphores (allow N concurrent, not just 1).
    
         │
         │ For rate limiting (requests per second):
         ▼
    
    CHAPTER 8.3 — Rate Limiting & Throttling:
    Limit how many requests per time window (not about exclusion).
    Token bucket, sliding window algorithms.
```

---

## 📝 Summary

```
    WHAT YOU LEARNED:
    ═══════════════════════════════════════════════════
    
    1. Function-Level Lock: Prevent same function/agent
       from running concurrently. Each agent has its own lock.
    
    2. Global Lock: Prevent ANY agent from running while
       one is active. All agents share one lock.
    
    3. In-Process Locks: threading.Lock (sync) or
       asyncio.Lock (async). Simple, fast, single-process only.
    
    4. Cross-Process Locks: File locks (same machine),
       Redis locks (same machine / cluster),
       DB locks (any number of servers).
    
    5. Always use TTL for external locks (Redis, DB)
       to prevent deadlocks from crashes.
    
    6. Always release locks in a finally block or
       use context managers (with statement).
    
    7. Choose blocking=False for agents (reject immediately)
       vs blocking=True for queued work (wait in line).
    
    
    QUICK REFERENCE:
    ═══════════════════════════════════════════════════
    
    ┌──────────────────────┬─────────────────────────────────┐
    │ Scenario             │ Use                             │
    ├──────────────────────┼─────────────────────────────────┤
    │ Single process,      │ threading.Lock (per function)   │
    │ per-agent lock       │ or asyncio.Lock (if async)      │
    ├──────────────────────┼─────────────────────────────────┤
    │ Single process,      │ One shared threading.Lock       │
    │ global lock          │ for all agents                  │
    ├──────────────────────┼─────────────────────────────────┤
    │ Multi-worker (same   │ File lock (no dependency) or    │
    │ machine), per-agent  │ Redis lock (if available)       │
    ├──────────────────────┼─────────────────────────────────┤
    │ Multi-worker (same   │ Single file lock / single Redis │
    │ machine), global     │ key for all agents              │
    ├──────────────────────┼─────────────────────────────────┤
    │ Multi-server,        │ Redis lock or DB lock           │
    │ per-agent            │ → Chapter 13.8 for guarantees   │
    ├──────────────────────┼─────────────────────────────────┤
    │ Multi-server,        │ Redis global key / DB lock      │
    │ global               │ → Chapter 13.8 for guarantees   │
    └──────────────────────┴─────────────────────────────────┘
```

---

> **Next**: [Chapter 13.8 — Distributed Locking (Redlock, ZooKeeper)](../13-distributed-systems/08-distributed-locking.md) — When your agents run on MULTIPLE servers and you need locks across the network.
