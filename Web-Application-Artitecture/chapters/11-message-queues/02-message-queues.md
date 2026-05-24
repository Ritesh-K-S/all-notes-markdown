# Message Queues — RabbitMQ, Amazon SQS

> **What you'll learn**: How message queues work internally, the architecture of RabbitMQ and Amazon SQS, and how to use them to build reliable, decoupled, and scalable systems.

---

## Real-Life Analogy

Think of a **post office**.

You (the sender) write a letter, put it in an envelope, and drop it in the mailbox. You don't wait for the recipient to read it — you go about your day. The post office **stores** your letter, **routes** it to the right destination, and **delivers** it when the recipient is ready to pick it up.

Now imagine the recipient is on vacation. Does the post office throw away your letter? No! It holds it safely until the recipient comes back.

A **message queue** is exactly like this post office:
- **Producer** = You (the sender)
- **Queue** = The post office (stores and routes messages)
- **Consumer** = The recipient (processes messages when ready)

The queue guarantees your message won't be lost, even if the recipient is temporarily unavailable.

---

## Core Concept Explained Step-by-Step

### Step 1: What IS a Message Queue?

A message queue is a **middleware component** that:
1. **Accepts messages** from producers (senders)
2. **Stores them durably** (on disk or in memory)
3. **Delivers them** to consumers (receivers) when they're ready
4. **Removes them** once successfully processed

```
┌──────────┐                              ┌──────────┐
│ Producer │                              │ Consumer │
│(Order    │     ┌────────────────┐       │(Email    │
│ Service) │────▶│  MESSAGE QUEUE │──────▶│ Service) │
│          │     │                │       │          │
└──────────┘     │ [msg1][msg2].. │       └──────────┘
                 │                │
                 │ FIFO (First In │
                 │  First Out)    │
                 └────────────────┘
```

### Step 2: Point-to-Point vs Pub/Sub

There are two fundamental messaging patterns:

**Point-to-Point (Queue)**: One message → One consumer

```
                    POINT-TO-POINT
                    
Producer ──▶ [Queue] ──▶ Consumer A  ✓ (gets message)
                    ╳──▶ Consumer B  ✗ (doesn't get it)
                    
Each message is delivered to EXACTLY ONE consumer.
Used for: task distribution, work queues
```

**Pub/Sub (Topic)**: One message → ALL subscribers

```
                    PUB/SUB
                    
                         ┌──▶ Subscriber A  ✓ (gets copy)
Publisher ──▶ [Topic] ──┼──▶ Subscriber B  ✓ (gets copy)
                         └──▶ Subscriber C  ✓ (gets copy)
                    
Each message is delivered to ALL subscribers.
Used for: event broadcasting, notifications
```

