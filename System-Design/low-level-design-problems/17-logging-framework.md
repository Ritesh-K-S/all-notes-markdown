# 17 - Logging Framework

## 📋 Problem Statement

Design a logging framework (like Log4j) that:
- Supports multiple log levels (DEBUG, INFO, WARN, ERROR, FATAL)
- Outputs to multiple destinations (console, file, database)
- Is configurable and thread-safe

---

## 📌 Requirements

### Functional Requirements
1. Log levels: **DEBUG < INFO < WARN < ERROR < FATAL**
2. Only log messages **at or above** configured level
3. Multiple output sinks: **Console, File, Database**
4. **Configurable** log level at runtime
5. **Formatted messages** with timestamp, level, class, message
6. **Thread-safe** — multiple threads can log concurrently
7. **Singleton** logger instance

---

## 💻 Code Implementation

### Log Level Enum

```java
public enum LogLevel {
    DEBUG(0), INFO(1), WARN(2), ERROR(3), FATAL(4);

    private final int priority;

    LogLevel(int priority) { this.priority = priority; }
    public int getPriority() { return priority; }
}
```

### Log Sink — Chain of Responsibility

```java
public abstract class LogSink {
    protected LogLevel level;
    protected LogSink nextSink;

    public LogSink(LogLevel level) {
        this.level = level;
    }

    public void setNext(LogSink next) {
        this.nextSink = next;
    }

    public void log(LogLevel msgLevel, String message) {
        if (msgLevel.getPriority() >= level.getPriority()) {
            write(msgLevel, message);
        }
        if (nextSink != null) {
            nextSink.log(msgLevel, message);
        }
    }

    protected abstract void write(LogLevel level, String message);
}

public class ConsoleSink extends LogSink {
    public ConsoleSink(LogLevel level) { super(level); }

    @Override
    protected void write(LogLevel level, String message) {
        System.out.println("[CONSOLE] " + message);
    }
}

public class FileSink extends LogSink {
    private String filePath;

    public FileSink(LogLevel level, String filePath) {
        super(level);
        this.filePath = filePath;
    }

    @Override
    protected void write(LogLevel level, String message) {
        try (FileWriter fw = new FileWriter(filePath, true)) {
            fw.write(message + "\n");
        } catch (IOException e) {
            System.err.println("Failed to write to file: " + e.getMessage());
        }
    }
}

public class DatabaseSink extends LogSink {
    public DatabaseSink(LogLevel level) { super(level); }

    @Override
    protected void write(LogLevel level, String message) {
        // Save to database
        System.out.println("[DB] " + message);
    }
}
```

### Logger (Singleton + Chain of Responsibility)

```java
public class Logger {
    private static volatile Logger instance;
    private LogLevel minLevel;
    private LogSink sinkChain;

    private Logger() {
        this.minLevel = LogLevel.DEBUG;
    }

    public static Logger getInstance() {
        if (instance == null) {
            synchronized (Logger.class) {
                if (instance == null) {
                    instance = new Logger();
                }
            }
        }
        return instance;
    }

    public void configure(LogLevel minLevel, LogSink sinkChain) {
        this.minLevel = minLevel;
        this.sinkChain = sinkChain;
    }

    public synchronized void log(LogLevel level, String message) {
        if (level.getPriority() < minLevel.getPriority()) return;

        String formatted = String.format("[%s] [%s] [%s] %s",
            LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME),
            level,
            Thread.currentThread().getName(),
            message
        );

        if (sinkChain != null) {
            sinkChain.log(level, formatted);
        }
    }

    // Convenience methods
    public void debug(String msg) { log(LogLevel.DEBUG, msg); }
    public void info(String msg)  { log(LogLevel.INFO, msg); }
    public void warn(String msg)  { log(LogLevel.WARN, msg); }
    public void error(String msg) { log(LogLevel.ERROR, msg); }
    public void fatal(String msg) { log(LogLevel.FATAL, msg); }
}
```

### Setup & Usage

```java
// Configure sinks: Console (DEBUG+), File (ERROR+)
LogSink consoleSink = new ConsoleSink(LogLevel.DEBUG);
LogSink fileSink = new FileSink(LogLevel.ERROR, "app.log");
consoleSink.setNext(fileSink);

Logger logger = Logger.getInstance();
logger.configure(LogLevel.DEBUG, consoleSink);

logger.debug("Application started");         // → Console only
logger.info("User logged in");               // → Console only
logger.error("Database connection failed");  // → Console + File
logger.fatal("System crash!");               // → Console + File
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Singleton** | Logger — one instance globally |
| **Chain of Responsibility** | Sink chain — Console → File → DB |
| **Strategy** | Different formatting strategies |
| **Observer** | Log sinks observe log events |

---

## 🔑 Key Takeaways

1. **Singleton + Double-checked locking** for thread-safe Logger
2. **Chain of Responsibility** — each sink decides independently
3. **Level filtering** at both Logger and Sink level
4. Each sink can have **different minimum levels**
5. **Formatted output** includes timestamp, level, thread name
