# Chapter 51: Azure Queue Storage

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Queue Storage Fundamentals](#part-1-queue-storage-fundamentals)
- [Part 2: Creating Queues (Full Portal Walkthrough)](#part-2-creating-queues-full-portal-walkthrough)
- [Part 3: Working with Messages](#part-3-working-with-messages)
- [Part 4: Managing & Monitoring Queues (Portal)](#part-4-managing--monitoring-queues-portal)
- [Part 5: Deleting Queues (Portal & CLI)](#part-5-deleting-queues-portal--cli)
- [Part 6: Patterns & Use Cases](#part-6-patterns--use-cases)
- [Part 7: Terraform & Bicep](#part-7-terraform--bicep)
- [Part 8: az CLI Reference](#part-8-az-cli-reference)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Queue Storage is the simplest and cheapest way to queue messages in Azure. It's part of Azure Storage (same account as blobs and files). Think of it like a to-do list for your applications — one service adds tasks (messages) and another service picks them up and processes them. Perfect for simple background job processing where you don't need the enterprise features of Service Bus.

**Simple analogy:** Imagine a restaurant kitchen. Waiters write orders on slips (messages) and put them on a rail (queue). The chef picks up orders one by one, cooks them, and removes the slip. If the chef is busy, orders pile up on the rail but nothing gets lost. That's exactly how Queue Storage works!

```
What you'll learn:
├── Queue Storage Fundamentals
│   ├── How queues work (producer/consumer pattern)
│   ├── When to use Queue Storage vs Service Bus
│   ├── Message lifecycle explained
│   └── Pricing (extremely cheap)
├── Creating Queues (Full Portal Walkthrough)
├── Working with Messages (send, receive, peek, delete)
├── Managing & Monitoring Queues
├── Deleting Queues
├── Patterns & Use Cases
├── Terraform, Bicep, az CLI
├── Real-World Patterns
└── Quick reference
```

---

## Part 1: Queue Storage Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW QUEUE STORAGE WORKS                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ The Producer/Consumer Pattern:                                       │
│                                                                       │
│   Producer (sends messages)     Consumer (processes messages)       │
│   ┌──────────────┐             ┌──────────────┐                    │
│   │ Web App      │  ──add──→   │  Queue        │  ──pick up──→    │
│   │ API          │             │ ┌───┬───┬───┐ │   ┌───────────┐  │
│   │ Function     │             │ │msg│msg│msg│ │   │ Worker    │  │
│   └──────────────┘             │ └───┴───┴───┘ │   │ Function  │  │
│                                └──────────────┘   │ VM         │  │
│                                                    └───────────┘  │
│                                                                       │
│ Message lifecycle:                                                   │
│ 1. Producer sends message → Message enters queue                  │
│ 2. Consumer receives message → Message becomes invisible          │
│ 3. Consumer processes → Deletes message (done!)                   │
│ 4. If consumer crashes → Message reappears after timeout          │
│                                                                       │
│ Key facts:                                                           │
│ ├── Part of Azure Storage Account (same as blobs/files)          │
│ ├── Max message size: 64 KB                                       │
│ ├── Max queue size: No limit (millions of messages)              │
│ ├── Max message TTL: 7 days (or set to never expire)             │
│ ├── Messages are NOT ordered (no FIFO guarantee)                 │
│ ├── At-least-once delivery (may get duplicates)                  │
│ ├── Visibility timeout: 30 sec default (configurable)            │
│ └── Accessed via REST API, SDKs, or Azure Functions trigger      │
│                                                                       │
│ ⚡ AWS equivalent: Amazon SQS (Simple Queue Service)                │
│ ⚡ GCP equivalent: Cloud Tasks                                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Queue Storage vs Service Bus

```
┌──────────────────────┬──────────────────────┬──────────────────────┐
│ Feature              │ Queue Storage        │ Service Bus          │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Max message size     │ 64 KB                │ 256 KB (1 MB premium)│
│ FIFO ordering        │ No ❌                │ Yes ✅ (sessions)    │
│ Dead-letter queue    │ No ❌                │ Yes ✅               │
│ Duplicate detection  │ No ❌                │ Yes ✅               │
│ Transactions         │ No ❌                │ Yes ✅               │
│ Topics (pub/sub)     │ No ❌                │ Yes ✅               │
│ Pricing              │ $0.004/10K ops       │ $0.05/million ops    │
│ Max queue size       │ No limit             │ 1-100 GB             │
│ Best for             │ Simple background    │ Enterprise messaging │
│                      │ jobs, cheap & easy   │ complex workflows    │
│ Requires             │ Storage Account      │ Service Bus namespace│
└──────────────────────┴──────────────────────┴──────────────────────┘

When to use Queue Storage:
├── Simple background job processing
├── Message size < 64 KB
├── Queue size can exceed 80 GB (large backlogs)
├── Don't need FIFO ordering, dead-letter, sessions
├── Cost is primary concern
└── Already have a Storage Account

When to use Service Bus instead:
├── Need FIFO ordering (guaranteed order)
├── Need dead-letter queue (failed message handling)
├── Need topics (publish-subscribe pattern)
├── Need duplicate detection
├── Need transactions across multiple queues
└── Enterprise messaging requirements
```

### Pricing

```
┌─────────────────────────────────────────────────────────────────────┐
│           QUEUE STORAGE PRICING (Extremely Cheap!)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Storage: $0.045/GB/month                                            │
│ Operations: $0.004 per 10,000 operations                           │
│                                                                       │
│ Example costs:                                                       │
│ ├── 1 million messages/day ≈ $0.12/month                         │
│ ├── 10 million messages/day ≈ $1.20/month                        │
│ └── 100 million messages/day ≈ $12.00/month                      │
│                                                                       │
│ Compare with Service Bus:                                            │
│ ├── Service Bus Basic: $0.05/million operations                  │
│ ├── Service Bus Standard: $10/month + $0.80/million operations   │
│ └── Queue Storage is 10-100x cheaper for simple scenarios!       │
│                                                                       │
│ ⚡ Queue Storage is one of the cheapest messaging options in      │
│   any cloud provider. Great for startups and cost-conscious apps. │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating Queues (Full Portal Walkthrough)

```
Console → Storage Accounts → [your-storage-account] → Queues

┌─────────────────────────────────────────────────────────────────────┐
│           STORAGE ACCOUNT → QUEUES                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ⚡ Prerequisite: You need a Storage Account first!               │
│   If you don't have one, create a General-purpose v2 account     │
│   (see Chapter 22: Storage Accounts).                              │
│                                                                       │
│ Left menu → Data storage → Queues                                │
│                                                                       │
│ [+ Queue]                                                           │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Queue name: [background-jobs]                                │  │
│ │                                                              │  │
│ │ ⚡ Rules:                                                    │  │
│ │ ├── 3-63 characters                                         │  │
│ │ ├── Lowercase letters, numbers, hyphens only                │  │
│ │ ├── Must start with letter or number                        │  │
│ │ ├── No consecutive hyphens                                   │  │
│ │ └── Cannot end with hyphen                                   │  │
│ │                                                              │  │
│ │ [OK]                                                         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ ⚡ That's it! No extra configuration needed.                     │
│ ⚡ Queue is created inside your existing Storage Account.        │
│ ⚡ No separate resource or namespace required (unlike Service Bus)│
│ ⚡ Inherits Storage Account settings (replication, networking).  │
│                                                                       │
│ Created queues appear in the list:                                  │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Name               │ Messages (approx)                      │  │
│ ├────────────────────┼────────────────────────────────────────┤  │
│ │ background-jobs    │ 0                                       │  │
│ │ email-queue        │ 142                                     │  │
│ │ image-processing   │ 1,247                                  │  │
│ │ order-notifications│ 0                                       │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Working with Messages

```
┌─────────────────────────────────────────────────────────────────────┐
│           ADDING A MESSAGE (Portal)                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Storage Account → Queues → background-jobs → [+ Add message]     │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Message text:                                                │  │
│ │ ┌──────────────────────────────────────────────────────────┐│  │
│ │ │ {"task": "send-email", "to": "user@example.com",        ││  │
│ │ │  "subject": "Welcome!", "template": "onboarding"}       ││  │
│ │ └──────────────────────────────────────────────────────────┘│  │
│ │                                                              │  │
│ │ Expires in: [7] days                                        │  │
│ │ ⚡ Range: 1 second to 7 days                                │  │
│ │ ⚡ Check "Message never expires" for indefinite retention  │  │
│ │                                                              │  │
│ │ ☐ Encode the message body in Base64                        │  │
│ │ ⚡ Use Base64 if message has special characters/binary data│  │
│ │                                                              │  │
│ │ [OK]                                                         │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Message appears in queue:                                            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ID            │ Inserted         │ Expires          │ Size  │  │
│ ├───────────────┼──────────────────┼──────────────────┼───────┤  │
│ │ abc123-def... │ 2024-01-15 10:30 │ 2024-01-22 10:30│ 112 B │  │
│ └───────────────┴──────────────────┴──────────────────┴───────┘  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Message Operations Explained

```
┌─────────────────────────────────────────────────────────────────────┐
│           MESSAGE LIFECYCLE                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Operations:                                                          │
│                                                                       │
│ 1. SEND (Enqueue)                                                   │
│    └── Add a new message to the end of the queue                  │
│                                                                       │
│ 2. RECEIVE (Dequeue)                                                │
│    ├── Gets message from front of queue                           │
│    ├── Message becomes INVISIBLE for visibility timeout           │
│    ├── Other consumers CAN'T see it during this time             │
│    ├── You get a "pop receipt" (proof you received it)           │
│    └── Use pop receipt to delete or update the message            │
│                                                                       │
│ 3. PEEK                                                              │
│    ├── Look at messages WITHOUT making them invisible             │
│    ├── Useful for monitoring what's in the queue                 │
│    └── Does NOT change visibility or dequeue count               │
│                                                                       │
│ 4. DELETE                                                            │
│    ├── Remove a specific message (using message ID + pop receipt)│
│    ├── Do this AFTER successfully processing the message         │
│    └── If you don't delete → Message reappears after timeout    │
│                                                                       │
│ 5. UPDATE                                                            │
│    ├── Change the content of a message                            │
│    ├── Extend or reset the visibility timeout                    │
│    └── Useful for progress tracking or retry delays              │
│                                                                       │
│ 6. CLEAR                                                             │
│    └── Delete ALL messages from the queue at once                │
│                                                                       │
│ Visibility timeout explained:                                        │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Time 0s:   Consumer A reads message → invisible for 30s     │  │
│ │ Time 10s:  Consumer B tries to read → gets NOTHING          │  │
│ │ Time 20s:  Consumer A finishes → deletes message ✅         │  │
│ │                                                              │  │
│ │ OR if Consumer A crashes:                                    │  │
│ │ Time 30s:  Message reappears → Consumer B can now read it  │  │
│ │ This ensures no messages are lost!                           │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Dequeue count:                                                       │
│ ├── Tracks how many times a message has been received            │
│ ├── If dequeue count > threshold → Treat as "poison message"   │
│ ├── Move poison messages to a separate queue for investigation  │
│ └── Prevents bad messages from blocking the queue forever        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Poison Message Handling

```
┌─────────────────────────────────────────────────────────────────────┐
│           POISON MESSAGE PATTERN                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ A "poison message" is a message that keeps failing to process.     │
│ Without handling, it blocks the queue forever!                       │
│                                                                       │
│   ┌───────────┐     ┌───────────┐     ┌───────────────────┐       │
│   │ Main Queue│ ──→ │ Consumer  │ ──→ │ Process message   │       │
│   │           │     │ reads msg │     │ ├── Success? ✅    │       │
│   │           │     │           │     │ │   Delete message│       │
│   │           │     │           │     │ └── Fail? ❌      │       │
│   └───────────┘     └───────────┘     │     Check count  │       │
│                                        │     ├── < 5? Let │       │
│                                        │     │   it retry │       │
│                                        │     └── >= 5?    │       │
│   ┌───────────────┐                   │         Move to  │       │
│   │ Poison Queue  │ ←──────────────── │         poison Q │       │
│   │ (investigate) │                    └───────────────────┘       │
│   └───────────────┘                                                │
│                                                                       │
│ Example code:                                                        │
│   if (message.dequeueCount >= 5) {                                 │
│     // Move to poison queue                                        │
│     poisonQueue.sendMessage(message.messageText);                  │
│     mainQueue.deleteMessage(message.messageId, message.popReceipt);│
│   }                                                                  │
│                                                                       │
│ ⚡ Azure Functions handles this automatically!                     │
│   Set maxDequeueCount in host.json (default: 5).                  │
│   Failed messages go to {queuename}-poison queue.                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Code Example (Node.js)

```javascript
const { QueueServiceClient } = require("@azure/storage-queue");

const client = QueueServiceClient.fromConnectionString(process.env.STORAGE_CONNECTION_STRING);
const queue = client.getQueueClient("background-jobs");

// Send message
await queue.sendMessage(JSON.stringify({ task: "send-email", to: "user@example.com" }));

// Receive and process
const response = await queue.receiveMessages({ numberOfMessages: 1 });
for (const msg of response.receivedMessageItems) {
  const job = JSON.parse(msg.messageText);
  console.log("Processing:", job);
  await queue.deleteMessage(msg.messageId, msg.popReceipt); // Done!
}

// Peek at messages (without making them invisible)
const peeked = await queue.peekMessages({ numberOfMessages: 5 });
for (const msg of peeked.peekedMessageItems) {
  console.log("Peeked:", msg.messageText);
}

// Update message (change content or extend visibility)
const received = await queue.receiveMessages();
const msg = received.receivedMessageItems[0];
await queue.updateMessage(msg.messageId, msg.popReceipt, 
  JSON.stringify({ task: "send-email", status: "retrying" }), 60);

// Clear all messages
await queue.clearMessages();

// Get queue properties (approximate message count)
const properties = await queue.getProperties();
console.log("Approximate messages:", properties.approximateMessagesCount);
```

### Code Example (C# / .NET)

```csharp
using Azure.Storage.Queues;
using Azure.Storage.Queues.Models;

var client = new QueueClient(connectionString, "background-jobs");
await client.CreateIfNotExistsAsync();

// Send message
await client.SendMessageAsync(
    JsonSerializer.Serialize(new { task = "send-email", to = "user@example.com" }));

// Receive and process
QueueMessage[] messages = await client.ReceiveMessagesAsync(maxMessages: 1);
foreach (var msg in messages)
{
    Console.WriteLine($"Processing: {msg.Body}");
    await client.DeleteMessageAsync(msg.MessageId, msg.PopReceipt);
}

// Peek (no visibility change)
PeekedMessage[] peeked = await client.PeekMessagesAsync(maxMessages: 5);

// Get approximate count
QueueProperties props = await client.GetPropertiesAsync();
Console.WriteLine($"Messages: {props.ApproximateMessagesCount}");
```

---

## Part 4: Managing & Monitoring Queues (Portal)

```
┌─────────────────────────────────────────────────────────────────────┐
│           MANAGING QUEUES (Portal)                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Storage Account → Queues → [queue-name]                            │
│                                                                       │
│ Queue details:                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Queue: background-jobs                                       │  │
│ │ URL: https://stmyapp.queue.core.windows.net/background-jobs │  │
│ │ Approximate messages: 1,247                                  │  │
│ │                                                              │  │
│ │ Actions:                                                     │  │
│ │ [+ Add message] [Dequeue message] [Clear queue] [Delete]    │  │
│ │                                                              │  │
│ │ ⚡ "Dequeue message" reads and removes the next message      │  │
│ │ ⚡ "Clear queue" removes ALL messages (careful!)             │  │
│ │ ⚡ Message count is approximate (eventual consistency)       │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Access policies (for SAS tokens):                                   │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ [+ Add policy]                                               │  │
│ │                                                              │  │
│ │ Identifier: [consumer-policy]                                │  │
│ │ Permissions: ☑ Read ☑ Process ☐ Add ☐ Update              │  │
│ │ Start time: [2024-01-01]                                    │  │
│ │ Expiry time: [2025-01-01]                                   │  │
│ │                                                              │  │
│ │ ⚡ Use stored access policies for SAS token management      │  │
│ │ ⚡ Revoke access by deleting the policy                     │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Metadata:                                                            │
│ ├── Add key-value pairs to queues                                │
│ ├── Example: environment=production, owner=team-backend        │
│ └── Useful for organizing and filtering queues                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Monitoring Queue Metrics

```
┌─────────────────────────────────────────────────────────────────────┐
│           MONITORING QUEUES                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Storage Account → Monitoring → Metrics                              │
│                                                                       │
│ Key metrics to watch:                                                │
│ ├── Queue Message Count: How many messages are waiting?           │
│ │   ⚡ High count = workers can't keep up → scale workers!      │
│ │                                                                  │
│ ├── Queue Capacity: Total storage used by queues                 │
│ │   ⚡ Charges based on storage used                              │
│ │                                                                  │
│ ├── Transactions: Number of queue operations                     │
│ │   ⚡ Spike = unusual activity (burst or attack)               │
│ │                                                                  │
│ └── Availability: Is the queue service healthy?                  │
│                                                                       │
│ Set up alerts:                                                       │
│ ├── "Alert me if queue message count > 10,000"                  │
│ ├── "Alert me if queue has messages but count isn't decreasing" │
│ └── Use Azure Monitor action groups for notifications            │
│                                                                       │
│ Integration with Azure Functions:                                    │
│ ├── Functions automatically scale based on queue length!         │
│ ├── More messages → More function instances (up to limit)      │
│ ├── Queue reaches 0 → Functions scale down to 0                │
│ └── This is the most common pattern for queue processing        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Deleting Queues (Portal & CLI)

```
┌─────────────────────────────────────────────────────────────────────┐
│           DELETING QUEUES                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Portal:                                                              │
│ Storage Account → Queues → Select queue → [...] → Delete          │
│                                                                       │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ ⚠️  Delete queue "background-jobs"?                         │  │
│ │                                                              │  │
│ │ This will permanently delete the queue and ALL messages     │  │
│ │ in it. This action cannot be undone.                        │  │
│ │                                                              │  │
│ │ Type the queue name to confirm: [background-jobs]           │  │
│ │                                                              │  │
│ │ [Delete] [Cancel]                                            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ CLI:                                                                 │
│   az storage queue delete \                                        │
│     --name background-jobs \                                        │
│     --account-name stmyappprod                                     │
│                                                                       │
│ Clear all messages (keep queue):                                    │
│   az storage message clear \                                        │
│     --queue-name background-jobs \                                  │
│     --account-name stmyappprod                                     │
│                                                                       │
│ ⚡ Deleting a queue also deletes ALL messages inside it.          │
│ ⚡ There's no soft delete for queues (unlike blob containers).   │
│ ⚡ You can recreate a queue with the same name after deletion.   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Patterns & Use Cases

```
┌─────────────────────────────────────────────────────────────────────┐
│           COMMON PATTERNS                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Background Job Processing                                │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Web App → Queue → Azure Function (Queue trigger)           │  │
│ │                                                              │  │
│ │ User uploads photo → "resize-image" message → Function     │  │
│ │ resizes and saves to blob storage                           │  │
│ │                                                              │  │
│ │ User response: Instant (no waiting for resize!)            │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Pattern 2: Load Leveling                                            │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ API receives burst of 10,000 requests/sec                   │  │
│ │ → Queue absorbs the burst (messages pile up)               │  │
│ │ → Workers process at steady 100/sec rate                   │  │
│ │ → No overload, no dropped requests!                        │  │
│ │                                                              │  │
│ │ Without queue: Server crashes under burst load              │  │
│ │ With queue: Messages wait patiently, processed in order    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Pattern 3: Decoupling Services                                      │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Order Service → Queue → Email Service                      │  │
│ │                                                              │  │
│ │ If Email Service is down:                                   │  │
│ │ ├── Without queue: Order fails! Bad experience.            │  │
│ │ └── With queue: Messages wait, emails sent when back up.  │  │
│ │                                                              │  │
│ │ Services don't need to know about each other.              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Pattern 4: Fan-out Work Distribution                                │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Single producer → Queue → Multiple worker consumers        │  │
│ │                                                              │  │
│ │ "Process 10,000 CSV rows" →                               │  │
│ │ 10,000 individual messages → 10 workers each process 1,000│  │
│ │                                                              │  │
│ │ Result: 10x faster processing!                              │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Azure Functions Queue trigger:                                       │
│ ├── Functions automatically process Queue Storage messages!      │
│ ├── Each new message triggers a function invocation              │
│ ├── Auto-scales: More messages = more function instances         │
│ ├── Queue empty = scale down to 0 (pay nothing!)               │
│ └── Most popular way to consume Queue Storage messages          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & Bicep

### Terraform

```hcl
# Storage account (prerequisite)
resource "azurerm_storage_account" "main" {
  name                     = "stmyappprod"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

# Queue
resource "azurerm_storage_queue" "jobs" {
  name                 = "background-jobs"
  storage_account_name = azurerm_storage_account.main.name
}

# Multiple queues
resource "azurerm_storage_queue" "email" {
  name                 = "email-notifications"
  storage_account_name = azurerm_storage_account.main.name
}

resource "azurerm_storage_queue" "poison" {
  name                 = "background-jobs-poison"
  storage_account_name = azurerm_storage_account.main.name
}
```

### Bicep

```bicep
// Storage account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'stmyappprod'
  location: resourceGroup().location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}

// Queue service
resource queueService 'Microsoft.Storage/storageAccounts/queueServices@2023-01-01' = {
  parent: storageAccount
  name: 'default'
}

// Queue
resource queue 'Microsoft.Storage/storageAccounts/queueServices/queues@2023-01-01' = {
  parent: queueService
  name: 'background-jobs'
  properties: {
    metadata: {
      environment: 'production'
      owner: 'backend-team'
    }
  }
}

// Poison queue
resource poisonQueue 'Microsoft.Storage/storageAccounts/queueServices/queues@2023-01-01' = {
  parent: queueService
  name: 'background-jobs-poison'
}
```

---

## Part 8: az CLI Reference

```bash
# ─── CREATE ───

# Create queue
az storage queue create \
  --name background-jobs \
  --account-name stmyappprod

# Create queue with metadata
az storage queue create \
  --name email-queue \
  --account-name stmyappprod \
  --metadata environment=production owner=backend-team

# ─── SEND MESSAGES ───

# Send a message
az storage message put \
  --queue-name background-jobs \
  --account-name stmyappprod \
  --content '{"task":"send-email","to":"user@example.com"}'

# Send with custom TTL (time-to-live in seconds)
az storage message put \
  --queue-name background-jobs \
  --account-name stmyappprod \
  --content '{"task":"urgent-report"}' \
  --time-to-live 3600

# Send with visibility delay (message visible after 60 seconds)
az storage message put \
  --queue-name background-jobs \
  --account-name stmyappprod \
  --content '{"task":"delayed-job"}' \
  --visibility-timeout 60

# ─── RECEIVE MESSAGES ───

# Peek at messages (without removing visibility)
az storage message peek \
  --queue-name background-jobs \
  --account-name stmyappprod \
  --num-messages 5

# Receive messages (makes them invisible)
az storage message get \
  --queue-name background-jobs \
  --account-name stmyappprod \
  --num-messages 1 \
  --visibility-timeout 60

# ─── UPDATE MESSAGES ───

# Update message content (need message ID and pop receipt from GET)
az storage message update \
  --queue-name background-jobs \
  --account-name stmyappprod \
  --id <message-id> \
  --pop-receipt <pop-receipt> \
  --content '{"task":"send-email","status":"retrying"}' \
  --visibility-timeout 120

# ─── DELETE MESSAGES ───

# Delete specific message (need ID and pop receipt from GET)
az storage message delete \
  --queue-name background-jobs \
  --account-name stmyappprod \
  --id <message-id> \
  --pop-receipt <pop-receipt>

# Clear ALL messages from queue
az storage message clear \
  --queue-name background-jobs \
  --account-name stmyappprod

# ─── MANAGE QUEUES ───

# List all queues in storage account
az storage queue list \
  --account-name stmyappprod \
  --output table

# Get queue metadata
az storage queue metadata show \
  --name background-jobs \
  --account-name stmyappprod

# Update queue metadata
az storage queue metadata update \
  --name background-jobs \
  --account-name stmyappprod \
  --metadata environment=staging owner=devops-team

# Check if queue exists
az storage queue exists \
  --name background-jobs \
  --account-name stmyappprod

# ─── DELETE QUEUE ───

# Delete queue and all its messages
az storage queue delete \
  --name background-jobs \
  --account-name stmyappprod

# ─── ACCESS POLICIES ───

# Set stored access policy
az storage queue policy create \
  --queue-name background-jobs \
  --account-name stmyappprod \
  --name consumer-policy \
  --permissions rp \
  --expiry 2025-01-01

# List policies
az storage queue policy list \
  --queue-name background-jobs \
  --account-name stmyappprod

# Generate SAS token using policy
az storage queue generate-sas \
  --name background-jobs \
  --account-name stmyappprod \
  --policy-name consumer-policy
```

---

## Part 9: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERN: E-COMMERCE ORDER PROCESSING              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌─────────┐   ┌─────────────┐   ┌───────────────────────────────┐│
│ │ Web App  │──→│ order-queue │──→│ Order Worker (Azure Function) ││
│ │ (Place   │   │             │   │ ├── Validate payment         ││
│ │  order)  │   │ msg: {      │   │ ├── Reserve inventory        ││
│ │          │   │  orderId,   │   │ ├── Send confirmation email  ││
│ │ Response:│   │  items,     │   │ └── Update order status      ││
│ │ "Order   │   │  customer   │   │                               ││
│ │ received"│   │ }           │   │ On failure:                   ││
│ └─────────┘   └─────────────┘   │ → Message goes to poison Q  ││
│                                   │ → Alert ops team             ││
│                                   └───────────────────────────────┘│
│                                                                       │
│ Why this works:                                                      │
│ ├── User gets instant response (no waiting for payment processing)│
│ ├── If worker fails, message reappears (automatic retry)         │
│ ├── Burst of Black Friday orders? Queue absorbs the load!        │
│ └── Cost: Processing 1M orders/day ≈ $0.12/month in queue costs│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERN: IMAGE PROCESSING PIPELINE                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────┐    ┌──────────────┐    ┌──────────────┐              │
│ │ Blob     │──→ │ resize-queue │──→ │ Resize Func  │──→ Blob    │
│ │ Upload   │    │              │    │ (thumbnail)  │   (output) │
│ │ Trigger  │    │              │    └──────────────┘              │
│ │          │──→ │ analyze-queue│──→ │ AI Analyze   │──→ DB      │
│ │          │    │              │    │ (tags, text)  │   (metadata)│
│ └──────────┘    └──────────────┘    └──────────────┘              │
│                                                                       │
│ User uploads image → Blob trigger sends to multiple queues       │
│ Each queue has its own worker → Parallel processing!              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERN: SCHEDULED REPORT GENERATION              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Timer Trigger (daily 2 AM) → Queue message per report type       │
│                                                                       │
│ Queue: report-queue                                                 │
│ ├── {type: "sales-daily", date: "2024-01-15"}                   │
│ ├── {type: "inventory", date: "2024-01-15"}                     │
│ └── {type: "customer-activity", date: "2024-01-15"}             │
│                                                                       │
│ Workers generate each report independently → Save to Blob       │
│ → Send email with download link                                   │
│                                                                       │
│ Benefits:                                                            │
│ ├── Reports generated in parallel (fast!)                        │
│ ├── If one report fails, others still complete                   │
│ └── Retry failed reports automatically                            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────────────┐
│           QUEUE STORAGE QUICK REFERENCE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Simplest, cheapest Azure message queue                       │
│ Where: Part of Storage Account (no separate resource)              │
│ Max message: 64 KB | Max queue: Unlimited (500 TB storage account)│
│ TTL: Up to 7 days (or never expire)                                │
│ Visibility timeout: 30s default (configurable up to 7 days)       │
│                                                                       │
│ Operations: Send, Receive, Peek, Delete, Update, Clear            │
│ Pricing: ~$0.004 per 10,000 operations (extremely cheap!)         │
│                                                                       │
│ Key differences from Service Bus:                                   │
│ ├── No FIFO, no dead-letter, no topics, no sessions              │
│ ├── Much cheaper (10-100x)                                        │
│ ├── No separate namespace needed                                  │
│ └── Perfect for simple background jobs                            │
│                                                                       │
│ Most common pattern:                                                 │
│ Web App → Queue → Azure Function (Queue trigger)                 │
│ Functions auto-scale based on queue depth!                         │
│                                                                       │
│ Poison messages: Track dequeue count, move to poison queue         │
│ Azure Functions: Auto-creates {queue}-poison queue after 5 fails  │
│                                                                       │
│ CLI quick commands:                                                  │
│ ├── Create: az storage queue create --name X --account-name Y    │
│ ├── Send:   az storage message put --queue-name X --content '{}'│
│ ├── Peek:   az storage message peek --queue-name X               │
│ ├── List:   az storage queue list --account-name Y               │
│ └── Delete: az storage queue delete --name X                      │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Next chapter: [Chapter 52: Azure Logic Apps](52-logic-apps.md) — Low-code workflow automation with 400+ connectors. Learn how to build automated workflows that integrate with queues, email, databases, and hundreds of other services without writing code.
