# Chapter 47: Amazon EventBridge

---

## Table of Contents

- [Overview](#overview)
- [Part 1: EventBridge Fundamentals](#part-1-eventbridge-fundamentals)
- [Part 2: Creating an Event Rule (Full Portal Walkthrough)](#part-2-creating-an-event-rule-full-portal-walkthrough)
- [Part 3: Custom Event Bus & Event Patterns](#part-3-custom-event-bus--event-patterns)
- [Part 4: Schema Registry & Pipes](#part-4-schema-registry--pipes)
- [Part 5: Terraform & CLI Examples](#part-5-terraform--cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Event-Driven Architecture? Why Do We Need EventBridge?

Think of a **doorbell**: when someone presses it (event), it triggers an action (you go to the door). You don't constantly check the door — you only react when the bell rings.

**EventBridge works the same way for your applications.** When something happens in AWS (an EC2 instance stops, an S3 object is uploaded, a CodePipeline fails), EventBridge detects the **event** and routes it to the right **target** (Lambda, SQS, Step Functions, etc.).

**EventBridge vs SNS — What's the difference?**
- **SNS**: Simple pub/sub — publish a message, all subscribers get it
- **EventBridge**: Smart event router — you write **rules** that filter events by content and route them to different targets. Much more powerful filtering and routing.

**Simple real-world examples:**
- 💾 S3 object uploaded → EventBridge rule → Lambda processes the file
- ⏰ Every day at 2 AM → EventBridge schedule → Lambda runs cleanup job
- 🚨 EC2 instance terminated → EventBridge rule → SNS sends alert to team
- 🔗 Stripe payment received (SaaS event) → EventBridge → Lambda updates order status

Amazon EventBridge is a serverless event bus that connects applications using events from AWS services, SaaS apps, and custom applications. It's the evolution of CloudWatch Events and the preferred way to build event-driven architectures on AWS.

```
What you'll learn:
├── EventBridge fundamentals (event bus, rules, targets)
├── Creating event rules (portal walkthrough)
├── Custom event buses
├── Event patterns (filtering)
├── Schema Registry & Pipes
└── Terraform & CLI examples
```

---

## Part 1: EventBridge Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW EVENTBRIDGE WORKS                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Event Sources → Event Bus → Rules → Targets                       │
│                                                                       │
│ ┌────────────────┐   ┌───────────┐   ┌────────────────────────┐  │
│ │ Event Sources  │   │ Event Bus │   │ Targets               │  │
│ ├────────────────┤   │           │   ├────────────────────────┤  │
│ │ AWS Services   │──▶│ Rules     │──▶│ Lambda                │  │
│ │ (EC2, S3, etc.)│   │ (filter  │   │ SQS / SNS             │  │
│ │                │   │  & route) │   │ Step Functions         │  │
│ │ Custom Apps    │──▶│           │──▶│ API Gateway            │  │
│ │ (PutEvents)    │   │           │   │ Kinesis                │  │
│ │                │   │           │   │ ECS Task               │  │
│ │ SaaS Partners  │──▶│           │──▶│ CodePipeline           │  │
│ │ (Zendesk, etc.)│   │           │   │ CloudWatch Logs        │  │
│ │                │   │           │   │ Another Event Bus      │  │
│ │ Scheduled      │   │           │   │ API Destination (HTTP) │  │
│ └────────────────┘   └───────────┘   └────────────────────────┘  │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Event: JSON document describing what happened               │
│ ├── Event Bus: Channel that receives events                     │
│ │   ├── Default bus: Receives AWS service events automatically │
│ │   ├── Custom bus: Your application events                     │
│ │   └── Partner bus: SaaS partner events                       │
│ ├── Rule: Pattern match + target(s)                             │
│ │   ├── Event pattern: Match specific events                   │
│ │   └── Schedule: Cron/rate based (like CloudWatch Events)    │
│ ├── Target: Where matching events are sent (up to 5/rule)     │
│ └── Schema: Event structure definition                          │
│                                                                       │
│ EventBridge vs SNS:                                                  │
│ ┌──────────────────────┬───────────────┬─────────────────────┐   │
│ │                      │ EventBridge   │ SNS                  │   │
│ ├──────────────────────┼───────────────┼─────────────────────┤   │
│ │ Event sources        │ AWS, SaaS,    │ Custom only          │   │
│ │                      │ custom        │                      │   │
│ │ Filtering            │ Content-based │ Attribute-based      │   │
│ │                      │ (powerful)    │ (simple)             │   │
│ │ Targets              │ 20+ AWS svc   │ SQS, Lambda, HTTP   │   │
│ │ Schema discovery     │ ✅            │ ❌                   │   │
│ │ Archive & Replay     │ ✅            │ ❌                   │   │
│ │ Scheduling           │ ✅            │ ❌                   │   │
│ │ Input transformation │ ✅            │ ❌                   │   │
│ │ ⚡ Preferred for     │ Event-driven  │ Simple fan-out       │   │
│ │                      │ architecture  │                      │   │
│ └──────────────────────┴───────────────┴─────────────────────┘   │
│                                                                       │
│ Pricing:                                                             │
│ ├── AWS service events: FREE (published to default bus)         │
│ ├── Custom events: $1.00/million events                         │
│ ├── Cross-account events: $1.00/million                        │
│ ├── Schema discovery: Free                                      │
│ └── Archive: $0.10/GB stored                                    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating an Event Rule (Full Portal Walkthrough)

```
Console → EventBridge → Rules → Create rule

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: DEFINE RULE DETAIL                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [ec2-state-change-rule]                                 │
│ Description: [Trigger on EC2 instance state changes]          │
│                                                                   │
│ Event bus: [default ▼]                                        │
│ → default: AWS service events                                │
│ → custom-bus: Your application events                        │
│                                                                   │
│ Rule type:                                                     │
│ ● Rule with an event pattern                                  │
│   → Matches incoming events                                  │
│ ○ Schedule                                                     │
│   → Runs on a cron or rate schedule                          │
│   → ⚡ Use EventBridge Scheduler for new schedules          │
│                                                                   │
│ ☑ Enable the rule on the selected event bus                  │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: BUILD EVENT PATTERN                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Event source:                                                  │
│ ● AWS events or EventBridge partner events                   │
│ ○ Other (custom event pattern)                                │
│                                                                   │
│ ── AWS Events ──                                                │
│                                                                   │
│ Creation method:                                               │
│ ● Use pattern form                                            │
│ ○ Use custom pattern (JSON)                                   │
│                                                                   │
│ Event source: [AWS services ▼]                                │
│ AWS service: [EC2 ▼]                                          │
│ Event type: [EC2 Instance State-change Notification ▼]       │
│                                                                   │
│ Specific state(s):                                             │
│ ☑ terminated                                                  │
│ ☑ stopped                                                     │
│ ☐ running                                                     │
│ ☐ pending                                                     │
│                                                                   │
│ Specific instance Id(s): (optional)                           │
│ ☐ Any instance                                                │
│ ○ Specific instances: [i-abc123, i-def456]                   │
│                                                                   │
│ Generated event pattern:                                      │
│ {                                                              │
│   "source": ["aws.ec2"],                                     │
│   "detail-type": ["EC2 Instance State-change Notification"], │
│   "detail": {                                                 │
│     "state": ["terminated", "stopped"]                       │
│   }                                                            │
│ }                                                              │
│                                                                   │
│ [Test pattern] → Test against sample events                  │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: SELECT TARGETS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Target 1:                                                      │
│ Target type:                                                   │
│ ● AWS service                                                  │
│ ○ EventBridge event bus (cross-account/region)               │
│ ○ EventBridge API destination (HTTP endpoint)                │
│                                                                   │
│ Select a target: [Lambda function ▼]                          │
│ Function: [ec2-state-handler ▼]                               │
│                                                                   │
│ Configure version/alias: [$LATEST ▼]                         │
│                                                                   │
│ ── Input transformer (optional) ──                             │
│ ● Matched event (full event JSON)                            │
│ ○ Part of the matched event                                   │
│ ○ Constant (JSON text)                                        │
│ ○ Input transformer (template)                               │
│                                                                   │
│ Input transformer example:                                    │
│ Input path:                                                    │
│ {"instance":"$.detail.instance-id","state":"$.detail.state"} │
│                                                                   │
│ Input template:                                                │
│ "Instance <instance> changed to <state>"                     │
│                                                                   │
│ Retry policy:                                                  │
│ Maximum age of event: [24] hours (1 min - 24 hours)          │
│ Retry attempts: [185] (0 - 185)                              │
│                                                                   │
│ Dead-letter queue:                                             │
│ ☑ Select an SQS queue as dead-letter queue                   │
│ [arn:aws:sqs:us-east-1:123456:eventbridge-dlq ▼]            │
│                                                                   │
│ [+ Add another target] (up to 5 per rule)                    │
│                                                                   │
│ Target 2:                                                      │
│ Select a target: [SNS topic ▼]                                │
│ Topic: [ops-alerts ▼]                                         │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 4: TAGS (optional)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ [Environment] = [Production]                                  │
│ [Team] = [Platform]                                           │
│                                                                   │
│                    [Next → Review → Create rule]              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Custom Event Bus & Event Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           CUSTOM EVENT BUS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → EventBridge → Event buses → Create event bus             │
│                                                                       │
│ Name: [my-app-events]                                               │
│                                                                       │
│ Resource-based policy (optional):                                   │
│ → Allow cross-account PutEvents                                    │
│ → Allow specific IAM roles/accounts                                │
│                                                                       │
│ Event archive (optional):                                            │
│ ☑ Enable archiving                                                  │
│ Archive name: [my-app-events-archive]                              │
│ Retention: [90] days (0 = indefinite)                              │
│ → Replay archived events for debugging/recovery                  │
│                                                                       │
│ Schema discovery: ☑ Start discovery                                │
│ → Auto-discovers event schemas from published events             │
│ → Generates code bindings (Java, Python, TypeScript)             │
│                                                                       │
│ Publishing custom events:                                            │
│ aws events put-events --entries '[{                                 │
│   "Source": "com.myapp.orders",                                     │
│   "DetailType": "Order Created",                                    │
│   "Detail": "{\"orderId\":\"12345\",\"amount\":99.99}",           │
│   "EventBusName": "my-app-events"                                  │
│ }]'                                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           EVENT PATTERN SYNTAX                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Exact match:                                                         │
│ {"source": ["com.myapp.orders"]}                                   │
│                                                                       │
│ Prefix match:                                                        │
│ {"source": [{"prefix": "com.myapp"}]}                              │
│                                                                       │
│ Suffix match:                                                        │
│ {"detail": {"filename": [{"suffix": ".png"}]}}                    │
│                                                                       │
│ Anything-but:                                                        │
│ {"detail": {"status": [{"anything-but": ["test"]}]}}              │
│                                                                       │
│ Numeric:                                                              │
│ {"detail": {"amount": [{"numeric": [">=", 100, "<", 1000]}]}}    │
│                                                                       │
│ Exists:                                                               │
│ {"detail": {"error": [{"exists": true}]}}                          │
│                                                                       │
│ IP address:                                                          │
│ {"detail": {"sourceIP": [{"cidr": "10.0.0.0/8"}]}}               │
│                                                                       │
│ Wildcard:                                                            │
│ {"detail": {"key": [{"wildcard": "prod-*-app"}]}}                 │
│                                                                       │
│ Combining (AND = nested, OR = array):                               │
│ {                                                                     │
│   "source": ["com.myapp.orders"],                                  │
│   "detail-type": ["Order Created", "Order Updated"],               │
│   "detail": {                                                        │
│     "amount": [{"numeric": [">=", 100]}],                          │
│     "region": ["us-east-1", "us-west-2"]                          │
│   }                                                                   │
│ }                                                                     │
│ → source AND detail-type AND amount AND region all must match     │
│ → Arrays = OR (any value matches)                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Schema Registry & Pipes

```
┌─────────────────────────────────────────────────────────────────────┐
│           SCHEMA REGISTRY                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → EventBridge → Schema registry                             │
│                                                                       │
│ Registries:                                                          │
│ ├── aws.events: All AWS service event schemas                    │
│ ├── discovered-schemas: Auto-discovered from your events        │
│ └── Custom registries: Your manually created schemas            │
│                                                                       │
│ Features:                                                            │
│ ├── Browse schemas for any AWS event                             │
│ ├── Download code bindings (Java, Python, TypeScript)           │
│ ├── Auto-discover schemas from custom events                    │
│ └── Use in IDE with EventBridge toolkit                         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           EVENTBRIDGE PIPES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Point-to-point integrations between event producers          │
│ and consumers with optional filtering, enrichment, transformation │
│                                                                       │
│ Source → Filter → Enrichment → Target                              │
│                                                                       │
│ Sources:                                                              │
│ ├── SQS queue                                                     │
│ ├── Kinesis stream                                                │
│ ├── DynamoDB stream                                               │
│ ├── Managed Kafka / self-managed Kafka                           │
│ └── Amazon MQ (ActiveMQ, RabbitMQ)                               │
│                                                                       │
│ Enrichment (optional):                                               │
│ ├── Lambda function                                               │
│ ├── API Gateway                                                   │
│ ├── API destination (HTTP)                                       │
│ └── Step Functions                                                │
│                                                                       │
│ Targets:                                                             │
│ ├── Lambda, SQS, SNS, Step Functions                             │
│ ├── EventBridge event bus                                         │
│ ├── Kinesis, Firehose, CloudWatch Logs                          │
│ ├── API Gateway, API destination                                 │
│ └── ECS task, Batch job, Redshift                                │
│                                                                       │
│ ⚡ Use Pipes for: SQS → enrich with Lambda → send to Step Fns   │
│ ⚡ Use Rules for: Event matching and fan-out to multiple targets │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4.5: EventBridge Scheduler

EventBridge Scheduler is the **recommended way to create scheduled tasks** on AWS. It replaces CloudWatch Events scheduled rules and EventBridge scheduled rules with a purpose-built scheduler.

### Why Use Scheduler Instead of Scheduled Rules?

| Feature | Scheduled Rules | EventBridge Scheduler |
|---------|----------------|----------------------|
| One-time schedules | ❌ No | ✅ Yes |
| Time zones | ❌ UTC only | ✅ Any timezone |
| Flexible time windows | ❌ No | ✅ Yes (spread execution) |
| Scale | Limited rules per bus | Millions of schedules |
| Retry with backoff | ❌ No | ✅ Built-in |
| Target types | EventBridge targets | 270+ AWS API actions |
| Cost | $1/million events | $1/million invocations |

### Console Walkthrough

```
Console → EventBridge → Scheduler → Schedules → Create schedule

  ┌─────────────────────────────────────────────────────────┐
  │ Schedule pattern                                         │
  │                                                          │
  │ ● Recurring schedule                                     │
  │   ● Rate-based: Every 5 minutes                         │
  │   ○ Cron-based: cron(0 9 * * ? *)  ← 9 AM daily        │
  │ ○ One-time schedule                                      │
  │   At: 2025-12-31T23:59:00                               │
  │                                                          │
  │ Timezone: Asia/Kolkata                                   │
  │                                                          │
  │ Flexible time window: 15 minutes                         │
  │ (Spreads execution randomly within window - good for     │
  │  avoiding thundering herd on shared resources)           │
  │                                                          │
  │ Target: Lambda function / Step Functions / SQS / SNS /   │
  │         Any of 270+ AWS API actions                      │
  └─────────────────────────────────────────────────────────┘
```

### Common Use Cases

```
├── Daily reports: cron(0 9 * * ? *) → Lambda → generate & email report
├── Nightly cleanup: cron(0 2 * * ? *) → Lambda → delete old logs/temp files
├── One-time: Schedule a deployment at 2 AM Sunday
├── Database backup: Every 6 hours → trigger RDS snapshot
└── Scaling schedule: Scale up at 8 AM, scale down at 6 PM
```

### Terraform Example

```hcl
resource "aws_scheduler_schedule" "daily_report" {
  name       = "daily-report"
  group_name = "default"

  flexible_time_window {
    mode = "OFF"
  }

  schedule_expression          = "cron(0 9 * * ? *)"
  schedule_expression_timezone = "Asia/Kolkata"

  target {
    arn      = aws_lambda_function.report.arn
    role_arn = aws_iam_role.scheduler.arn
  }
}
```

> 💡 **Migration tip:** If you have existing CloudWatch Events scheduled rules, migrate them to EventBridge Scheduler for better features and scale.

---

## Part 5: Terraform & CLI Examples

```hcl
# EventBridge rule (AWS service event)
resource "aws_cloudwatch_event_rule" "ec2_state" {
  name        = "ec2-state-change"
  description = "Capture EC2 instance state changes"
  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Instance State-change Notification"]
    detail      = { state = ["terminated", "stopped"] }
  })
}

resource "aws_cloudwatch_event_target" "lambda" {
  rule = aws_cloudwatch_event_rule.ec2_state.name
  arn  = aws_lambda_function.handler.arn
}

# Custom event bus
resource "aws_cloudwatch_event_bus" "app" {
  name = "my-app-events"
}

# Rule on custom bus
resource "aws_cloudwatch_event_rule" "orders" {
  name           = "order-created"
  event_bus_name = aws_cloudwatch_event_bus.app.name
  event_pattern = jsonencode({
    source      = ["com.myapp.orders"]
    detail-type = ["Order Created"]
    detail      = { amount = [{ numeric = [">=", 100] }] }
  })
}

# Archive
resource "aws_cloudwatch_event_archive" "app" {
  name             = "my-app-events-archive"
  event_source_arn = aws_cloudwatch_event_bus.app.arn
  retention_days   = 90
}

# Schedule (EventBridge Scheduler)
resource "aws_scheduler_schedule" "daily_report" {
  name       = "daily-report"
  group_name = "default"

  flexible_time_window { mode = "OFF" }

  schedule_expression = "cron(0 8 * * ? *)"  # 8 AM UTC daily

  target {
    arn      = aws_lambda_function.report.arn
    role_arn = aws_iam_role.scheduler.arn
  }
}
```

```bash
# Put custom events
aws events put-events --entries '[{
  "Source": "com.myapp.orders",
  "DetailType": "Order Created",
  "Detail": "{\"orderId\":\"12345\",\"amount\":99.99}",
  "EventBusName": "my-app-events"
}]'

# List rules
aws events list-rules --event-bus-name my-app-events

# Describe rule
aws events describe-rule --name order-created --event-bus-name my-app-events

# Replay archived events
aws events start-replay \
  --replay-name debug-replay-jan \
  --event-source-arn arn:aws:events:us-east-1:123456:event-bus/my-app-events \
  --destination '{"Arn":"arn:aws:events:us-east-1:123456:event-bus/my-app-events"}' \
  --event-start-time 2024-01-01T00:00:00Z \
  --event-end-time 2024-01-02T00:00:00Z
```

---

## Part 6: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Event-Driven Microservices                                │
│ Order Service → EventBridge → Inventory Service                   │
│                             → Payment Service                      │
│                             → Notification Service                 │
│ → Services are completely decoupled                               │
│ → New consumers can subscribe without changing producer          │
│                                                                       │
│ Pattern 2: AWS Resource Automation                                  │
│ EC2 terminated → EventBridge → Lambda (cleanup DNS)              │
│ S3 object created → EventBridge → Lambda (process file)          │
│ GuardDuty finding → EventBridge → Lambda (isolate instance)      │
│ → Automate responses to AWS events                                │
│                                                                       │
│ Pattern 3: Scheduled Tasks (replace cron)                           │
│ EventBridge Scheduler → Lambda (daily report)                     │
│                       → Lambda (cleanup old resources)            │
│                       → Step Functions (nightly batch)            │
│ → Serverless cron with precise scheduling                        │
│                                                                       │
│ Pattern 4: Cross-Account Event Hub                                  │
│ Account A (orders) ──┐                                              │
│ Account B (payments)─┼─▶ Central Event Bus → Consumers           │
│ Account C (shipping) ┘   (account D)                                │
│ → Centralized event routing across organization                  │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Use custom event bus for app events (not default bus)         │
│ 2. Enable schema discovery for documentation                     │
│ 3. Archive events for replay/debugging                            │
│ 4. Use input transformer to shape target input                   │
│ 5. Set up DLQ for each rule target                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
EventBridge Quick Reference:
├── Event Bus: Channel for events (default, custom, partner)
├── Rules: Pattern match → route to targets (up to 5)
├── AWS events on default bus: FREE
├── Custom events: $1/million events
├── Event patterns: Exact, prefix, suffix, numeric, wildcard
├── Input transformer: Reshape event before target delivery
├── Archive: Store events, replay later ($0.10/GB)
├── Schema Registry: Auto-discover event structure
├── Pipes: Point-to-point (source → filter → enrich → target)
├── Scheduler: Cron/rate schedules (replaces CW Events schedules)
├── Targets: Lambda, SQS, SNS, Step Functions, API GW, ECS, etc.
├── ⚡ Use EventBridge for event-driven architecture
├── ⚡ Use SNS for simple fan-out
└── ⚡ Always archive events for replay capability
```

---

## What's Next?

In **Chapter 48: Step Functions**, we'll cover serverless workflow orchestration, state machines, the Workflow Studio visual designer, and common orchestration patterns.
