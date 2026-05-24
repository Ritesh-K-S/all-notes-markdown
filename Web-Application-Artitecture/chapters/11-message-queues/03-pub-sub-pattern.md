# Pub/Sub Pattern — Publishing & Subscribing to Events

> **What you'll learn**: How the Publish/Subscribe pattern works, why it's the backbone of event-driven systems, and how to implement it with Redis Pub/Sub, AWS SNS, Google Pub/Sub, and RabbitMQ Fanout exchanges.

---

## Real-Life Analogy

Think of a **YouTube channel**.

When a YouTuber uploads a new video, they don't personally call each of their 10 million subscribers to say "Hey, I posted a video!" Instead:

1. The YouTuber **publishes** a video (the event)
2. YouTube's system **broadcasts** a notification to ALL subscribers
3. Each subscriber gets the notification **independently** — some watch immediately, some watch later, some ignore it

The YouTuber (publisher) doesn't know or care who the subscribers are. Subscribers don't need to know about each other. They're **completely decoupled**.

This is **Pub/Sub** — one publisher, many subscribers, zero knowledge of each other.

---

## Core Concept Explained Step-by-Step

### Step 1: Pub/Sub vs Point-to-Point

In the previous chapter, we saw **point-to-point queues** where one message goes to one consumer. Pub/Sub is fundamentally different:

```
POINT-TO-POINT (Queue):               PUB/SUB (Topic):
                                       
One message → One consumer             One message → ALL subscribers
                                       
Producer ──▶ [Queue] ──▶ Consumer      Publisher ──▶ [Topic] ──▶ Subscriber A
                                                             ──▶ Subscriber B
"Task distribution"                                          ──▶ Subscriber C
                                       
                                       "Event broadcasting"
```

### Step 2: The Three Actors in Pub/Sub

```
┌───────────┐                                    ┌──────────────┐
│ Publisher  │         ┌─────────────┐           │ Subscriber A │
│ (doesn't  │────────▶│             │──────────▶│ (Email Svc)  │
│  know who │         │   TOPIC /   │           └──────────────┘
│  listens) │         │   CHANNEL   │           ┌──────────────┐
└───────────┘         │             │──────────▶│ Subscriber B │
                      │ "UserSignup"│           │ (Analytics)  │
                      │             │           └──────────────┘
                      │             │           ┌──────────────┐
                      │             │──────────▶│ Subscriber C │
                      └─────────────┘           │ (CRM Sync)   │
                                                └──────────────┘
```

1. **Publisher**: Emits events. Doesn't know (or care) who subscribes.
2. **Topic/Channel**: A named category that messages are published to.
3. **Subscribers**: Services that register interest in a topic and receive ALL messages published to it.

### Step 3: Why Pub/Sub is Powerful

Imagine adding a new feature — "send SMS on user signup":

**Without Pub/Sub (tight coupling):**
```
// UserService.java — must be modified every time!
public void signup(User user) {
    saveToDb(user);
    emailService.sendWelcome(user);    // Direct call
    analyticsService.track(user);       // Direct call
    crmService.sync(user);              // Direct call
    smsService.sendSMS(user);           // NEW: Must modify this file!
}
```

**With Pub/Sub (loose coupling):**
```
// UserService.java — NEVER changes!
public void signup(User user) {
    saveToDb(user);
    eventBus.publish("user.signup", user);  // Fire and forget
}

// SMS Service subscribes independently — UserService doesn't know it exists
@Subscribe("user.signup")
public void onUserSignup(User user) {
    sendWelcomeSMS(user.getPhone());
}
```

Adding a new subscriber **requires ZERO changes** to the publisher. This is the power of decoupling.

### Step 4: Fan-Out Architecture

Pub/Sub enables **fan-out** — one event triggers multiple independent actions:

```
                           USER SIGNUP EVENT
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
            ▼                    ▼                    ▼
    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
    │ Send Welcome │    │   Track in   │    │   Create     │
    │    Email     │    │  Analytics   │    │  CRM Record  │
    └──────────────┘    └──────────────┘    └──────────────┘
            │                    │                    │
            ▼                    ▼                    ▼
    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
    │ Send Welcome │    │  Update      │    │  Assign to   │
    │    SMS       │    │  Dashboard   │    │  Sales Rep   │
    └──────────────┘    └──────────────┘    └──────────────┘

Each subscriber processes the event INDEPENDENTLY.
If one fails, others are NOT affected.
```

