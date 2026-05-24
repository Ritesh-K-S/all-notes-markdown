# 23 - Task Scheduler / Cron Job System

## 📋 Problem Statement

Design a task scheduler that:
- Executes tasks at scheduled times or intervals
- Supports one-time and recurring tasks
- Handles task priority and concurrency

---

## 📌 Requirements

### Functional Requirements
1. **Schedule** a task for a specific time (one-time)
2. **Schedule** recurring tasks (every N seconds/minutes/hours)
3. **Priority-based** execution
4. **Cancel** a scheduled task
5. **Concurrent execution** — thread pool for tasks
6. **Retry** on task failure

---

## 💻 Code Implementation

### Task Definition (Command Pattern)

```java
public interface Task {
    void execute();
    String getTaskId();
    String getName();
}

public class ScheduledTask {
    private String taskId;
    private Task task;
    private LocalDateTime scheduledTime;
    private Duration interval; // null for one-time
    private int priority;
    private int maxRetries;
    private int retryCount;
    private TaskStatus status;

    public ScheduledTask(Task task, LocalDateTime scheduledTime, Duration interval, int priority) {
        this.taskId = UUID.randomUUID().toString();
        this.task = task;
        this.scheduledTime = scheduledTime;
        this.interval = interval;
        this.priority = priority;
        this.maxRetries = 3;
        this.retryCount = 0;
        this.status = TaskStatus.SCHEDULED;
    }

    public boolean isRecurring() {
        return interval != null;
    }

    public void reschedule() {
        if (isRecurring()) {
            this.scheduledTime = scheduledTime.plus(interval);
            this.status = TaskStatus.SCHEDULED;
            this.retryCount = 0;
        }
    }

    // Getters
    public LocalDateTime getScheduledTime() { return scheduledTime; }
    public int getPriority() { return priority; }
    public Task getTask() { return task; }
    public TaskStatus getStatus() { return status; }
    public void setStatus(TaskStatus status) { this.status = status; }
}

public enum TaskStatus {
    SCHEDULED, RUNNING, COMPLETED, FAILED, CANCELLED
}
```

### Task Scheduler

```java
public class TaskScheduler {
    private PriorityQueue<ScheduledTask> taskQueue;
    private ExecutorService executorService;
    private Map<String, ScheduledTask> taskMap;
    private volatile boolean running;

    public TaskScheduler(int threadPoolSize) {
        // Priority: earliest time first, then higher priority
        this.taskQueue = new PriorityQueue<>((a, b) -> {
            int timeCompare = a.getScheduledTime().compareTo(b.getScheduledTime());
            if (timeCompare != 0) return timeCompare;
            return Integer.compare(b.getPriority(), a.getPriority()); // higher priority first
        });
        this.executorService = Executors.newFixedThreadPool(threadPoolSize);
        this.taskMap = new ConcurrentHashMap<>();
        this.running = false;
    }

    public String scheduleTask(Task task, LocalDateTime time, int priority) {
        ScheduledTask scheduled = new ScheduledTask(task, time, null, priority);
        addTask(scheduled);
        return scheduled.getTaskId();
    }

    public String scheduleRecurring(Task task, LocalDateTime startTime, 
                                     Duration interval, int priority) {
        ScheduledTask scheduled = new ScheduledTask(task, startTime, interval, priority);
        addTask(scheduled);
        return scheduled.getTaskId();
    }

    public void cancelTask(String taskId) {
        ScheduledTask task = taskMap.get(taskId);
        if (task != null) {
            task.setStatus(TaskStatus.CANCELLED);
            taskMap.remove(taskId);
        }
    }

    private synchronized void addTask(ScheduledTask task) {
        taskQueue.offer(task);
        taskMap.put(task.getTaskId(), task);
        notifyAll(); // wake up scheduler thread
    }

    public void start() {
        running = true;
        Thread schedulerThread = new Thread(() -> {
            while (running) {
                synchronized (this) {
                    while (taskQueue.isEmpty()) {
                        try { wait(); } catch (InterruptedException e) { return; }
                    }

                    ScheduledTask next = taskQueue.peek();
                    LocalDateTime now = LocalDateTime.now();

                    if (next.getScheduledTime().isAfter(now)) {
                        try {
                            long waitMs = Duration.between(now, next.getScheduledTime()).toMillis();
                            wait(Math.max(1, waitMs));
                        } catch (InterruptedException e) { return; }
                        continue;
                    }

                    // Time to execute
                    taskQueue.poll();

                    if (next.getStatus() == TaskStatus.CANCELLED) continue;

                    next.setStatus(TaskStatus.RUNNING);
                    executorService.submit(() -> executeTask(next));
                }
            }
        });
        schedulerThread.setDaemon(true);
        schedulerThread.start();
    }

    private void executeTask(ScheduledTask scheduledTask) {
        try {
            scheduledTask.getTask().execute();
            scheduledTask.setStatus(TaskStatus.COMPLETED);

            // Reschedule if recurring
            if (scheduledTask.isRecurring()) {
                scheduledTask.reschedule();
                addTask(scheduledTask);
            }
        } catch (Exception e) {
            scheduledTask.incrementRetry();
            if (scheduledTask.getRetryCount() < scheduledTask.getMaxRetries()) {
                scheduledTask.setStatus(TaskStatus.SCHEDULED);
                addTask(scheduledTask); // retry immediately
            } else {
                scheduledTask.setStatus(TaskStatus.FAILED);
                System.err.println("Task failed after retries: " + scheduledTask.getTaskId());
            }
        }
    }

    public void stop() {
        running = false;
        executorService.shutdown();
    }
}
```

---

## 🧪 Usage Example

```java
TaskScheduler scheduler = new TaskScheduler(4);
scheduler.start();

// One-time task: 5 seconds from now
scheduler.scheduleTask(
    new PrintTask("Hello!"),
    LocalDateTime.now().plusSeconds(5),
    1
);

// Recurring task: every 10 seconds
scheduler.scheduleRecurring(
    new HealthCheckTask(),
    LocalDateTime.now(),
    Duration.ofSeconds(10),
    5
);

// Cancel a task
String taskId = scheduler.scheduleTask(new CleanupTask(), LocalDateTime.now().plusMinutes(1), 2);
scheduler.cancelTask(taskId);
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Command** | Task interface — encapsulates work |
| **Priority Queue** | Task scheduling by time + priority |
| **Observer** | Can notify on task completion/failure |

---

## 🔑 Key Takeaways

1. **PriorityQueue** sorted by scheduled time, then priority
2. **Recurring tasks** reschedule themselves after execution
3. **Thread pool** (ExecutorService) for concurrent execution
4. **Retry** with max attempts on failure
5. **wait/notify** to sleep until next task is due
