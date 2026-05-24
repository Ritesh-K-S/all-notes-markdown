# Concurrency & Multithreading in LLD

Concurrency is critical in LLD when designing systems that handle multiple users, shared resources, or parallel operations.

---

## 1. Thread Basics

### Creating Threads in Java

```java
// Way 1: Extend Thread class
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}

// Way 2: Implement Runnable (Preferred)
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}

// Way 3: Lambda (Java 8+)
Thread t = new Thread(() -> {
    System.out.println("Running in: " + Thread.currentThread().getName());
});

// Starting threads
new MyThread().start();
new Thread(new MyRunnable()).start();
t.start();
```

### Thread Lifecycle

```
NEW → (start()) → RUNNABLE → (scheduler) → RUNNING
                      ↕
                   BLOCKED / WAITING / TIMED_WAITING
                      ↓
                  TERMINATED
```

| State | Description |
|-------|-------------|
| `NEW` | Created but not started |
| `RUNNABLE` | Ready to run, waiting for CPU |
| `RUNNING` | Currently executing |
| `BLOCKED` | Waiting to acquire a lock |
| `WAITING` | Waiting indefinitely (wait(), join()) |
| `TIMED_WAITING` | Waiting for a specific time (sleep(), wait(timeout)) |
| `TERMINATED` | Execution completed |

---

## 2. Thread Safety Problems

### Race Condition

When multiple threads access shared data concurrently and the result depends on the timing of execution.

```java
// NOT thread-safe — race condition
public class Counter {
    private int count = 0;

    public void increment() {
        count++;  // Read → Modify → Write (3 steps, not atomic)
    }

    public int getCount() {
        return count;
    }
}

// With 2 threads each doing 10000 increments:
// Expected: 20000
// Actual: Something less (e.g., 18543) — due to race condition
```

### How Race Condition Happens

```
Thread A: read count (0)
Thread B: read count (0)        ← Both read same value
Thread A: increment to 1
Thread B: increment to 1        ← Both write 1
Thread A: write count = 1
Thread B: write count = 1       ← One increment is lost!
```

---

## 3. Synchronization

### synchronized Keyword

```java
// Method-level synchronization
public class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;  // Only one thread can execute at a time
    }

    public synchronized int getCount() {
        return count;
    }
}

// Block-level synchronization (more granular)
public class Counter {
    private int count = 0;
    private final Object lock = new Object();

    public void increment() {
        synchronized (lock) {
            count++;
        }
    }
}
```

### Problems with synchronized
- **Performance** — Only one thread at a time (bottleneck)
- **Deadlock** — Two threads waiting for each other's lock
- **Starvation** — Some threads never get the lock

---

## 4. Volatile Keyword

`volatile` ensures that changes to a variable are **visible to all threads** immediately. It prevents caching the variable in thread-local memory.

```java
public class StatusFlag {
    private volatile boolean running = true;  // Visible to all threads

    public void stop() {
        running = false;  // Immediately visible to other threads
    }

    public void doWork() {
        while (running) {
            // Process work
        }
        System.out.println("Stopped");
    }
}
```

### volatile vs synchronized

| Feature | volatile | synchronized |
|---------|----------|-------------|
| Atomicity | No (only visibility) | Yes |
| Blocking | No | Yes |
| Use for | Flags, status variables | Compound operations |
| Performance | Better | Slower |

---

## 5. Atomic Classes

`java.util.concurrent.atomic` provides lock-free thread-safe operations.

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;

public class Counter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();  // Atomic — thread-safe without locks
    }

    public int getCount() {
        return count.get();
    }
}

// Other Atomic classes
AtomicLong atomicLong = new AtomicLong(0);
AtomicBoolean atomicBoolean = new AtomicBoolean(false);
AtomicReference<String> atomicRef = new AtomicReference<>("initial");