---

## How It Works Internally

### Pattern Variations

**1. Ephemeral Pub/Sub (Fire-and-Forget)**
- Messages are NOT stored. If subscriber is offline, it misses the message.
- Fast, low latency.
- Example: Redis Pub/Sub, WebSocket broadcasts.

```
Publisher ──▶ Broker ──▶ Online subscribers get it
                    ──╳──▶ Offline subscriber MISSES it
```

**2. Durable Pub/Sub (Persistent)**
- Messages ARE stored until all subscribers have consumed them.
- Reliable, no missed messages.
- Example: Kafka topics, Google Pub/Sub, AWS SNS+SQS.

```
Publisher ──▶ Broker ──▶ Online subscriber gets it immediately
                    ──▶ Offline subscriber gets it WHEN IT COMES BACK
                         (message was stored)
```

### How Redis Pub/Sub Works Internally

```
┌─────────────────────────────────────────────────────────────────┐
│                      REDIS SERVER                                 │
│                                                                   │
│  Channel Registry (hash map):                                    │
│  ┌────────────────────────────────────────────────────────┐      │
│  │ "user.signup"  → [Client_A, Client_B, Client_C]       │      │
│  │ "order.placed" → [Client_D, Client_E]                  │      │
│  │ "user.*"       → [Client_F]  (pattern subscription)   │      │
│  └────────────────────────────────────────────────────────┘      │
│                                                                   │
│  On PUBLISH "user.signup" message:                               │
│  1. Look up "user.signup" → find [Client_A, B, C]               │
│  2. Check pattern subs → "user.*" matches → add Client_F        │
│  3. Send message to all matched clients via TCP                  │
│  4. Message is NOT stored (ephemeral!)                           │
└─────────────────────────────────────────────────────────────────┘
```

**Redis Pub/Sub limitations:**
- No persistence — messages lost if subscriber is offline
- No acknowledgments — no way to know if subscriber processed it
- No replay — can't go back and read old messages
- Best for: real-time notifications, cache invalidation, chat rooms

### How AWS SNS + SQS Works (Fan-Out Pattern)

AWS combines SNS (Pub/Sub) with SQS (Queues) for durable fan-out:

```
┌──────────┐     ┌───────────────┐     ┌──────────────┐     ┌──────────────┐
│Publisher  │────▶│   SNS TOPIC   │────▶│  SQS Queue 1 │────▶│ Email Service│
│(Order Svc)│     │ "OrderPlaced" │     │(email-orders)│     └──────────────┘
└──────────┘     │               │     └──────────────┘
                 │               │     ┌──────────────┐     ┌──────────────┐
                 │               │────▶│  SQS Queue 2 │────▶│Inventory Svc │
                 │               │     │(inv-orders)  │     └──────────────┘
                 │               │     └──────────────┘
                 │               │     ┌──────────────┐     ┌──────────────┐
                 │               │────▶│  SQS Queue 3 │────▶│Analytics Svc │
                 └───────────────┘     │(analytics)   │     └──────────────┘
                                       └──────────────┘

SNS = Broadcasting layer (pub/sub)
SQS = Durable buffer per subscriber (queue)
Together = Reliable fan-out with independent processing
```

**Why SNS + SQS together?**
- SNS alone doesn't store messages (like Redis Pub/Sub)
- Adding SQS behind each subscription gives durability
- Each service has its OWN queue → independent processing speed
- If analytics service is slow, it doesn't affect email service

### How Google Cloud Pub/Sub Works

Google Pub/Sub is a fully managed durable Pub/Sub system:

