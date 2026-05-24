# Chapter 49 — Workflows

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Workflows Fundamentals](#part-1--workflows-fundamentals)
- [Part 2: Workflow Syntax & Structure](#part-2--workflow-syntax--structure)
- [Part 3: Steps & Expressions](#part-3--steps--expressions)
- [Part 4: Variables & Data Manipulation](#part-4--variables--data-manipulation)
- [Part 5: HTTP Requests & API Calls](#part-5--http-requests--api-calls)
- [Part 6: Conditional Logic & Switches](#part-6--conditional-logic--switches)
- [Part 7: Loops & Iteration](#part-7--loops--iteration)
- [Part 8: Error Handling & Retries](#part-8--error-handling--retries)
- [Part 9: Subworkflows & Reusability](#part-9--subworkflows--reusability)
- [Part 10: Connectors — Google Cloud Services](#part-10--connectors--google-cloud-services)
- [Part 11: Parallel Execution](#part-11--parallel-execution)
- [Part 12: Callbacks & Long-Running Operations](#part-12--callbacks--long-running-operations)
- [Part 13: Triggering Workflows](#part-13--triggering-workflows)
- [Part 14: Console Walkthrough — Workflow Creation & Execution](#part-14-console-walkthrough--workflow-creation--execution)
- [Part 15: Terraform & gcloud CLI Reference](#part-15--terraform--gcloud-cli-reference)
- [Part 16: Real-World Patterns](#part-16--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Cloud Workflows is a fully managed orchestration service that executes services in an order you define — a workflow. Workflows can combine Google Cloud services (Cloud Functions, Cloud Run, BigQuery, etc.), HTTP-based APIs, and other workflows into serverless, reliable, multi-step pipelines. Workflows are written in YAML or JSON, support conditional logic, loops, error handling, parallel execution, and human-in-the-loop callbacks.

---

## Part 1 — Workflows Fundamentals

### What Is Workflows?

```
┌─────────────────────────────────────────────────────────────────┐
│              WORKFLOWS OVERVIEW                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Orchestrate multi-step processes as code:                      │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Workflow: "order-processing"                             │   │
│  │                                                           │   │
│  │  Step 1: Validate order ──► Cloud Function               │   │
│  │       │                                                   │   │
│  │  Step 2: Check inventory ──► Cloud Run API               │   │
│  │       │                                                   │   │
│  │  Step 3: if available?                                    │   │
│  │       ├── YES: Process payment ──► Stripe API            │   │
│  │       └── NO: Notify customer ──► Cloud Tasks            │   │
│  │       │                                                   │   │
│  │  Step 4: Send confirmation ──► Email service             │   │
│  │       │                                                   │   │
│  │  Step 5: Update BigQuery ──► BigQuery connector          │   │
│  │       │                                                   │   │
│  │  DONE: return order status                                │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  Key features:                                                   │
│  • Serverless — no infrastructure to manage                    │
│  • Durable — state is persisted automatically                  │
│  • Built-in retries and error handling                          │
│  • Connectors for 100+ Google Cloud APIs                       │
│  • Parallel step execution                                      │
│  • Callbacks for human approval / external events              │
│  • Execution can run up to 1 year                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Feature | GCP Workflows | AWS Step Functions | Azure Logic Apps |
|---------|--------------|-------------------|-----------------|
| Service | Workflows | Step Functions | Logic Apps |
| Definition | YAML / JSON | ASL (JSON) | JSON (ARM) / Designer |
| Visual editor | No (code only) | Yes (Workflow Studio) | Yes (Designer) |
| Connectors | 100+ GCP services | 200+ AWS services | 400+ connectors |
| Parallel execution | Yes | Yes | Yes |
| Human approval | Yes (callbacks) | Yes (activity tasks) | Yes |
| Max duration | 1 year | 1 year (Standard) | Unlimited |
| Error handling | try/except/retry | Catch/Retry | Scope + run-after |
| Pricing | Per step + external calls | Per state transition | Per action/trigger |
| Serverless | Yes | Yes | Yes |

### Pricing

| Component | Cost |
|-----------|------|
| Internal steps (assign, switch, etc.) | $0.01 per 1,000 steps |
| External steps (HTTP calls, connectors) | $0.025 per 1,000 steps |
| Free tier | 5,000 internal + 2,000 external steps/month |
| Max execution duration | 1 year |

---

## Part 2 — Workflow Syntax & Structure

### Basic Workflow Structure

```yaml
# Workflow definition (YAML)
main:
  params: [input]
  steps:
    - step1_name:
        # step definition
    - step2_name:
        # step definition
    - return_result:
        return: ${result}
```

### Hello World Example

```yaml
main:
  params: [args]
  steps:
    - greet:
        assign:
          - greeting: ${"Hello, " + args.name + "!"}
    - return_greeting:
        return: ${greeting}
```

```bash
# Deploy workflow
gcloud workflows deploy hello-workflow \
    --source=hello.yaml \
    --location=us-central1

# Execute workflow
gcloud workflows run hello-workflow \
    --location=us-central1 \
    --data='{"name": "World"}'
```

---

## Part 3 — Steps & Expressions

### Step Types

```
┌──────────────────────────────────────────────────────────────┐
│         STEP TYPES                                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. assign — set variables                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  - set_vars:                                          │    │
│  │      assign:                                          │    │
│  │        - name: "Alice"                                │    │
│  │        - count: 42                                    │    │
│  │        - items: ["a", "b", "c"]                       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  2. call — HTTP request or connector call                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  - call_api:                                          │    │
│  │      call: http.post                                  │    │
│  │      args:                                            │    │
│  │        url: https://api.example.com/orders            │    │
│  │        body:                                          │    │
│  │          orderId: ${orderId}                          │    │
│  │      result: apiResponse                              │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  3. switch — conditional branching                           │
│  4. for — iteration / loops                                  │
│  5. try / except — error handling                            │
│  6. parallel — concurrent execution                          │
│  7. return — end workflow with result                        │
│  8. raise — throw an error                                   │
│  9. next — jump to another step                              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Expressions

```yaml
# Expressions are enclosed in ${}
main:
  params: [args]
  steps:
    - compute:
        assign:
          # String concatenation
          - fullName: ${args.firstName + " " + args.lastName}
          # Arithmetic
          - total: ${args.price * args.quantity}
          # Built-in functions
          - today: ${time.format(sys.now())}
          - encoded: ${base64.encode(text.encode("hello"))}
          - length: ${len(args.items)}
          # Map access
          - email: ${args.user.email}
          # List access
          - first: ${args.items[0]}
          # Type checking
          - isString: ${text.match_regex(args.input, "^[a-z]+$")}
```

---

## Part 4 — Variables & Data Manipulation

### Working with Data

```yaml
main:
  params: [input]
  steps:
    - init:
        assign:
          # Maps (objects)
          - order:
              id: ${input.orderId}
              status: "pending"
              items: ${input.items}
              total: 0.0

          # Lists
          - results: []
          - numbers: [1, 2, 3, 4, 5]

    - calculate_total:
        assign:
          - order.total: ${input.price * input.quantity}
          - order.status: "calculated"

    - add_to_results:
        assign:
          - results: ${list.concat(results, [order])}

    - build_response:
        assign:
          - response:
              success: true
              order: ${order}
              processedAt: ${sys.now()}

    - done:
        return: ${response}
```

---

## Part 5 — HTTP Requests & API Calls

### Making HTTP Requests

```yaml
main:
  steps:
    # GET request
    - fetch_data:
        call: http.get
        args:
          url: https://api.example.com/data
          headers:
            Accept: application/json
          query:
            limit: 10
            page: 1
          timeout: 30
        result: getResponse

    # POST request
    - create_order:
        call: http.post
        args:
          url: https://api.example.com/orders
          headers:
            Content-Type: application/json
          body:
            customerId: "cust-123"
            items:
              - productId: "prod-456"
                quantity: 2
          auth:
            type: OIDC
            audience: "https://api.example.com"
        result: postResponse

    # Access response data
    - process_response:
        assign:
          - statusCode: ${getResponse.code}
          - body: ${getResponse.body}
          - orderId: ${postResponse.body.orderId}

    - return_result:
        return:
          status: ${statusCode}
          orderId: ${orderId}
```

### Authentication for HTTP Calls

```yaml
# OIDC token (for Cloud Run, Cloud Functions)
- call_cloud_run:
    call: http.post
    args:
      url: https://my-service.run.app/process
      auth:
        type: OIDC
        audience: "https://my-service.run.app"

# OAuth2 token (for Google APIs)
- call_google_api:
    call: http.get
    args:
      url: https://storage.googleapis.com/storage/v1/b/my-bucket/o
      auth:
        type: OAuth2
```

---

## Part 6 — Conditional Logic & Switches

### Switch Statements

```yaml
main:
  params: [input]
  steps:
    - evaluate:
        switch:
          - condition: ${input.amount > 1000}
            next: high_value_order
          - condition: ${input.amount > 100}
            next: standard_order
          - condition: ${input.amount <= 100}
            next: small_order

    - high_value_order:
        call: http.post
        args:
          url: https://my-service.run.app/review
          body:
            orderId: ${input.orderId}
            type: "manual_review"
        next: finalize

    - standard_order:
        call: http.post
        args:
          url: https://my-service.run.app/auto-process
          body:
            orderId: ${input.orderId}
        next: finalize

    - small_order:
        call: http.post
        args:
          url: https://my-service.run.app/express
          body:
            orderId: ${input.orderId}
        next: finalize

    - finalize:
        return: "Order processed"
```

---

## Part 7 — Loops & Iteration

### For Loops

```yaml
main:
  params: [input]
  steps:
    - init:
        assign:
          - results: []

    # Iterate over a list
    - process_items:
        for:
          value: item
          in: ${input.items}
          steps:
            - process_one:
                call: http.post
                args:
                  url: https://my-service.run.app/process-item
                  body:
                    itemId: ${item.id}
                    quantity: ${item.quantity}
                result: itemResult

            - collect:
                assign:
                  - results: ${list.concat(results, [itemResult.body])}

    # Iterate with index
    - numbered_loop:
        for:
          value: item
          index: i
          in: ${input.items}
          steps:
            - log_item:
                call: sys.log
                args:
                  text: ${"Processing item " + string(i) + ": " + item.name}

    - done:
        return: ${results}
```

### Range-Based Loops

```yaml
    # Loop with range (0 to N-1)
    - count_loop:
        for:
          value: i
          range: [0, 9]  # 0, 1, 2, ..., 9
          steps:
            - log_number:
                call: sys.log
                args:
                  text: ${"Iteration " + string(i)}
```

---

## Part 8 — Error Handling & Retries

### Try / Except / Retry

```yaml
main:
  steps:
    # Try with automatic retry
    - call_flaky_api:
        try:
          call: http.post
          args:
            url: https://flaky-api.example.com/process
            body:
              data: "important"
          result: apiResult
        retry:
          predicate: ${default_retry_predicate}
          max_retries: 5
          backoff:
            initial_delay: 1
            max_delay: 60
            multiplier: 2

    # Try / except for custom error handling
    - risky_operation:
        try:
          steps:
            - call_api:
                call: http.post
                args:
                  url: https://api.example.com/charge
                  body:
                    amount: ${amount}
                result: chargeResult
        except:
          as: e
          steps:
            - check_error:
                switch:
                  - condition: ${e.code == 404}
                    next: not_found_handler
                  - condition: ${e.code == 429}
                    next: rate_limited_handler
                  - condition: true
                    next: generic_error_handler

    - not_found_handler:
        return:
          error: "Resource not found"
          code: 404

    - rate_limited_handler:
        call: sys.sleep
        args:
          seconds: 30
        next: risky_operation  # retry after sleep

    - generic_error_handler:
        raise: ${e}  # re-raise the error
```

### Custom Retry Predicates

```yaml
    - custom_retry:
        try:
          call: http.get
          args:
            url: https://api.example.com/data
          result: response
        retry:
          predicate: ${custom_predicate}
          max_retries: 3
          backoff:
            initial_delay: 2
            max_delay: 30
            multiplier: 2

# Custom predicate subworkflow
custom_predicate:
  params: [e]
  steps:
    - check:
        switch:
          - condition: ${e.code == 429}
            return: true    # retry on rate limit
          - condition: ${e.code >= 500}
            return: true    # retry on server errors
          - condition: true
            return: false   # don't retry on other errors
```

---

## Part 9 — Subworkflows & Reusability

### Defining Subworkflows

```yaml
# Main workflow calls subworkflows
main:
  params: [input]
  steps:
    - validate:
        call: validate_order
        args:
          order: ${input.order}
        result: validationResult

    - process:
        call: process_payment
        args:
          orderId: ${input.order.id}
          amount: ${input.order.total}
        result: paymentResult

    - done:
        return:
          validation: ${validationResult}
          payment: ${paymentResult}

# Subworkflow definitions (in same file)
validate_order:
  params: [order]
  steps:
    - check_required:
        switch:
          - condition: ${order.items == null OR len(order.items) == 0}
            raise: "Order must have at least one item"
    - valid:
        return:
          valid: true
          itemCount: ${len(order.items)}

process_payment:
  params: [orderId, amount]
  steps:
    - charge:
        call: http.post
        args:
          url: https://payments.example.com/charge
          body:
            orderId: ${orderId}
            amount: ${amount}
        result: chargeResult
    - done:
        return: ${chargeResult.body}
```

---

## Part 10 — Connectors — Google Cloud Services

### Built-in Connectors

```
┌──────────────────────────────────────────────────────────────┐
│         CONNECTORS                                             │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Connectors simplify calling Google Cloud APIs:              │
│  • Type-safe, no raw HTTP calls needed                       │
│  • Automatic authentication (workflow's SA)                  │
│  • Handle pagination and long-running operations             │
│                                                                │
│  Popular connectors:                                           │
│  ├── googleapis.com/bigquery/v2                              │
│  ├── googleapis.com/storage/v1                               │
│  ├── googleapis.com/compute/v1                               │
│  ├── googleapis.com/run/v2                                   │
│  ├── googleapis.com/cloudfunctions/v2                        │
│  ├── googleapis.com/firestore/v1                             │
│  ├── googleapis.com/secretmanager/v1                         │
│  ├── googleapis.com/cloudtasks/v2                            │
│  └── googleapis.com/pubsub/v1                               │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```yaml
main:
  steps:
    # BigQuery connector — run a query
    - query_bq:
        call: googleapis.bigquery.v2.jobs.query
        args:
          projectId: ${sys.get_env("GOOGLE_CLOUD_PROJECT_ID")}
          body:
            query: "SELECT COUNT(*) as total FROM dataset.orders WHERE date = CURRENT_DATE()"
            useLegacySql: false
        result: bqResult

    # Cloud Storage connector — list objects
    - list_objects:
        call: googleapis.storage.v1.objects.list
        args:
          bucket: "my-data-bucket"
          prefix: "exports/"
        result: objects

    # Secret Manager connector — access secret
    - get_secret:
        call: googleapis.secretmanager.v1.projects.secrets.versions.accessString
        args:
          secret_id: "api-key"
          project_id: ${sys.get_env("GOOGLE_CLOUD_PROJECT_ID")}
        result: apiKey

    # Pub/Sub connector — publish message
    - publish:
        call: googleapis.pubsub.v1.projects.topics.publish
        args:
          topic: ${"projects/" + sys.get_env("GOOGLE_CLOUD_PROJECT_ID") + "/topics/notifications"}
          body:
            messages:
              - data: ${base64.encode(text.encode("Order completed"))}
```

---

## Part 11 — Parallel Execution

### Parallel Steps

```yaml
main:
  params: [input]
  steps:
    # Execute branches in parallel
    - parallel_processing:
        parallel:
          branches:
            - notify_branch:
                steps:
                  - send_email:
                      call: http.post
                      args:
                        url: https://email-service.run.app/send
                        body:
                          to: ${input.email}
                          subject: "Order confirmed"

            - inventory_branch:
                steps:
                  - update_inventory:
                      call: http.post
                      args:
                        url: https://inventory-service.run.app/update
                        body:
                          items: ${input.items}

            - analytics_branch:
                steps:
                  - log_order:
                      call: googleapis.bigquery.v2.jobs.query
                      args:
                        projectId: ${sys.get_env("GOOGLE_CLOUD_PROJECT_ID")}
                        body:
                          query: ${"INSERT INTO analytics.orders VALUES('" + input.orderId + "')"}

    # All branches must complete before next step
    - done:
        return: "All parallel tasks completed"
```

### Parallel For Loops

```yaml
    # Process items in parallel (with concurrency limit)
    - parallel_items:
        parallel:
          shared: [results]  # shared variables across iterations
          for:
            value: item
            in: ${input.items}
            steps:
              - process:
                  call: http.post
                  args:
                    url: https://my-service.run.app/process
                    body: ${item}
                  result: itemResult
              - collect:
                  assign:
                    - results: ${list.concat(results, [itemResult.body])}
```

---

## Part 12 — Callbacks & Long-Running Operations

### Callbacks (Human Approval)

```yaml
main:
  params: [input]
  steps:
    - create_callback:
        call: events.create_callback_endpoint
        args:
          http_callback_method: "POST"
        result: callbackDetails

    - send_approval_request:
        call: http.post
        args:
          url: https://slack-bot.run.app/request-approval
          body:
            message: ${"Approve order " + input.orderId + " for $" + string(input.amount)}
            callback_url: ${callbackDetails.url}
            approver: ${input.approverEmail}

    # Workflow PAUSES here until callback is received
    - wait_for_approval:
        call: events.await_callback
        args:
          callback: ${callbackDetails}
          timeout: 86400  # 24 hours
        result: approvalResponse

    - check_approval:
        switch:
          - condition: ${approvalResponse.http_request.body.approved == true}
            next: process_order
          - condition: true
            next: reject_order

    - process_order:
        call: http.post
        args:
          url: https://orders.run.app/process
          body:
            orderId: ${input.orderId}
        result: processResult
        next: done

    - reject_order:
        return:
          status: "rejected"
          reason: ${approvalResponse.http_request.body.reason}

    - done:
        return:
          status: "approved"
          result: ${processResult.body}
```

---

## Part 13 — Triggering Workflows

### Execution Methods

```
┌──────────────────────────────────────────────────────────────┐
│         TRIGGERING WORKFLOWS                                   │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  1. gcloud CLI                                                │
│  gcloud workflows run WORKFLOW --data='{"key":"value"}'      │
│                                                                │
│  2. Cloud Scheduler (recurring)                               │
│  Scheduler → HTTP POST → Workflows Execution API             │
│                                                                │
│  3. Eventarc (event-driven)                                   │
│  Cloud Storage event → Eventarc → Workflows                  │
│  Pub/Sub message → Eventarc → Workflows                      │
│                                                                │
│  4. REST API / Client Libraries                               │
│  POST https://workflowexecutions.googleapis.com/v1/...       │
│                                                                │
│  5. Cloud Functions / Cloud Run                               │
│  Code creates workflow execution via client library           │
│                                                                │
│  6. From another Workflow                                     │
│  Connector: googleapis.workflowexecutions.v1                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Execute workflow
gcloud workflows run order-processing \
    --location=us-central1 \
    --data='{"orderId":"12345","amount":99.99}'

# Execute and wait for result
gcloud workflows execute order-processing \
    --location=us-central1 \
    --data='{"orderId":"12345"}' \
    --call-log-level=LOG_ALL_STEPS

# Check execution status
gcloud workflows executions describe EXECUTION_ID \
    --workflow=order-processing \
    --location=us-central1

# List executions
gcloud workflows executions list \
    --workflow=order-processing \
    --location=us-central1
```

---

## Part 14: Console Walkthrough — Workflow Creation & Execution

### Creating a Workflow

```
Console → Workflows → CREATE

┌─────────────────────────────────────────────────────────────────┐
│           CREATE WORKFLOW                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Workflow name ──                                              │
│ Name: [order-processing-workflow]                               │
│ ⚡ Lowercase, hyphens only. Cannot be changed after creation.   │
│                                                                   │
│ ── Description ──                                                │
│ Description: [Processes incoming orders end-to-end]             │
│                                                                   │
│ ── Region ──                                                     │
│ Region: [us-central1 ▼]                                         │
│                                                                   │
│ ── Service account ──                                            │
│ Service account:                                                │
│   [workflow-sa@project.iam.gserviceaccount.com ▼]               │
│ ⚡ The workflow runs with this SA's permissions.                 │
│   Grant it access to services the workflow calls               │
│   (Cloud Run Invoker, Pub/Sub Publisher, etc.)                 │
│                                                                   │
│ ── Call log level ──                                             │
│ ○ None                                                         │
│ ● Log errors only                                              │
│ ○ Log all calls                                                 │
│                                                                   │
│ ── Encryption ──                                                 │
│ ● Google-managed encryption key                                │
│ ○ Customer-managed encryption key (CMEK)                       │
│                                                                   │
│                              [NEXT]                             │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│           DEFINE WORKFLOW (YAML editor)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ⚡ Inline YAML editor with syntax highlighting.                  │
│   You can write or paste your workflow definition here.        │
│                                                                   │
│ ┌──────────────────────────────────────────────────────────┐  │
│ │ main:                                                       │  │
│ │   steps:                                                    │  │
│ │     - init:                                                 │  │
│ │         assign:                                              │  │
│ │           - project_id: ${sys.get_env("GOOGLE_CLOUD_PROJ.")} │  │
│ │     - call_api:                                              │  │
│ │         call: http.get                                      │  │
│ │         args:                                                │  │
│ │           url: https://api.example.com/data                 │  │
│ │         result: api_response                                │  │
│ │     - return_result:                                         │  │
│ │         return: ${api_response.body}                         │  │
│ └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│                             [DEPLOY]                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Executing a Workflow

```
Console → Workflows → Click workflow name → EXECUTE

┌─────────────────────────────────────────────────────────────────┐
│           EXECUTE WORKFLOW                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Input (JSON) ──                                              │
│ ┌──────────────────────────────────────────────────────────┐  │
│ │ {                                                           │  │
│ │   "order_id": "ORD-12345",                                  │  │
│ │   "customer": "john@example.com",                            │  │
│ │   "amount": 99.99                                            │  │
│ │ }                                                           │  │
│ └──────────────────────────────────────────────────────────┘  │
│ ⚡ Input is passed as the workflow's runtime argument.           │
│   Access via: ${args.order_id}                                 │
│                                                                   │
│                            [EXECUTE]                            │
│                                                                   │
│ After execution:                                                │
│ ├── Status: ACTIVE / SUCCEEDED / FAILED                       │
│ ├── Output: JSON result from return step                      │
│ ├── Logs: Link to Cloud Logging                                │
│ └── Step details: Expand each step to see input/output         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 15 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Workflow ─────────────────────────────────────────────────
resource "google_workflows_workflow" "order_processing" {
  name            = "order-processing"
  region          = var.region
  project         = var.project_id
  description     = "Process incoming orders"
  service_account = google_service_account.workflow.id
  call_log_level  = "LOG_ERRORS_ONLY"

  source_contents = <<-YAML
    main:
      params: [input]
      steps:
        - validate:
            call: http.post
            args:
              url: https://validator.run.app/validate
              body: $${input}
              auth:
                type: OIDC
            result: validation

        - check:
            switch:
              - condition: $${validation.body.valid}
                next: process
              - condition: true
                raise: "Invalid order"

        - process:
            call: http.post
            args:
              url: https://processor.run.app/process
              body:
                orderId: $${input.orderId}
              auth:
                type: OIDC
            result: processResult

        - done:
            return: $${processResult.body}
  YAML
}

# ─── Service Account ─────────────────────────────────────────
resource "google_service_account" "workflow" {
  account_id   = "workflow-sa"
  display_name = "Workflows Service Account"
  project      = var.project_id
}

# Grant workflow SA permissions
resource "google_project_iam_member" "workflow_invoker" {
  project = var.project_id
  role    = "roles/run.invoker"
  member  = "serviceAccount:${google_service_account.workflow.email}"
}

resource "google_project_iam_member" "workflow_log" {
  project = var.project_id
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.workflow.email}"
}

# ─── Schedule Workflow via Cloud Scheduler ────────────────────
resource "google_cloud_scheduler_job" "trigger_workflow" {
  name     = "trigger-order-processing"
  schedule = "0 */6 * * *"
  region   = var.region
  project  = var.project_id

  http_target {
    http_method = "POST"
    uri         = "https://workflowexecutions.googleapis.com/v1/${google_workflows_workflow.order_processing.id}/executions"
    body = base64encode(jsonencode({
      argument = jsonencode({ source = "scheduler" })
    }))

    oauth_token {
      service_account_email = google_service_account.workflow.email
    }
  }
}

# ─── Eventarc Trigger ─────────────────────────────────────────
resource "google_eventarc_trigger" "storage_trigger" {
  name     = "storage-to-workflow"
  location = var.region
  project  = var.project_id

  matching_criteria {
    attribute = "type"
    value     = "google.cloud.storage.object.v1.finalized"
  }
  matching_criteria {
    attribute = "bucket"
    value     = google_storage_bucket.uploads.name
  }

  destination {
    workflow = google_workflows_workflow.order_processing.id
  }

  service_account = google_service_account.workflow.email
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# WORKFLOWS
# ═══════════════════════════════════════════════════════════════
gcloud workflows deploy NAME --source=FILE --location=LOC \
    --service-account=SA
gcloud workflows describe NAME --location=LOC
gcloud workflows list --location=LOC
gcloud workflows delete NAME --location=LOC

# ═══════════════════════════════════════════════════════════════
# EXECUTIONS
# ═══════════════════════════════════════════════════════════════
gcloud workflows run NAME --location=LOC --data='JSON'
gcloud workflows execute NAME --location=LOC --data='JSON'
gcloud workflows executions list --workflow=NAME --location=LOC
gcloud workflows executions describe EXEC_ID --workflow=NAME --location=LOC
gcloud workflows executions cancel EXEC_ID --workflow=NAME --location=LOC
```

---

## Part 15 — Real-World Patterns

### Pattern 1: ETL Data Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: SCHEDULED ETL PIPELINE                                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Cloud Scheduler (daily 2 AM) → triggers Workflow:                   │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Step 1: Extract — call 3 APIs in parallel               │        │
│  │  ┌─────────────┬───────────────┬──────────────┐         │        │
│  │  │ Salesforce  │ Stripe API    │ Internal DB  │         │        │
│  │  │ → GCS raw/  │ → GCS raw/    │ → GCS raw/   │         │        │
│  │  └─────────────┴───────────────┴──────────────┘         │        │
│  │                                                           │        │
│  │  Step 2: Transform — BigQuery SQL jobs                   │        │
│  │  → Clean and join data                                    │        │
│  │  → Write to BigQuery staging tables                       │        │
│  │                                                           │        │
│  │  Step 3: Validate — check row counts, data quality       │        │
│  │  → if validation fails → alert + callback for review     │        │
│  │                                                           │        │
│  │  Step 4: Load — promote staging → production tables      │        │
│  │                                                           │        │
│  │  Step 5: Notify — Pub/Sub "pipeline-complete" message    │        │
│  │  → Downstream services consume                            │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Error handling: retry each step 3x, alert on final failure          │
│  Logging: LOG_ALL_STEPS for audit trail                              │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Order Processing with Approval

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: ORDER WITH HUMAN APPROVAL                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  API → creates Workflow execution:                                    │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Step 1: Validate order (Cloud Function)                  │        │
│  │  Step 2: Check fraud score (external API)                │        │
│  │  Step 3: Switch on fraud score:                          │        │
│  │    ├── Low risk → auto-approve, skip to Step 5           │        │
│  │    └── High risk → Step 4 (human review)                │        │
│  │  Step 4: Create callback + send Slack notification       │        │
│  │    → Workflow PAUSES (up to 24 hours)                    │        │
│  │    → Manager clicks approve/reject in Slack              │        │
│  │    → Callback triggers workflow resume                   │        │
│  │  Step 5: Process payment (Stripe)                        │        │
│  │  Step 6: Parallel: email + inventory + analytics         │        │
│  │  Step 7: Return order confirmation                       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Durable: workflow state persisted across the 24-hour wait           │
│  No servers running during the wait                                   │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Infrastructure Provisioning

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: AUTOMATED ENVIRONMENT PROVISIONING                     │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Developer requests new environment → Workflow orchestrates:         │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Step 1: Create GCP project (Resource Manager connector) │        │
│  │  Step 2: Link billing account                            │        │
│  │  Step 3: Enable APIs (compute, run, sql, etc.)           │        │
│  │  Step 4: Parallel:                                        │        │
│  │    ├── Create VPC + subnets (Compute connector)          │        │
│  │    ├── Create Cloud SQL instance (SQL connector)         │        │
│  │    └── Create GCS buckets (Storage connector)            │        │
│  │  Step 5: Wait for long-running operations (LRO polling) │        │
│  │  Step 6: Deploy Cloud Run services                       │        │
│  │  Step 7: Configure DNS records                           │        │
│  │  Step 8: Send Slack notification with environment URLs   │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Benefits over Terraform for this use case:                          │
│  • Built-in waiting for long-running operations                     │
│  • Conditional logic (skip steps based on environment type)         │
│  • Notifications at each step                                        │
│  • No state file to manage                                           │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Deploy workflow | `gcloud workflows deploy NAME --source=FILE --location=L` |
| Execute workflow | `gcloud workflows run NAME --data='JSON' --location=L` |
| List executions | `gcloud workflows executions list --workflow=NAME` |
| Cancel execution | `gcloud workflows executions cancel EXEC_ID` |
| Step types | assign, call, switch, for, try/except, parallel, return, raise |
| HTTP call | `call: http.post` with `args: {url, body, auth}` |
| Connector call | `call: googleapis.SERVICE.VERSION.METHOD` |
| Parallel branches | `parallel: { branches: [...] }` |
| Parallel for | `parallel: { for: { value, in, steps } }` |
| Callbacks | `events.create_callback_endpoint` + `events.await_callback` |
| Max duration | 1 year |
| Free tier | 5,000 internal + 2,000 external steps/month |

---

## What is Orchestration? (Beginner Explanation)

### The Recipe Analogy

Think of **Workflows as a recipe** — it tells GCP what steps to do, in what order, with conditions for what to do if something goes wrong.

```
┌──────────────────────────────────────────────────────────────┐
│         WORKFLOWS = A RECIPE                                  │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Cooking Recipe                Cloud Workflows                │
│  ──────────────                ────────────────                │
│                                                                │
│  Recipe title          ═══►   Workflow name                   │
│  ("Chocolate Cake")            ("order-processing")            │
│                                                                │
│  Ingredients list      ═══►   Input parameters                │
│  (flour, eggs, sugar)          ({orderId, customer, amount})   │
│                                                                │
│  Step 1: Mix flour     ═══►   Step 1: Validate order          │
│  Step 2: Add eggs      ═══►   Step 2: Check inventory         │
│  Step 3: Bake 30 min   ═══►   Step 3: Process payment         │
│                                                                │
│  "If too dry, add      ═══►   switch: if error → retry /      │
│   more milk"                   notify / take alternate path    │
│                                                                │
│  "While not golden,    ═══►   for: loop through items         │
│   keep baking"                 until all processed             │
│                                                                │
│  "Oops, burned? Start  ═══►   try/except: catch errors,       │
│   over with lower temp"        retry with backoff              │
│                                                                │
│  Final result: Cake!   ═══►   return: {status: "completed"}   │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Breaking It Down

| Concept | Recipe Analogy | What It Really Means |
|---------|---------------|---------------------|
| **Workflow** | The complete recipe from start to finish | A YAML definition of steps that call services in order |
| **Step** | One instruction ("mix flour and sugar") | One action — an API call, an assignment, a condition |
| **Input params** | Ingredients you need before starting | JSON data passed when you execute the workflow |
| **switch** | "If mixture is lumpy, blend more" | Conditional logic — do different things based on data |
| **for loop** | "Repeat for each layer of the cake" | Process each item in a list, one by one |
| **try/except** | "If the oven breaks, use the neighbor's" | Catch errors and handle them (retry, skip, alert) |
| **retry** | "If cake doesn't rise, try again with more yeast" | Automatically retry a failed step with backoff |
| **parallel** | "While cake bakes, make the frosting" | Run multiple steps at the same time |
| **return** | The finished cake! | The final output of the workflow |

### Why Does Orchestration Matter?

1. **Coordinates multiple services** — Instead of one service calling another calling another, Workflows manages the whole chain
2. **Built-in error handling** — If step 3 fails, automatically retry or take an alternate path
3. **Durable state** — If GCP has an outage mid-workflow, it resumes where it left off
4. **No servers** — Completely serverless. You don't manage infrastructure for the orchestrator itself
5. **Readable** — YAML workflows are easier to understand than spaghetti code calling 10 APIs

> **One-liner**: If your microservices are individual chefs, Workflows is the head chef reading the recipe and telling each one what to do, in what order, and what to do if something goes wrong.

---

## Console Walkthrough: Managing & Deleting

### Deleting a Workflow

```
┌──────────────────────────────────────────────────────────────┐
│         DELETE A WORKFLOW FROM CONSOLE                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Workflows                                          │
│                                                                │
│  Step 2: Find the workflow                                     │
│  • You'll see a list of all workflows in the project          │
│  • Use the filter/search bar to find a specific one           │
│                                                                │
│  Step 3: Delete                                                │
│  • Check the checkbox next to the workflow name               │
│  • Click DELETE in the top action bar                         │
│  • Confirm the deletion in the dialog                         │
│                                                                │
│  Alternative:                                                  │
│  • Click the workflow name to open its detail page            │
│  • Click the ⋮ (three-dot menu) or DELETE button             │
│  • Confirm the deletion                                       │
│                                                                │
│  ⚠ Deleting a workflow also deletes all its execution         │
│    history. Running executions will be cancelled.              │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# CLI equivalent
gcloud workflows delete WORKFLOW_NAME --location=REGION

# Example
gcloud workflows delete order-processing --location=us-central1
```

### Cancelling an Execution

```
┌──────────────────────────────────────────────────────────────┐
│         CANCEL A WORKFLOW EXECUTION                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Navigate                                              │
│  Console → Workflows → Click workflow name                    │
│                                                                │
│  Step 2: Find the execution                                    │
│  • The "Executions" tab shows all runs                        │
│  • Look for executions with status: ACTIVE                    │
│  • Click the execution ID to open its detail                  │
│                                                                │
│  Step 3: Cancel                                                │
│  • Click the CANCEL button at the top of the execution detail │
│  • Status changes to: CANCELLED                               │
│  • The workflow stops at its current step                     │
│                                                                │
│  ⚡ Note: Cancellation is not instant. If a step is mid-      │
│    execution (e.g., waiting for an HTTP response), it will    │
│    finish that step before the workflow stops.                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

```bash
# CLI equivalent
gcloud workflows executions cancel EXECUTION_ID \
    --workflow=WORKFLOW_NAME \
    --location=REGION

# Example: List executions then cancel one
gcloud workflows executions list \
    --workflow=order-processing \
    --location=us-central1

gcloud workflows executions cancel abc-123-def-456 \
    --workflow=order-processing \
    --location=us-central1
```

---

## What's Next?

Continue to **Chapter 50: API Gateway & Apigee** → `50-api-gateway-apigee.md`