(We'll cover Pub/Sub in depth in Chapter 11.3)

### Step 3: Message Queue Components

```
┌─────────────────────────────────────────────────────────────┐
│                    MESSAGE BROKER                             │
│                                                              │
│  ┌───────────┐    ┌─────────────────────────┐    ┌───────┐ │
│  │ Exchange/ │    │         QUEUE            │    │Delivery│ │
│  │  Router   │───▶│ [msg4][msg3][msg2][msg1] │───▶│Engine  │ │
│  └───────────┘    └─────────────────────────┘    └───────┘ │
│       ▲                      │                       │      │
│       │                 Persistence                   │      │
│       │                 (disk/WAL)                    │      │
└───────┼──────────────────────────────────────────────┼──────┘
        │                                              │
   Producer                                        Consumer
  (publishes)                                    (subscribes)
```

**Key components:**
- **Producer**: Application that sends messages
- **Exchange/Router**: Decides which queue gets the message (RabbitMQ concept)
- **Queue**: Ordered storage for messages
- **Consumer**: Application that receives and processes messages
- **Acknowledgment (ACK)**: Consumer tells broker "I processed this successfully"

### Step 4: Message Lifecycle

```
1. PRODUCE         2. STORE           3. DELIVER         4. ACK & DELETE
                   
Producer           Broker             Consumer           Broker
   │                  │                  │                  │
   │── publish ──────▶│                  │                  │
   │                  │── write to ──┐   │                  │
   │                  │   disk       │   │                  │
   │◀── ACK ─────────│◀─────────────┘   │                  │
   │                  │                  │                  │
   │                  │── deliver ──────▶│                  │
   │                  │                  │── process ──┐    │
   │                  │                  │◀────────────┘    │
   │                  │◀── ACK ──────────│                  │
   │                  │                  │                  │
   │                  │── delete msg ─┐  │                  │
   │                  │◀──────────────┘  │                  │
```

---

## How It Works Internally

### RabbitMQ Architecture

RabbitMQ uses the **AMQP (Advanced Message Queuing Protocol)** standard.

```
┌──────────────────────────────────────────────────────────────────┐
│                      RABBITMQ BROKER                               │
│                                                                    │
│  ┌──────────┐     ┌──────────────┐     ┌──────────────────────┐  │
│  │          │     │   EXCHANGE   │     │       QUEUES          │  │
│  │ Producer │────▶│              │────▶│                       │  │
│  │          │     │ • Direct     │     │  queue_a: [m1][m2]    │  │
│  └──────────┘     │ • Fanout    │     │  queue_b: [m3][m4]    │  │
│                   │ • Topic     │     │  queue_c: [m5]        │  │
│                   │ • Headers   │     │                       │  │
│                   └──────────────┘     └──────────┬───────────┘  │
│                         │                         │               │
│                    Routing Key              Consumers pull         │
│                    Bindings                 or push                │
│                                                   │               │
└───────────────────────────────────────────────────┼───────────────┘
                                                    │
                                            ┌───────┴───────┐
                                            │   Consumers   │
                                            └───────────────┘
```

**RabbitMQ Exchange Types:**

| Exchange Type | Routing Logic | Use Case |
|--------------|---------------|----------|
| **Direct** | Exact routing key match | Task queues, specific routing |
| **Fanout** | Sends to ALL bound queues | Broadcasting events |
| **Topic** | Pattern matching (`order.*`, `#.error`) | Flexible routing |
| **Headers** | Match on message headers | Complex routing rules |

**Internal Storage:**
- Messages stored in **Mnesia** (Erlang's built-in database) for metadata
- Message bodies stored on **disk** (for durable queues) or in **RAM** (for transient)
- Uses **Write-Ahead Log (WAL)** for crash recovery
- **Flow control**: If queues fill up, RabbitMQ throttles producers (backpressure)

### Amazon SQS Architecture

SQS is a fully managed queue service — you don't manage servers.

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS SQS (Managed)                          │
│                                                              │
│  ┌─────────┐    ┌──────────────────────────────┐            │
│  │Producer │───▶│    DISTRIBUTED QUEUE          │            │
│  │  (API   │    │                              │            │
│  │  call)  │    │  ┌─────┐ ┌─────┐ ┌─────┐   │            │
│  └─────────┘    │  │Node1│ │Node2│ │Node3│   │ ──▶ Consumer│
│                 │  │[m1] │ │[m3] │ │[m5] │   │            │
│                 │  │[m2] │ │[m4] │ │[m6] │   │            │
│                 │  └─────┘ └─────┘ └─────┘   │            │
│                 │  (replicated across AZs)     │            │
│                 └──────────────────────────────┘            │
│                                                              │
│  Features: Auto-scaling, no capacity planning,              │
│  replicated across 3+ AZs, virtually unlimited throughput   │
└─────────────────────────────────────────────────────────────┘
```

**SQS Types:**

| Feature | Standard Queue | FIFO Queue |
|---------|---------------|------------|
| **Throughput** | Unlimited | 300 msg/s (3000 with batching) |
| **Ordering** | Best-effort (not guaranteed) | Strict FIFO |
| **Delivery** | At-least-once (may duplicate) | Exactly-once |
| **Use case** | High-throughput tasks | Order-sensitive workflows |

**SQS Internal Mechanisms:**
- **Visibility Timeout**: When a consumer reads a message, it becomes "invisible" to other consumers for N seconds. If not ACK'd, it reappears.
- **Long Polling**: Consumer waits up to 20s for messages (reduces empty responses and cost)
- **Message Retention**: Up to 14 days
- **Dead Letter Queue**: Failed messages automatically moved after N retries

```
VISIBILITY TIMEOUT FLOW:

Consumer A reads msg1
    │
    ▼
msg1 becomes INVISIBLE (30s default)
    │
    ├── Consumer A processes successfully → deletes msg1 ✓
    │
    └── Consumer A crashes/times out → msg1 becomes VISIBLE again
                                       → another consumer can retry
```

---

## Code Examples

### Python — RabbitMQ Producer & Consumer

```python
import pika
import json
import time

# ============ PRODUCER ============
def send_order_to_queue(order_data):
    """Send an order event to RabbitMQ"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(
            host='localhost',
            credentials=pika.PlainCredentials('admin', 'secret')
        )
    )
    channel = connection.channel()
    
    # Declare queue (idempotent - safe to call multiple times)
    channel.queue_declare(queue='orders', durable=True)
    
    # Publish message with persistence
    channel.basic_publish(
        exchange='',                    # Default exchange (direct to queue)
        routing_key='orders',           # Queue name
        body=json.dumps(order_data),
        properties=pika.BasicProperties(
            delivery_mode=2,            # Make message persistent (survives restart)
            content_type='application/json'
        )
    )
    print(f"[✓] Sent order: {order_data['order_id']}")
    connection.close()


# ============ CONSUMER ============
def process_orders():
    """Consume and process orders from the queue"""
    connection = pika.BlockingConnection(
        pika.ConnectionParameters('localhost')
    )
    channel = connection.channel()
    channel.queue_declare(queue='orders', durable=True)
    
    # Fair dispatch: don't give a worker more than 1 unacked message
    channel.basic_qos(prefetch_count=1)
    
    def callback(ch, method, properties, body):
        order = json.loads(body)
        print(f"[...] Processing order: {order['order_id']}")
        
        try:
            # Simulate processing
            time.sleep(2)
            print(f"[✓] Order {order['order_id']} completed!")
            # ACK = tell RabbitMQ we're done, safe to delete
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception as e:
            print(f"[✗] Failed: {e}")
            # NACK = tell RabbitMQ to requeue the message
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
    
    channel.basic_consume(queue='orders', on_message_callback=callback)
    print("[*] Waiting for orders. Press CTRL+C to exit")
    channel.start_consuming()


# Usage
send_order_to_queue({"order_id": "ORD-123", "item": "Laptop", "qty": 1})
```

### Python — Amazon SQS

```python
import boto3
import json

# ============ PRODUCER ============
def send_to_sqs(order_data):
    """Send message to Amazon SQS"""
    sqs = boto3.client('sqs', region_name='us-east-1')
    queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789/orders-queue'
    
    response = sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=json.dumps(order_data),
        MessageAttributes={
            'OrderType': {
                'StringValue': 'purchase',
                'DataType': 'String'
            }
        }
    )
    print(f"Message sent! ID: {response['MessageId']}")


# ============ CONSUMER ============
def poll_sqs():
    """Long-poll messages from SQS"""
    sqs = boto3.client('sqs', region_name='us-east-1')
    queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789/orders-queue'
    
    while True:
        # Long polling (waits up to 20s for messages)
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,      # Batch up to 10
            WaitTimeSeconds=20,          # Long poll
            VisibilityTimeout=60         # 60s to process before requeue
        )
        
        messages = response.get('Messages', [])
        for msg in messages:
            order = json.loads(msg['Body'])
            print(f"Processing: {order}")
            
            # Process successfully → delete from queue
            sqs.delete_message(
                QueueUrl=queue_url,
                ReceiptHandle=msg['ReceiptHandle']
            )
            print(f"Deleted message: {msg['MessageId']}")
```

### Java — RabbitMQ with Spring AMQP

```java
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Service;

// ============ CONFIGURATION ============
@Configuration
public class RabbitConfig {
    @Bean
    public Queue ordersQueue() {
        return new Queue("orders", true);  // durable = true
    }
}

// ============ PRODUCER ============
@Service
public class OrderProducer {
    private final RabbitTemplate rabbitTemplate;
    
    public OrderProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }
    
    public void sendOrder(Order order) {
        rabbitTemplate.convertAndSend("orders", order);
        System.out.println("Order sent: " + order.getId());
    }
}

// ============ CONSUMER ============
@Service
public class OrderConsumer {
    
    @RabbitListener(queues = "orders")
    public void processOrder(Order order) {
        System.out.println("Received order: " + order.getId());
        // Process the order (update DB, send notification, etc.)
        inventoryService.reduceStock(order.getItems());
        notificationService.notifyCustomer(order.getCustomerId());
    }
}
```

### Java — Amazon SQS with AWS SDK

```java
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.*;

public class SqsOrderService {
    
    private final SqsClient sqs = SqsClient.builder().build();
    private final String queueUrl = "https://sqs.us-east-1.amazonaws.com/123/orders";
    
    // PRODUCER
    public void sendOrder(String orderJson) {
        SendMessageRequest request = SendMessageRequest.builder()
            .queueUrl(queueUrl)
            .messageBody(orderJson)
            .delaySeconds(0)              // No delay
            .build();
        
        SendMessageResponse response = sqs.sendMessage(request);
        System.out.println("Sent: " + response.messageId());
    }
    
    // CONSUMER (long polling)
    public void pollOrders() {
        while (true) {
            ReceiveMessageRequest receiveReq = ReceiveMessageRequest.builder()
                .queueUrl(queueUrl)
                .maxNumberOfMessages(10)
                .waitTimeSeconds(20)       // Long polling
                .visibilityTimeout(60)
                .build();
            
            List<Message> messages = sqs.receiveMessage(receiveReq).messages();
            
            for (Message msg : messages) {
                processOrder(msg.body());
                
                // Delete after successful processing
                sqs.deleteMessage(DeleteMessageRequest.builder()
                    .queueUrl(queueUrl)
                    .receiptHandle(msg.receiptHandle())
                    .build());
            }
        }
    }
}
```

---

## Infrastructure Examples

### RabbitMQ Cluster with Docker Compose

```yaml
version: '3.8'
services:
  rabbitmq-node1:
    image: rabbitmq:3.12-management
    hostname: rabbit1
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_ERLANG_COOKIE: 'secret_cookie'
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: secret
    volumes:
      - rabbit1_data:/var/lib/rabbitmq
    networks:
      - rabbit_net

  rabbitmq-node2:
    image: rabbitmq:3.12-management
    hostname: rabbit2
    environment:
      RABBITMQ_ERLANG_COOKIE: 'secret_cookie'
    depends_on:
      - rabbitmq-node1
    networks:
      - rabbit_net

  # Producer service
  order-api:
    build: ./order-api
    environment:
      RABBITMQ_URL: amqp://admin:secret@rabbitmq-node1:5672
    depends_on:
      - rabbitmq-node1

  # Consumer service (can scale independently)
  order-processor:
    build: ./order-processor
    environment:
      RABBITMQ_URL: amqp://admin:secret@rabbitmq-node1:5672
    deploy:
      replicas: 3  # 3 workers processing in parallel
    depends_on:
      - rabbitmq-node1

volumes:
  rabbit1_data:
networks:
  rabbit_net:
```

### AWS SQS with Terraform

```hcl
resource "aws_sqs_queue" "orders" {
  name                       = "orders-queue"
  delay_seconds              = 0
  max_message_size           = 262144          # 256 KB
  message_retention_seconds  = 1209600         # 14 days
  receive_wait_time_seconds  = 20              # Long polling
  visibility_timeout_seconds = 60

  # Enable Dead Letter Queue
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.orders_dlq.arn
    maxReceiveCount     = 3                    # After 3 failures → DLQ
  })

  tags = {
    Environment = "production"
    Service     = "order-processing"
  }
}

# Dead Letter Queue for failed messages
resource "aws_sqs_queue" "orders_dlq" {
  name                      = "orders-queue-dlq"
  message_retention_seconds = 1209600           # Keep failed msgs 14 days
}
```

---

## Real-World Example

### Uber — Message Queues for Ride Matching

When a rider requests a ride, Uber doesn't synchronously call every driver:

```
UBER'S ASYNC ARCHITECTURE (Simplified):

Rider requests ride
       │
       ▼
┌─────────────────┐         ┌──────────────────┐
│   Ride Request  │────────▶│   KAFKA QUEUE    │
│    Service      │         │ (ride_requests)   │
└─────────────────┘         └────────┬─────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
            ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
            │  Matching   │  │   Pricing   │  │    ETA      │
            │  Service    │  │   Service   │  │   Service   │
            └─────────────┘  └─────────────┘  └─────────────┘
                    │
                    ▼
            ┌─────────────┐         ┌──────────────────┐
            │ Driver Push │────────▶│  PUSH QUEUE      │
            │  Service    │         │ (driver_notifs)   │
            └─────────────┘         └──────────────────┘
                                            │
                                     ┌──────┼──────┐
                                     ▼      ▼      ▼
                                  Driver  Driver  Driver
                                    A       B       C
```

### Slack — Message Delivery

Slack uses message queues internally to handle millions of messages:
- When you send a message, it goes to a queue
- Multiple consumers handle: delivery to recipients, push notifications, search indexing, analytics, file processing (if attachment)
- This allows Slack to handle 10+ billion messages/day

---

## Common Mistakes / Pitfalls

| Mistake | Consequence | Solution |
|---------|-------------|----------|
| Not setting messages as durable | Messages lost on broker restart | Set `delivery_mode=2` (persistent) |
| Not acknowledging messages | Messages redelivered infinitely | Always ACK after successful processing |
| Single consumer, no scaling | Queue depth grows, processing delays | Run multiple consumers, use `prefetch_count` |
| Not monitoring queue depth | Silent failures, SLA breaches | Alert on queue depth > threshold |
| Poison messages (can't be processed) | Consumer crashes in loop | Use Dead Letter Queues (max retry count) |
| Very large messages (>1MB) | Slow, memory issues | Store payload in S3/DB, put reference in queue |
| No idempotency | Duplicate processing = double charges | Use idempotency keys, deduplication |
| Forgetting visibility timeout (SQS) | Two consumers process same message | Set timeout > max processing time |

---

## When to Use / When NOT to Use

### ✅ Use Message Queues When:

- **Decoupling services**: Producer and consumer can evolve independently
- **Buffering traffic spikes**: Queue absorbs bursts, consumers process at steady rate
- **Task distribution**: Spread work across multiple workers
- **Reliability needed**: Messages must not be lost (even if consumer is down)
- **Different processing speeds**: Producer is fast, consumer is slow
- **Cross-service communication**: In microservices architectures

### ❌ Don't Use Message Queues When:

- **You need real-time response**: User needs answer NOW (use sync HTTP)
- **Simple in-process communication**: Threads in same app (use in-memory queues)
- **Low throughput, simple system**: Adding a broker is unnecessary overhead for 100 req/day
- **Strong ordering is critical AND high throughput**: Standard queues sacrifice ordering for speed
- **You can't handle eventual consistency**: If data must be immediately consistent everywhere

### RabbitMQ vs SQS Decision Guide:

| Criteria | Choose RabbitMQ | Choose SQS |
|----------|----------------|------------|
| **Infrastructure** | Self-managed/on-prem | AWS-native, serverless |
| **Routing complexity** | Complex routing (exchanges, bindings) | Simple queue semantics |
| **Protocol** | AMQP, MQTT, STOMP | HTTP/HTTPS (AWS SDK) |
| **Throughput** | ~50K msg/s per node | Virtually unlimited |
| **Cost model** | Server costs (fixed) | Per-request pricing (pay-per-use) |
| **Ops burden** | You manage clustering, HA | AWS manages everything |

---

## Key Takeaways

- A **message queue** is middleware that stores messages between producers and consumers, enabling asynchronous decoupled communication.
- **RabbitMQ** uses exchanges and bindings for flexible routing (AMQP protocol), ideal for complex routing scenarios.
- **Amazon SQS** is a fully managed queue with unlimited scale, ideal for AWS-native applications.
- **Durability** (persistent messages + durable queues) ensures messages survive broker restarts.
- **Acknowledgments (ACKs)** prevent message loss — a message is only deleted after the consumer confirms processing.
- **Visibility Timeout** (SQS) / **Prefetch + ACK** (RabbitMQ) prevent multiple consumers from processing the same message simultaneously.
- Always plan for failure: use **Dead Letter Queues**, **monitoring**, and **idempotent consumers**.

---

## What's Next?

Now that you understand how point-to-point message queues work, let's explore the **Pub/Sub Pattern** — where a single message is delivered to MULTIPLE subscribers. In the next chapter, [03-pub-sub-pattern.md](./03-pub-sub-pattern.md), we'll dive into event broadcasting, fan-out architectures, and tools like Redis Pub/Sub, SNS, and Google Pub/Sub.