```
┌──────────┐     ┌───────────────┐     ┌──────────────────┐     ┌──────────┐
│Publisher  │────▶│    TOPIC      │────▶│  Subscription A  │────▶│Consumer A│
│           │     │               │     │  (pull or push)  │     └──────────┘
└──────────┘     │               │     └──────────────────┘
                 │               │     ┌──────────────────┐     ┌──────────┐
                 │               │────▶│  Subscription B  │────▶│Consumer B│
                 └───────────────┘     │  (pull or push)  │     └──────────┘
                                       └──────────────────┘

Features:
- Messages stored until ACK'd (7 days default)
- Exactly-once delivery (with dedup)
- Global distribution
- Auto-scaling consumers
```

---

## Code Examples

### Python — Redis Pub/Sub

```python
import redis
import json
import threading

# ============ PUBLISHER ============
def publish_event(channel, event_data):
    """Publish an event to a Redis channel"""
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)
    
    message = json.dumps(event_data)
    # Returns number of subscribers who received the message
    num_receivers = r.publish(channel, message)
    print(f"Published to '{channel}' → {num_receivers} subscribers received it")


# ============ SUBSCRIBER ============
def subscribe_to_events(channel, handler_fn):
    """Subscribe to a Redis channel and process events"""
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)
    pubsub = r.pubsub()
    
    pubsub.subscribe(channel)
    print(f"Subscribed to '{channel}'. Listening...")
    
    for message in pubsub.listen():
        if message['type'] == 'message':
            event = json.loads(message['data'])
            handler_fn(event)


# ============ USAGE ============
# Subscriber 1: Email service
def email_handler(event):
    print(f"[EMAIL] Sending welcome to: {event['email']}")

# Subscriber 2: Analytics
def analytics_handler(event):
    print(f"[ANALYTICS] Tracking signup: {event['user_id']}")

# Start subscribers in background threads
threading.Thread(target=subscribe_to_events, 
                 args=("user.signup", email_handler), daemon=True).start()
threading.Thread(target=subscribe_to_events, 
                 args=("user.signup", analytics_handler), daemon=True).start()

# Publish an event (both subscribers receive it!)
publish_event("user.signup", {
    "user_id": "U-123",
    "email": "alice@example.com",
    "name": "Alice"
})
```

### Python — AWS SNS Pub/Sub

```python
import boto3
import json

# ============ PUBLISHER ============
def publish_to_sns(topic_arn, event):
    """Publish event to SNS topic (all subscribers get it)"""
    sns = boto3.client('sns', region_name='us-east-1')
    
    response = sns.publish(
        TopicArn=topic_arn,
        Message=json.dumps(event),
        Subject='OrderPlaced',
        MessageAttributes={
            'event_type': {
                'DataType': 'String',
                'StringValue': 'order.placed'
            }
        }
    )
    print(f"Published! MessageId: {response['MessageId']}")


# ============ SUBSCRIBE SQS QUEUE TO SNS TOPIC ============
def setup_fan_out(topic_arn, queue_arn):
    """Subscribe an SQS queue to receive all messages from SNS topic"""
    sns = boto3.client('sns', region_name='us-east-1')
    
    sns.subscribe(
        TopicArn=topic_arn,
        Protocol='sqs',
        Endpoint=queue_arn
    )
    print(f"Queue {queue_arn} now receives all messages from topic")


# Usage
topic = 'arn:aws:sns:us-east-1:123456789:order-events'
publish_to_sns(topic, {"order_id": "ORD-456", "amount": 99.99})
```

### Java — RabbitMQ Fanout Exchange (Pub/Sub)

```java
import com.rabbitmq.client.*;

// ============ PUBLISHER ============
public class EventPublisher {
    private static final String EXCHANGE_NAME = "user_events";
    
    public void publishSignupEvent(String userJson) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        
        try (Connection conn = factory.newConnection();
             Channel channel = conn.createChannel()) {
            
            // Declare FANOUT exchange (broadcasts to ALL bound queues)
            channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, true);
            
            channel.basicPublish(EXCHANGE_NAME, "", null, userJson.getBytes());
            System.out.println("Published signup event (all subscribers get it)");
        }
    }
}

// ============ SUBSCRIBER ============
public class EmailSubscriber {
    private static final String EXCHANGE_NAME = "user_events";
    
    public void startListening() throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection conn = factory.newConnection();
        Channel channel = conn.createChannel();
        
        // Declare exchange
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT, true);
        
        // Create exclusive queue for THIS subscriber
        String queueName = channel.queueDeclare().getQueue();
        
        // Bind our queue to the fanout exchange
        channel.queueBind(queueName, EXCHANGE_NAME, "");
        
        System.out.println("[EMAIL] Waiting for signup events...");
        
        DeliverCallback callback = (tag, delivery) -> {
            String message = new String(delivery.getBody());
            System.out.println("[EMAIL] Sending welcome email for: " + message);
        };
        
        channel.basicConsume(queueName, true, callback, tag -> {});
    }
}
```

