# Chapter 48: AWS Step Functions

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Step Functions Fundamentals](#part-1-step-functions-fundamentals)
- [Part 2: Creating a State Machine (Full Portal Walkthrough)](#part-2-creating-a-state-machine-full-portal-walkthrough)
- [Part 3: State Types & ASL](#part-3-state-types--asl)
- [Part 4: Advanced Features](#part-4-advanced-features)
- [Part 5: Terraform & CLI Examples](#part-5-terraform--cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Workflow Orchestration? Why Do We Need Step Functions?

Imagine you're baking a cake. The recipe has steps that must happen **in order**: mix ingredients → pour into pan → bake for 30 minutes → check if done → if not done, bake 5 more minutes → cool down → add frosting.

**Step Functions lets you build these kinds of multi-step workflows for your applications.** Each step can be a Lambda function, an ECS task, an API call, or a human approval. Step Functions handles the flow: what runs next, what to do if a step fails, how to retry, and how to run things in parallel.

**Without Step Functions:** You'd chain Lambda functions together with complex error handling code, retry logic, and state tracking — messy and fragile.

**With Step Functions:** You draw a visual flowchart (or write JSON), and AWS executes it reliably.

**Simple real-world examples:**
- 🛨️ Order processing: Validate order → charge payment → if fails, retry 3 times → update inventory → send confirmation
- 💳 Account opening: Submit application → identity verification → wait for human approval → create account
- 🎥 Video processing: Upload → extract audio → generate thumbnails → transcode to multiple resolutions (all in parallel)

AWS Step Functions is a serverless orchestration service that lets you build workflows by combining AWS services. You define workflows as state machines using the Amazon States Language (ASL) or the visual Workflow Studio.

```
What you'll learn:
├── Step Functions fundamentals (state machines, types)
├── Creating a state machine (Workflow Studio walkthrough)
├── State types (Task, Choice, Parallel, Map, Wait, etc.)
├── Amazon States Language (ASL) syntax
├── Standard vs Express workflows
└── Terraform & CLI examples
```

---

## Part 1: Step Functions Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           HOW STEP FUNCTIONS WORKS                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Input → State Machine → State 1 → State 2 → ... → Output         │
│                                                                       │
│ ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐       │
│ │  Start   │──▶│ Validate │──▶│ Process  │──▶│ Notify   │       │
│ │          │   │ Order    │   │ Payment  │   │ Customer │       │
│ └──────────┘   └──────────┘   └──────────┘   └──────────┘       │
│                     │                              │               │
│                     ▼ (failure)                     ▼               │
│               ┌──────────┐                   ┌──────────┐       │
│               │ Reject   │                   │   End    │       │
│               │ Order    │                   │          │       │
│               └──────────┘                   └──────────┘       │
│                                                                       │
│ Workflow types:                                                      │
│ ┌──────────────────────┬───────────────┬─────────────────────┐   │
│ │                      │ Standard      │ Express              │   │
│ ├──────────────────────┼───────────────┼─────────────────────┤   │
│ │ Duration             │ Up to 1 year  │ Up to 5 minutes      │   │
│ │ Execution model      │ Exactly-once  │ At-least-once/       │   │
│ │                      │               │ At-most-once         │   │
│ │ History              │ Yes (console) │ CloudWatch Logs only │   │
│ │ Pricing              │ Per transition│ Per request+duration │   │
│ │ Cost                 │ $0.025/1K     │ ~$1/million (cheap)  │   │
│ │                      │ transitions   │                      │   │
│ │ Use case             │ Long-running  │ High-volume,         │   │
│ │                      │ orchestration │ short-duration        │   │
│ └──────────────────────┴───────────────┴─────────────────────┘   │
│                                                                       │
│ Supported integrations (200+ AWS services):                        │
│ ├── Lambda (invoke function)                                     │
│ ├── DynamoDB (GetItem, PutItem, Query, etc.)                   │
│ ├── SQS (SendMessage)                                            │
│ ├── SNS (Publish)                                                │
│ ├── ECS (RunTask)                                                │
│ ├── Batch (SubmitJob)                                            │
│ ├── Glue (StartJobRun)                                          │
│ ├── SageMaker (CreateTrainingJob)                               │
│ ├── API Gateway (Invoke)                                         │
│ └── Any AWS SDK action (SDK integrations)                       │
│                                                                       │
│ Integration patterns:                                                │
│ ├── Request/Response: Call service, get response immediately    │
│ ├── Run a Job (.sync): Wait for job to complete                │
│ ├── Wait for Callback (.waitForTaskToken): Wait for external  │
│ │   system to call back with a task token                      │
│ └── Optimized: Direct SDK integration (no Lambda needed)       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a State Machine (Full Portal Walkthrough)

```
Console → Step Functions → Create state machine

┌─────────────────────────────────────────────────────────────────┐
│           CHOOSE AUTHORING METHOD                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ● Design your workflow visually                               │
│   → Drag-and-drop Workflow Studio                            │
│   → ⚡ Recommended for getting started                       │
│                                                                   │
│ ○ Write your workflow in code                                  │
│   → Write ASL (Amazon States Language) JSON directly         │
│                                                                   │
│ ○ Start from a template                                        │
│   → Pre-built patterns (e.g., process S3 objects)           │
│                                                                   │
│ Type:                                                           │
│ ● Standard                                                     │
│   → Long-running workflows (up to 1 year)                   │
│   → Exactly-once execution                                   │
│   → Visual execution history                                 │
│                                                                   │
│ ○ Express                                                      │
│   → High-volume, short-duration (up to 5 min)               │
│   → At-least-once or at-most-once                            │
│   → Logs to CloudWatch only                                  │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           WORKFLOW STUDIO (Visual Designer)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Left panel (drag states onto canvas):                         │
│                                                                   │
│ Actions:                                                       │
│ ├── AWS Lambda: Invoke                                        │
│ ├── Amazon DynamoDB: GetItem / PutItem / DeleteItem          │
│ ├── Amazon SQS: SendMessage                                  │
│ ├── Amazon SNS: Publish                                      │
│ ├── Amazon ECS/Fargate: RunTask                              │
│ ├── AWS Batch: SubmitJob                                     │
│ ├── Amazon Bedrock: InvokeModel                              │
│ ├── AWS Glue: StartJobRun                                    │
│ └── ... (200+ services)                                      │
│                                                                   │
│ Flow:                                                          │
│ ├── Choice: If/else branching                                │
│ ├── Parallel: Run branches concurrently                      │
│ ├── Map: Iterate over array                                  │
│ ├── Wait: Delay execution                                    │
│ ├── Pass: Transform data                                     │
│ ├── Succeed: End with success                                │
│ └── Fail: End with error                                     │
│                                                                   │
│ Canvas:                                                        │
│ ┌──────────────────────────────────────────┐                  │
│ │ Start                                     │                  │
│ │   ↓                                       │                  │
│ │ [Validate Order] (Lambda)                 │                  │
│ │   ↓                                       │                  │
│ │ [Choice: Valid?]                           │                  │
│ │   ├── Yes → [Process Payment] (Lambda)   │                  │
│ │   │          ↓                             │                  │
│ │   │         [Update DB] (DynamoDB)        │                  │
│ │   │          ↓                             │                  │
│ │   │         [Send Email] (SNS)            │                  │
│ │   │          ↓                             │                  │
│ │   │         End                            │                  │
│ │   └── No → [Fail: Invalid Order]          │                  │
│ └──────────────────────────────────────────┘                  │
│                                                                   │
│ Each state configuration (right panel):                       │
│ ├── API parameters (function ARN, table, etc.)              │
│ ├── Input/Output processing (InputPath, OutputPath)         │
│ ├── Result selector (pick fields from result)               │
│ ├── Error handling (Retry, Catch)                            │
│ └── Wait/timeout settings                                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STATE MACHINE SETTINGS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ State machine name: [order-processing-workflow]               │
│                                                                   │
│ Permissions:                                                   │
│ ● Create new role (⚡ auto-creates with needed permissions)  │
│ ○ Choose an existing role                                     │
│ ○ Enter a role ARN                                            │
│                                                                   │
│ Logging:                                                       │
│ Log level: [ALL ▼]                                            │
│ ├── ALL: Log all events                                      │
│ ├── ERROR: Log errors only                                   │
│ ├── FATAL: Log fatal errors only                             │
│ └── OFF: No logging                                           │
│ CloudWatch log group: [/aws/states/order-processing ▼]       │
│ Include execution data: ☑                                    │
│ → ⚡ Enable ALL logging for debugging                        │
│                                                                   │
│ Tracing:                                                       │
│ ☑ Enable X-Ray tracing                                       │
│ → Visual map of all service calls                            │
│                                                                   │
│ Tags:                                                          │
│ [Environment] = [Production]                                  │
│                                                                   │
│                    [Create state machine]                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: State Types & ASL

```
┌─────────────────────────────────────────────────────────────────────┐
│           STATE TYPES                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ 1. Task: Execute work (Lambda, SDK, Activity)                      │
│ "ValidateOrder": {                                                  │
│   "Type": "Task",                                                   │
│   "Resource": "arn:aws:lambda:us-east-1:123456:function:validate",│
│   "Next": "ProcessPayment",                                        │
│   "Retry": [{                                                       │
│     "ErrorEquals": ["States.TaskFailed"],                          │
│     "IntervalSeconds": 3,                                           │
│     "MaxAttempts": 2,                                               │
│     "BackoffRate": 2                                                │
│   }],                                                                │
│   "Catch": [{                                                       │
│     "ErrorEquals": ["States.ALL"],                                  │
│     "Next": "HandleError"                                           │
│   }]                                                                 │
│ }                                                                     │
│                                                                       │
│ 2. Choice: Conditional branching                                    │
│ "CheckStatus": {                                                    │
│   "Type": "Choice",                                                 │
│   "Choices": [                                                       │
│     {                                                                 │
│       "Variable": "$.status",                                       │
│       "StringEquals": "approved",                                   │
│       "Next": "ProcessOrder"                                        │
│     },                                                                │
│     {                                                                 │
│       "Variable": "$.amount",                                       │
│       "NumericGreaterThan": 1000,                                   │
│       "Next": "ManualReview"                                        │
│     }                                                                 │
│   ],                                                                  │
│   "Default": "RejectOrder"                                          │
│ }                                                                     │
│                                                                       │
│ 3. Parallel: Run branches concurrently                              │
│ "ProcessAll": {                                                      │
│   "Type": "Parallel",                                               │
│   "Branches": [                                                      │
│     { "StartAt": "SendEmail", "States": {...} },                  │
│     { "StartAt": "UpdateDB", "States": {...} },                   │
│     { "StartAt": "GenerateReport", "States": {...} }              │
│   ],                                                                  │
│   "Next": "Done"                                                    │
│ }                                                                     │
│                                                                       │
│ 4. Map: Iterate over array (for-each loop)                         │
│ "ProcessItems": {                                                    │
│   "Type": "Map",                                                    │
│   "ItemsPath": "$.orderItems",                                     │
│   "MaxConcurrency": 10,                                             │
│   "ItemProcessor": {                                                 │
│     "ProcessorConfig": {                                             │
│       "Mode": "INLINE"  (or "DISTRIBUTED" for large datasets)    │
│     },                                                                │
│     "StartAt": "ProcessItem",                                       │
│     "States": {                                                       │
│       "ProcessItem": {                                               │
│         "Type": "Task",                                              │
│         "Resource": "arn:aws:lambda:...:process-item",             │
│         "End": true                                                  │
│       }                                                               │
│     }                                                                 │
│   },                                                                  │
│   "Next": "Done"                                                    │
│ }                                                                     │
│                                                                       │
│ 5. Wait: Delay execution                                            │
│ "WaitForApproval": {                                                │
│   "Type": "Wait",                                                   │
│   "Seconds": 300,          // Fixed delay                          │
│   // OR                                                              │
│   "Timestamp": "2024-03-15T08:00:00Z",  // Specific time         │
│   // OR                                                              │
│   "TimestampPath": "$.scheduledTime",    // From input            │
│   "Next": "CheckApproval"                                           │
│ }                                                                     │
│                                                                       │
│ 6. Pass: Transform data / inject data                               │
│ "SetDefaults": {                                                     │
│   "Type": "Pass",                                                   │
│   "Result": {"status": "pending", "retryCount": 0},               │
│   "ResultPath": "$.defaults",                                      │
│   "Next": "ProcessOrder"                                            │
│ }                                                                     │
│                                                                       │
│ 7. Succeed / Fail: Terminal states                                  │
│ "OrderComplete": { "Type": "Succeed" }                              │
│ "OrderFailed": {                                                     │
│   "Type": "Fail",                                                   │
│   "Error": "OrderValidationError",                                  │
│   "Cause": "Invalid order data"                                     │
│ }                                                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Advanced Features

```
┌─────────────────────────────────────────────────────────────────────┐
│           ADVANCED FEATURES                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Error handling (Retry + Catch):                                     │
│ "Retry": [{                                                         │
│   "ErrorEquals": ["States.TaskFailed", "Lambda.ServiceException"],│
│   "IntervalSeconds": 2,                                             │
│   "MaxAttempts": 3,                                                  │
│   "BackoffRate": 2,                                                  │
│   "JitterStrategy": "FULL"                                         │
│ }]                                                                    │
│ → Retries: 2s, 4s, 8s (with jitter)                               │
│                                                                       │
│ "Catch": [{                                                          │
│   "ErrorEquals": ["States.ALL"],                                    │
│   "ResultPath": "$.error",                                          │
│   "Next": "ErrorHandler"                                            │
│ }]                                                                    │
│ → Catches unrecoverable errors                                     │
│                                                                       │
│ Input/Output processing:                                             │
│ ├── InputPath: Select portion of input ($.order)                │
│ ├── Parameters: Construct input for task                         │
│ ├── ResultSelector: Select portion of task result               │
│ ├── ResultPath: Where to put result in state input             │
│ └── OutputPath: Select portion of combined output               │
│                                                                       │
│ Callback pattern (.waitForTaskToken):                               │
│ ├── Step Functions pauses and generates a task token            │
│ ├── Token sent to external system (SQS, human approval)       │
│ ├── External system calls SendTaskSuccess/SendTaskFailure      │
│ └── Workflow resumes with callback result                       │
│                                                                       │
│ Distributed Map:                                                     │
│ ├── Process millions of items from S3/CSV/JSON                  │
│ ├── Up to 10,000 concurrent child executions                   │
│ ├── Results written to S3                                        │
│ └── ⚡ Use for large-scale data processing                     │
│                                                                       │
│ Execution:                                                           │
│ Console → Step Functions → State machine → Start execution       │
│ Input: {"orderId": "12345", "items": [...]}                        │
│ → Visual execution graph shows each state in real-time          │
│ → Green = success, Red = failure, Blue = in progress            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Terraform & CLI Examples

```hcl
resource "aws_sfn_state_machine" "order" {
  name     = "order-processing"
  role_arn = aws_iam_role.step_functions.arn
  type     = "STANDARD"

  definition = jsonencode({
    Comment = "Order processing workflow"
    StartAt = "ValidateOrder"
    States = {
      ValidateOrder = {
        Type     = "Task"
        Resource = aws_lambda_function.validate.arn
        Next     = "CheckValid"
        Retry = [{
          ErrorEquals     = ["States.TaskFailed"]
          IntervalSeconds = 2
          MaxAttempts     = 3
          BackoffRate     = 2
        }]
        Catch = [{
          ErrorEquals = ["States.ALL"]
          Next        = "OrderFailed"
        }]
      }
      CheckValid = {
        Type = "Choice"
        Choices = [{
          Variable     = "$.valid"
          BooleanEquals = true
          Next         = "ProcessPayment"
        }]
        Default = "OrderFailed"
      }
      ProcessPayment = {
        Type     = "Task"
        Resource = aws_lambda_function.payment.arn
        Next     = "NotifyCustomer"
      }
      NotifyCustomer = {
        Type     = "Task"
        Resource = "arn:aws:states:::sns:publish"
        Parameters = {
          TopicArn = aws_sns_topic.orders.arn
          "Message.$" = "$.notification"
        }
        Next = "OrderComplete"
      }
      OrderComplete = { Type = "Succeed" }
      OrderFailed = {
        Type  = "Fail"
        Error = "OrderFailed"
        Cause = "Order validation or processing failed"
      }
    }
  })

  logging_configuration {
    log_destination        = "${aws_cloudwatch_log_group.sfn.arn}:*"
    include_execution_data = true
    level                  = "ALL"
  }

  tracing_configuration {
    enabled = true
  }
}
```

```bash
# Start execution
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:123456:stateMachine:order-processing \
  --input '{"orderId":"12345","items":[{"id":"A","qty":2}]}'

# List executions
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:us-east-1:123456:stateMachine:order-processing \
  --status-filter RUNNING

# Describe execution
aws stepfunctions describe-execution \
  --execution-arn arn:aws:states:us-east-1:123456:execution:order-processing:exec-id

# Send task success (callback pattern)
aws stepfunctions send-task-success \
  --task-token "token-from-sqs" \
  --output '{"approved": true}'
```

---

## Part 6: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Order Processing (Standard)                               │
│ Validate → Check Inventory → Process Payment → Ship → Notify     │
│ ├── Choice states for error handling at each step               │
│ ├── Retry with exponential backoff                               │
│ ├── Catch → compensation (refund, restock)                      │
│ └── Human approval for high-value orders (callback)             │
│                                                                       │
│ Pattern 2: Data Pipeline (Standard + Distributed Map)              │
│ List S3 Objects → Distributed Map (process each file)            │
│                 → Aggregate Results → Store in DynamoDB          │
│ ├── Process millions of files in parallel                       │
│ ├── Automatic retry for failed items                             │
│ └── Results summary in S3                                        │
│                                                                       │
│ Pattern 3: API Orchestration (Express)                               │
│ API Gateway → Step Functions Express                               │
│ → Parallel: [Get User, Get Orders, Get Recommendations]          │
│ → Merge Results → Return to API                                   │
│ ├── Low latency (~100ms)                                         │
│ ├── Synchronous execution                                        │
│ └── No Lambda needed for DynamoDB/S3 calls                      │
│                                                                       │
│ Pattern 4: Human Approval Workflow                                   │
│ Submit Request → Send Approval Email (SNS)                        │
│ → Wait for Callback (task token) → Approved?                     │
│ → Yes: Execute Change → No: Reject                                │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Use Workflow Studio for initial design                        │
│ 2. Enable logging (ALL level) in production                      │
│ 3. Add Retry to every Task state                                  │
│ 4. Use SDK integrations instead of Lambda when possible         │
│ 5. Express for high-volume, Standard for long-running           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Step Functions Quick Reference:
├── Standard: Up to 1 year, exactly-once, $0.025/1K transitions
├── Express: Up to 5 min, at-least-once, ~$1/million executions
├── States: Task, Choice, Parallel, Map, Wait, Pass, Succeed, Fail
├── Integrations: 200+ AWS services (Lambda, DynamoDB, SQS, etc.)
├── Integration patterns: Request/Response, .sync, .waitForTaskToken
├── Error handling: Retry (backoff) + Catch (fallback state)
├── Input/Output: InputPath, Parameters, ResultSelector, ResultPath, OutputPath
├── Distributed Map: Process millions of items (S3, CSV, JSON)
├── Callback: Pause workflow, wait for external system
├── Workflow Studio: Visual drag-and-drop designer
├── ⚡ SDK integrations eliminate Lambda for simple operations
├── ⚡ Always add Retry + Catch to Task states
└── ⚡ Enable X-Ray tracing for debugging
```

---

## What's Next?

In **Chapter 49: API Gateway**, we'll cover REST APIs, HTTP APIs, WebSocket APIs, stages, authorizers, throttling, and API management.