// Compare-And-Swap (CAS)
atomicRef.compareAndSet("initial", "updated");  // Only sets if current value matches
```

---

## 6. Locks (java.util.concurrent.locks)

### ReentrantLock

More flexible than `synchronized` — supports try-lock, timed lock, and fairness.

```java
import java.util.concurrent.locks.ReentrantLock;

public class BankAccount {
    private double balance;
    private final ReentrantLock lock = new ReentrantLock();

    public BankAccount(double balance) {
        this.balance = balance;
    }

    public void withdraw(double amount) {
        lock.lock();
        try {
            if (balance >= amount) {
                balance -= amount;
                System.out.println("Withdrawn: " + amount + ", Balance: " + balance);
            } else {
                System.out.println("Insufficient balance");
            }
        } finally {
            lock.unlock();  // ALWAYS unlock in finally
        }
    }

    // Try-lock — don't wait if lock is unavailable
    public boolean tryWithdraw(double amount) {
        if (lock.tryLock()) {
            try {
                if (balance >= amount) {
                    balance -= amount;
                    return true;
                }
            } finally {
                lock.unlock();
            }
        }
        return false;
    }
}
```

### ReadWriteLock

Allows **multiple readers** OR **one writer** at a time. Great for read-heavy systems.

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Cache<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();

    public V get(K key) {
        lock.readLock().lock();  // Multiple threads can read simultaneously
        try {
            return map.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }

    public void put(K key, V value) {
        lock.writeLock().lock();  // Only one thread can write
        try {
            map.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

---

## 7. Concurrent Collections

Standard collections are NOT thread-safe. Use concurrent alternatives.

| Standard | Thread-Safe Alternative |
|----------|----------------------|
| `HashMap` | `ConcurrentHashMap` |
| `ArrayList` | `CopyOnWriteArrayList` |
| `HashSet` | `CopyOnWriteArraySet` |
| `LinkedList` | `ConcurrentLinkedQueue` |
| `TreeMap` | `ConcurrentSkipListMap` |
| `PriorityQueue` | `PriorityBlockingQueue` |

```java
// ConcurrentHashMap — thread-safe without locking entire map
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("key", 1);
map.computeIfAbsent("key2", k -> 2);

// CopyOnWriteArrayList — safe for iteration, expensive for writes
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
list.add("item");

// BlockingQueue — producer-consumer pattern
BlockingQueue<String> queue = new LinkedBlockingQueue<>(100);
queue.put("task");      // Blocks if queue is full
String task = queue.take();  // Blocks if queue is empty
```

---

## 8. Thread Pools (ExecutorService)

Don't create threads manually — use thread pools.

```java
import java.util.concurrent.*;

// Fixed thread pool — fixed number of threads
ExecutorService fixedPool = Executors.newFixedThreadPool(4);

// Cached thread pool — creates threads as needed, reuses idle ones
ExecutorService cachedPool = Executors.newCachedThreadPool();

// Single thread executor — one thread, tasks queued
ExecutorService singleThread = Executors.newSingleThreadExecutor();

// Scheduled thread pool — for periodic tasks
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// Submit tasks
fixedPool.submit(() -> {
    System.out.println("Task executed by: " + Thread.currentThread().getName());
});

// Submit with return value
Future<String> future = fixedPool.submit(() -> {
    return "Result from background task";
});

String result = future.get();  // Blocks until result is available

// Shutdown
fixedPool.shutdown();  // Graceful — waits for running tasks
fixedPool.shutdownNow();  // Forceful — interrupts running tasks
```

### Custom Thread Pool

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,              // Core pool size
    4,              // Maximum pool size
    60L,            // Keep-alive time
    TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100),  // Work queue
    new ThreadPoolExecutor.CallerRunsPolicy()  // Rejection policy
);
```

---

## 9. Deadlock

Two or more threads are **waiting for each other's locks** — resulting in infinite waiting.

### Deadlock Example