### Java — Google Cloud Pub/Sub

```java
import com.google.cloud.pubsub.v1.Publisher;
import com.google.cloud.pubsub.v1.Subscriber;
import com.google.pubsub.v1.TopicName;
import com.google.pubsub.v1.PubsubMessage;
import com.google.protobuf.ByteString;

// ============ PUBLISHER ============
public class PubSubPublisher {
    
    public void publishEvent(String projectId, String topicId, String data) 
            throws Exception {
        TopicName topicName = TopicName.of(projectId, topicId);
        Publisher publisher = Publisher.newBuilder(topicName).build();
        
        PubsubMessage message = PubsubMessage.newBuilder()
            .setData(ByteString.copyFromUtf8(data))
            .putAttributes("event_type", "user.signup")
            .build();
        
        // Publish returns a future (async internally)
        publisher.publish(message).get();
        System.out.println("Published to " + topicId);
        publisher.shutdown();
    }
}

// ============ SUBSCRIBER ============
public class PubSubSubscriber {
    
    public void subscribe(String projectId, String subscriptionId) {
        ProjectSubscriptionName subName = 
            ProjectSubscriptionName.of(projectId, subscriptionId);
        
        MessageReceiver receiver = (message, consumer) -> {
            String data = message.getData().toStringUtf8();
            System.out.println("Received: " + data);
            consumer.ack();  // Acknowledge processing
        };
        
        Subscriber subscriber = Subscriber.newBuilder(subName, receiver).build();
        subscriber.startAsync().awaitRunning();
        System.out.println("Listening on: " + subscriptionId);
    }
}
```

---

## Infrastructure Examples

### AWS SNS + SQS Fan-Out with Terraform

```hcl
# SNS Topic (the "publisher channel")
resource "aws_sns_topic" "order_events" {
  name = "order-events"
}

# SQS Queue for Email Service
resource "aws_sqs_queue" "email_queue" {
  name                       = "email-order-notifications"
  visibility_timeout_seconds = 60
}

# SQS Queue for Inventory Service
resource "aws_sqs_queue" "inventory_queue" {
  name                       = "inventory-updates"
  visibility_timeout_seconds = 120
}

# Subscribe Email Queue to SNS Topic
resource "aws_sns_topic_subscription" "email_sub" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.email_queue.arn
}

# Subscribe Inventory Queue to SNS Topic
resource "aws_sns_topic_subscription" "inventory_sub" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.inventory_queue.arn
}

# Allow SNS to send to SQS
resource "aws_sqs_queue_policy" "email_policy" {
  queue_url = aws_sqs_queue.email_queue.id
  policy = jsonencode({
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "sns.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.email_queue.arn
      Condition = {
        ArnEquals = { "aws:SourceArn" = aws_sns_topic.order_events.arn }
      }
    }]
  })
}
```

---

## Real-World Example

### Netflix — Event Broadcasting at Scale

Netflix uses Pub/Sub extensively for their recommendation and content systems:

```
NETFLIX EVENT ARCHITECTURE (Simplified):

User watches a movie
        │
        ▼
┌───────────────┐     ┌─────────────────────────────────────────┐
│  Playback     │────▶│  "viewing.completed" TOPIC (Kafka)      │
│  Service      │     └────────────┬────────────────────────────┘
└───────────────┘                  │
                    ┌──────────────┼──────────────┬──────────────┐
                    ▼              ▼              ▼              ▼
          ┌──────────────┐ ┌─────────────┐ ┌──────────┐ ┌──────────┐
          │Recommendation│ │  Billing    │ │  Content │ │  A/B     │
          │   Engine     │ │  Service    │ │  Ranking │ │  Testing │
          │(update model)│ │(track usage)│ │(trending)│ │(metrics) │
          └──────────────┘ └─────────────┘ └──────────┘ └──────────┘

Each subscriber processes the SAME event for DIFFERENT purposes.
If one subscriber is slow, it doesn't affect others.
Netflix can add new subscribers WITHOUT changing the Playback Service.
```

