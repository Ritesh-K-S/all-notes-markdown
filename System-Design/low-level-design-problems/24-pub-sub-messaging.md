# 24 - Pub-Sub Messaging System

## 📋 Problem Statement

Design a publish-subscribe messaging system that:
- Allows publishers to send messages to topics
- Subscribers receive messages from subscribed topics
- Supports multiple topics and subscribers

---

## 📌 Requirements

### Functional Requirements
1. **Create topics**
2. **Publishers** send messages to a topic
3. **Subscribers** subscribe to topics
4. Subscribers receive messages **asynchronously**
5. Support **multiple subscribers** per topic
6. **Unsubscribe** from a topic
7. **Message ordering** — FIFO per topic
8. **At-least-once** delivery

---

## 💻 Code Implementation

### Message Class

```java
public class Message {
    private String messageId;
    private String topic;
    private String body;
    private Map<String, String> headers;
    private LocalDateTime timestamp;

    public Message(String topic, String body) {
        this.messageId = UUID.randomUUID().toString();
        this.topic = topic;
        this.body = body;
        this.headers = new HashMap<>();
        this.timestamp = LocalDateTime.now();
    }

    public String getMessageId() { return messageId; }
    public String getBody() { return body; }
    public String getTopic() { return topic; }
}
```

### Subscriber Interface

```java
public interface Subscriber {
    String getId();
    void onMessage(Message message);
}

public class PrintSubscriber implements Subscriber {
    private String id;

    public PrintSubscriber(String id) {
        this.id = id;
    }

    @Override
    public String getId() { return id; }

    @Override
    public void onMessage(Message message) {
        System.out.println("[" + id + "] Received: " + message.getBody()
            + " from topic: " + message.getTopic());
    }
}
```

### Topic Class

```java
public class Topic {
    private String name;
    private List<Subscriber> subscribers;
    private BlockingQueue<Message> messageQueue;
    private ExecutorService executorService;

    public Topic(String name) {
        this.name = name;
        this.subscribers = new CopyOnWriteArrayList<>();
        this.messageQueue = new LinkedBlockingQueue<>();
        this.executorService = Executors.newCachedThreadPool();
    }

    public void subscribe(Subscriber subscriber) {
        subscribers.add(subscriber);
        System.out.println(subscriber.getId() + " subscribed to " + name);
    }

    public void unsubscribe(Subscriber subscriber) {
        subscribers.remove(subscriber);
        System.out.println(subscriber.getId() + " unsubscribed from " + name);
    }

    public void publish(Message message) {
        messageQueue.offer(message);
        // Fan-out to all subscribers asynchronously
        for (Subscriber subscriber : subscribers) {
            executorService.submit(() -> {
                try {
                    subscriber.onMessage(message);
                } catch (Exception e) {
                    System.err.println("Failed to deliver to " + subscriber.getId() + ": " + e.getMessage());
                    // Could add retry logic here
                }
            });
        }
    }

    public String getName() { return name; }
    public int getSubscriberCount() { return subscribers.size(); }
}
```

### Message Broker (Central Hub)

```java
public class MessageBroker {
    private Map<String, Topic> topics;

    public MessageBroker() {
        this.topics = new ConcurrentHashMap<>();
    }

    public Topic createTopic(String topicName) {
        return topics.computeIfAbsent(topicName, Topic::new);
    }

    public void deleteTopic(String topicName) {
        topics.remove(topicName);
    }

    public void subscribe(String topicName, Subscriber subscriber) {
        Topic topic = topics.get(topicName);
        if (topic == null) throw new TopicNotFoundException("Topic not found: " + topicName);
        topic.subscribe(subscriber);
    }

    public void unsubscribe(String topicName, Subscriber subscriber) {
        Topic topic = topics.get(topicName);
        if (topic != null) topic.unsubscribe(subscriber);
    }

    public void publish(String topicName, String messageBody) {
        Topic topic = topics.get(topicName);
        if (topic == null) throw new TopicNotFoundException("Topic not found: " + topicName);
        Message message = new Message(topicName, messageBody);
        topic.publish(message);
    }

    public List<String> listTopics() {
        return new ArrayList<>(topics.keySet());
    }
}
```

### With Consumer Groups (Kafka-style)

```java
public class ConsumerGroup implements Subscriber {
    private String groupId;
    private List<Subscriber> consumers;
    private AtomicInteger roundRobin;

    public ConsumerGroup(String groupId) {
        this.groupId = groupId;
        this.consumers = new CopyOnWriteArrayList<>();
        this.roundRobin = new AtomicInteger(0);
    }

    public void addConsumer(Subscriber consumer) {
        consumers.add(consumer);
    }

    @Override
    public String getId() { return groupId; }

    @Override
    public void onMessage(Message message) {
        if (consumers.isEmpty()) return;
        // Round-robin: only one consumer in the group processes each message
        int index = roundRobin.getAndIncrement() % consumers.size();
        consumers.get(index).onMessage(message);
    }
}
```

---

## 🧪 Usage Example

```java
MessageBroker broker = new MessageBroker();

// Create topics
broker.createTopic("orders");
broker.createTopic("notifications");

// Create subscribers
Subscriber emailService = new PrintSubscriber("EmailService");
Subscriber smsService = new PrintSubscriber("SMSService");
Subscriber analyticsService = new PrintSubscriber("Analytics");

// Subscribe
broker.subscribe("orders", emailService);
broker.subscribe("orders", smsService);
broker.subscribe("orders", analyticsService);
broker.subscribe("notifications", emailService);

// Publish
broker.publish("orders", "New order #1234 placed");
// → [EmailService] Received: New order #1234 placed from topic: orders
// → [SMSService] Received: New order #1234 placed from topic: orders
// → [Analytics] Received: New order #1234 placed from topic: orders

broker.publish("notifications", "System maintenance at 2 AM");
// → [EmailService] Received: System maintenance at 2 AM from topic: notifications

// Consumer group (load-balanced)
ConsumerGroup orderProcessors = new ConsumerGroup("order-processors");
orderProcessors.addConsumer(new PrintSubscriber("Processor-1"));
orderProcessors.addConsumer(new PrintSubscriber("Processor-2"));
broker.subscribe("orders", orderProcessors);
// Each message goes to only ONE processor in the group
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Observer** | Subscribers observe topic messages |
| **Strategy** | Can plug different delivery strategies |
| **Mediator** | MessageBroker mediates between publishers and subscribers |

---

## 🔑 Key Takeaways

1. **Observer pattern** is the foundation of pub-sub
2. **CopyOnWriteArrayList** for thread-safe subscriber list
3. **Async delivery** via ExecutorService — publishers don't block
4. **Consumer Groups** enable load-balanced consumption (one message per group)
5. **Topic isolation** — subscribers only get messages from subscribed topics