```java
public class DeadlockExample {
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();

    public void method1() {
        synchronized (lock1) {           // Acquires lock1
            System.out.println("Holding lock1...");
            synchronized (lock2) {       // Waits for lock2
                System.out.println("Holding lock1 and lock2");
            }
        }
    }

    public void method2() {
        synchronized (lock2) {           // Acquires lock2
            System.out.println("Holding lock2...");
            synchronized (lock1) {       // Waits for lock1 → DEADLOCK!
                System.out.println("Holding lock2 and lock1");
            }
        }
    }
}
```

### How to Prevent Deadlock

1. **Lock Ordering** — Always acquire locks in the same order

```java
// Always acquire lock1 before lock2
public void method1() {
    synchronized (lock1) {
        synchronized (lock2) { /* ... */ }
    }
}

public void method2() {
    synchronized (lock1) {    // Same order!
        synchronized (lock2) { /* ... */ }
    }
}
```

2. **Try-Lock with Timeout**

```java
public void safeMethod() {
    boolean gotLock = false;
    try {
        gotLock = lock.tryLock(1, TimeUnit.SECONDS);
        if (gotLock) {
            // Do work
        } else {
            // Handle failure — don't wait forever
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        if (gotLock) lock.unlock();
    }
}
```

3. **Avoid Nested Locks** — Minimize locking scope

---

## 10. Producer-Consumer Pattern

A fundamental concurrency pattern using `BlockingQueue`.

```java
public class ProducerConsumer {

    public static void main(String[] args) {
        BlockingQueue<String> queue = new LinkedBlockingQueue<>(10);

        // Producer
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 20; i++) {
                    String item = "Item-" + i;
                    queue.put(item);  // Blocks if queue is full
                    System.out.println("Produced: " + item);
                }
                queue.put("DONE");  // Poison pill
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        // Consumer
        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    String item = queue.take();  // Blocks if queue is empty
                    if ("DONE".equals(item)) break;
                    System.out.println("Consumed: " + item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
    }
}
```

---

## 11. Thread Safety in LLD Design

### Making a Class Thread-Safe

```java
// Approach 1: Immutable objects (best — no synchronization needed)
public final class ImmutableConfig {
    private final String host;
    private final int port;

    public ImmutableConfig(String host, int port) {
        this.host = host;
        this.port = port;
    }

    public String getHost() { return host; }
    public int getPort() { return port; }
}

// Approach 2: Synchronized access
public class ThreadSafeCounter {
    private int count = 0;
    
    public synchronized void increment() { count++; }
    public synchronized int getCount() { return count; }
}

// Approach 3: Atomic variables
public class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);
    
    public void increment() { count.incrementAndGet(); }
    public int getCount() { return count.get(); }
}

// Approach 4: Concurrent collections
public class ThreadSafeRegistry {
    private final ConcurrentHashMap<String, Object> registry = new ConcurrentHashMap<>();
    
    public void register(String key, Object value) { registry.put(key, value); }
    public Object get(String key) { return registry.get(key); }
}
```

### Thread-Safety Decision Guide

| Scenario | Approach |
|----------|----------|
| Object never changes | Immutable (best) |
| Simple counter/flag | Atomic classes |
| Complex shared state | Locks / synchronized |
| Shared collection | ConcurrentHashMap, etc. |
| Producer-consumer | BlockingQueue |
| Read-heavy, rare writes | ReadWriteLock |

---

## 12. Common Interview Questions

1. **What is thread safety?** — A class is thread-safe if it behaves correctly when accessed from multiple threads simultaneously.
2. **synchronized vs ReentrantLock?** — ReentrantLock offers try-lock, fairness, and interruptible locking.
3. **volatile vs AtomicInteger?** — volatile ensures visibility; AtomicInteger ensures both visibility and atomicity.
4. **How to detect deadlock?** — Thread dump analysis, `jstack`, or `ThreadMXBean`.
5. **What is a race condition?** — When the correctness of the program depends on the relative timing of threads.