### Shopify — Webhook Events

Shopify's webhook system is essentially Pub/Sub:
- Merchants subscribe to topics like `orders/create`, `products/update`
- When an order is created, Shopify publishes to the topic
- ALL subscribed merchants get notified via HTTP webhooks
- 26+ billion webhook deliveries per day

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using ephemeral Pub/Sub for critical events | Messages lost when subscriber is down | Use durable Pub/Sub (Kafka, SNS+SQS, Google Pub/Sub) |
| Subscribers that are too slow | Message backlog grows indefinitely | Scale consumers, set message TTL, monitor lag |
| Publishing large payloads | Network overhead, memory pressure | Publish event metadata + reference; let subscriber fetch full data |
| No schema for events | Breaking changes crash all subscribers | Use schema registry (Avro, Protobuf) with versioning |
| Circular event loops | Service A → event → Service B → event → Service A → ∞ | Design acyclic event flows, add loop detection |
| All subscribers in same failure domain | One crash takes down all processing | Run subscribers in independent deployment units |
| Not handling duplicate events | Data corruption, double processing | Make subscribers idempotent |

---

## When to Use / When NOT to Use

### ✅ Use Pub/Sub When:

- **One event triggers multiple independent actions** (fan-out)
- **Services should be unaware of each other** (loose coupling)
- **You add new consumers frequently** without changing publishers
- **Event broadcasting** across teams/services (e.g., "UserCreated" event)
- **Real-time notifications** to multiple clients (chat, live updates)

### ❌ Don't Use Pub/Sub When:

- **Only one consumer needs the message** (use point-to-point queue instead)
- **You need request-response semantics** (use sync HTTP/gRPC)
- **Message ordering is critical across all subscribers** (Pub/Sub makes ordering complex)
- **You need exactly-once processing** without additional infrastructure
- **Simple systems** where direct function calls suffice

### Pub/Sub Tool Comparison:

| Tool | Durability | Ordering | Scale | Best For |
|------|-----------|----------|-------|----------|
| **Redis Pub/Sub** | None (ephemeral) | Per channel | Medium | Real-time notifications, cache invalidation |
| **RabbitMQ Fanout** | Optional | Per queue | High | Complex routing + broadcasting |
| **AWS SNS + SQS** | Durable (with SQS) | FIFO available | Unlimited | AWS-native fan-out |
| **Google Pub/Sub** | Durable | Per key | Unlimited | GCP-native, global |
| **Apache Kafka** | Durable (log) | Per partition | Very High | Event sourcing, streaming at scale |

---

## Key Takeaways

- **Pub/Sub** = one message delivered to ALL subscribers. Unlike queues (one message → one consumer).
- Publishers and subscribers are **completely decoupled** — they don't know about each other.
- **Fan-out** is the killer use case: one event triggers many independent reactions.
- **Ephemeral** Pub/Sub (Redis) is fast but lossy. **Durable** Pub/Sub (Kafka, SNS+SQS) guarantees delivery.
- The **SNS + SQS** pattern on AWS gives you durable fan-out with independent processing per subscriber.
- Adding new subscribers requires **zero changes** to the publisher — this is the ultimate decoupling.
- Always make subscribers **idempotent** — duplicates are common in distributed Pub/Sub systems.

---

## What's Next?

We've covered basic Pub/Sub, but for truly large-scale event streaming (millions of events/second, replay capabilities, long-term storage), you need something more powerful. In the next chapter, [04-apache-kafka.md](./04-apache-kafka.md), we'll dive deep into **Apache Kafka** — the distributed event streaming platform that powers LinkedIn, Netflix, Uber, and thousands of other companies.
