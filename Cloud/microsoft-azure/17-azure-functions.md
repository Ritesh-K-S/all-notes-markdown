# Chapter 17: Azure Functions

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Azure Functions Fundamentals](#part-1-azure-functions-fundamentals)
- [Part 2: Creating a Function App (Full Portal Walkthrough)](#part-2-creating-a-function-app-full-portal-walkthrough)
- [Part 3: Triggers (Deep Dive)](#part-3-triggers-deep-dive)
- [Part 4: Bindings (Input & Output)](#part-4-bindings-input--output)
- [Part 5: Durable Functions](#part-5-durable-functions)
- [Part 6: Deployment Slots](#part-6-deployment-slots)
- [Part 7: Terraform & Bicep](#part-7-terraform--bicep)
- [Part 8: az CLI Reference](#part-8-az-cli-reference)
- [Part 9: Real-World Patterns](#part-9-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Functions is Microsoft's serverless compute service. You write functions (small pieces of code) that run in response to events — an HTTP request, a message in a queue, a file upload, a timer, a database change. You pay only when your code executes.

```
What you'll learn:
├── Azure Functions Fundamentals
│   ├── What & why (event-driven, serverless)
│   ├── Function App vs Function
│   ├── Triggers & Bindings concept
│   └── Functions vs Logic Apps vs Container Apps
├── Creating a Function App (Full Portal Walkthrough)
│   ├── Basics (name, runtime, region, OS)
│   ├── Hosting plans (Consumption, Premium, Dedicated)
│   ├── Storage account
│   ├── Networking (VNet integration, private endpoints)
│   └── Monitoring (App Insights)
├── Triggers (Deep Dive)
│   ├── HTTP trigger
│   ├── Timer trigger (CRON)
│   ├── Queue Storage trigger
│   ├── Blob Storage trigger
│   ├── Service Bus trigger
│   ├── Cosmos DB trigger (change feed)
│   ├── Event Grid trigger
│   └── Event Hub trigger
├── Bindings (Input & Output)
├── Durable Functions (Orchestrations)
├── Deployment Slots
├── Terraform, Bicep, az CLI
└── Real-world patterns
```

---

## Part 1: Azure Functions Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           AZURE FUNCTIONS CONCEPT                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What is Azure Functions?                                            │
│ ├── Write a function (code) → Deploy → It runs on events        │
│ ├── No servers to manage, no infra to provision                 │
│ ├── Auto-scales (0 to thousands of instances)                   │
│ ├── Pay-per-execution (Consumption plan)                        │
│ └── Supports: C#, JavaScript, Python, Java, PowerShell, Go    │
│                                                                       │
│ Hierarchy:                                                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Function App (container for functions)                       │  │
│ │ ├── Shared settings: runtime, plan, region, storage        │  │
│ │ ├── Shared configuration: App settings, connection strings│  │
│ │ │                                                           │  │
│ │ ├── Function 1: HttpTrigger-GetUsers                       │  │
│ │ │   Trigger: HTTP GET /api/users                          │  │
│ │ │   Output binding: → (none, returns JSON response)       │  │
│ │ │                                                           │  │
│ │ ├── Function 2: QueueTrigger-ProcessOrder                  │  │
│ │ │   Trigger: Queue message arrives in "orders" queue      │  │
│ │ │   Output binding: → Cosmos DB (save processed order)    │  │
│ │ │                                                           │  │
│ │ ├── Function 3: TimerTrigger-DailyReport                  │  │
│ │ │   Trigger: CRON "0 0 2 * * *" (2 AM daily)            │  │
│ │ │   Output binding: → Blob Storage (save report PDF)      │  │
│ │ │                                                           │  │
│ │ └── Function 4: BlobTrigger-ImageResize                   │  │
│ │     Trigger: New blob in "uploads" container              │  │
│ │     Output binding: → Blob Storage ("thumbnails" container)│  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Triggers & Bindings:                                                 │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │                                                              │  │
│ │ [Input Binding]                                              │  │
│ │       ↓                                                      │  │
│ │ [TRIGGER] → [YOUR FUNCTION CODE] → [Output Binding]        │  │
│ │                                                              │  │
│ │ Trigger: WHAT starts the function (1 per function)          │  │
│ │ Input Binding: Data to READ (0 or more)                    │  │
│ │ Output Binding: Where to WRITE results (0 or more)         │  │
│ │                                                              │  │
│ │ Example:                                                     │  │
│ │ Trigger: Queue message (order ID)                          │  │
│ │ Input: Cosmos DB document (order details, by order ID)     │  │
│ │ Output: Blob Storage (save receipt PDF)                    │  │
│ │         + Queue (send notification message)                │  │
│ │                                                              │  │
│ │ ⚡ Bindings are DECLARATIVE — no SDK code needed!           │  │
│ │   You declare what to connect to, Functions handles I/O.  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Comparison:                                                          │
│ ┌────────────────┬───────────────┬───────────────┬──────────────┐ │
│ │ Feature         │ Azure Funcs    │ Logic Apps     │ Container Apps│ │
│ ├────────────────┼───────────────┼───────────────┼──────────────┤ │
│ │ Code vs Config  │ Code-first    │ Visual designer│ Container    │ │
│ │ Trigger types   │ 15+ triggers  │ 400+ connectors│ HTTP/Events │ │
│ │ Scale to zero   │ Yes (Consumption)│ Yes         │ Yes           │ │
│ │ Stateful        │ Durable Funcs │ Built-in       │ Dapr sidecars│ │
│ │ Languages       │ C#/JS/Py/Java │ No code needed│ Any (Docker) │ │
│ │ Best for        │ Event-driven  │ Workflow/integ │ Microservices│ │
│ └────────────────┴───────────────┴───────────────┴──────────────┘ │
│                                                                       │
│ ⚡ AWS equivalent: Lambda                                           │
│ ⚡ GCP equivalent: Cloud Functions                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating a Function App (Full Portal Walkthrough)

```
Console → Function App → Create

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 1: BASICS                                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-functions-prod ▼]                              │
│                                                                       │
│ Function App name: [func-orders-prod]                              │
│ ⚡ Naming: func-{purpose}-{env}                                     │
│   Globally unique! Creates: func-orders-prod.azurewebsites.net  │
│                                                                       │
│ Deploy code or container image:                                     │
│ ● Code                                                              │
│ ○ Container image (Docker — for custom runtimes)                 │
│                                                                       │
│ Runtime stack: [Node.js ▼]                                         │
│   ○ .NET (C# — most native, best Azure integration)             │
│   ● Node.js (JavaScript/TypeScript)                               │
│   ○ Python                                                        │
│   ○ Java                                                          │
│   ○ PowerShell Core                                               │
│   ○ Custom Handler (Go, Rust, etc.)                              │
│                                                                       │
│ Version: [20 LTS ▼]                                               │
│                                                                       │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ Operating system:                                                    │
│ ● Linux (recommended for most — cheaper, faster cold start)     │
│ ○ Windows (needed for .NET Framework, PowerShell)                │
│                                                                       │
│ ── Hosting options ──                                               │
│                                                                       │
│ Hosting plan:                                                        │
│ ┌────────────────────────────────────────────────────────────┐    │
│ │                                                            │    │
│ │ ○ Consumption (Serverless)  ← Default, cheapest          │    │
│ │   ├── Scale: 0 to 200 instances automatically            │    │
│ │   ├── Pay: Per execution + execution time                │    │
│ │   ├── Free: 1M executions + 400K GB-s / month           │    │
│ │   ├── Timeout: Max 10 min (default 5 min)                │    │
│ │   ├── Cold start: Yes (can be 1-3 seconds)               │    │
│ │   ├── Memory: Up to 1.5 GB per instance                  │    │
│ │   └── Best for: Sporadic/event-driven workloads         │    │
│ │                                                            │    │
│ │ ● Flex Consumption (preview) ← Best of both worlds      │    │
│ │   ├── Scale: 0 to 1000 instances                         │    │
│ │   ├── Pay: Per execution (with always-ready instances)  │    │
│ │   ├── Always-ready: Keep N instances warm (no cold start)│    │
│ │   ├── VNet integration included                          │    │
│ │   ├── Higher memory/CPU options                          │    │
│ │   └── Best for: Production APIs with scale-to-zero     │    │
│ │                                                            │    │
│ │ ○ Functions Premium (EP1/EP2/EP3)                        │    │
│ │   ├── Scale: Pre-warmed instances (no cold start!) ✅    │    │
│ │   ├── Min 1 instance always running                      │    │
│ │   ├── VNet integration ✅                                 │    │
│ │   ├── Unlimited execution time                           │    │
│ │   ├── More CPU/memory (up to 14 GB)                     │    │
│ │   ├── Private endpoints ✅                                │    │
│ │   └── Best for: Production, VNet-connected workloads   │    │
│ │                                                            │    │
│ │ ○ App Service plan (Dedicated)                           │    │
│ │   ├── Runs on existing App Service plan VMs             │    │
│ │   ├── Predictable pricing (same as App Service)         │    │
│ │   ├── No auto-scale to zero                              │    │
│ │   ├── Useful when you already have an App Service plan  │    │
│ │   └── Best for: Already have plan, need consistent perf │    │
│ │                                                            │    │
│ └────────────────────────────────────────────────────────────┘    │
│                                                                       │
│ ⚡ Plan comparison:                                                   │
│ ┌────────────────┬──────────────┬──────────────┬───────────────┐  │
│ │ Feature         │ Consumption   │ Premium       │ Dedicated     │  │
│ ├────────────────┼──────────────┼──────────────┼───────────────┤  │
│ │ Scale to zero   │ Yes ✅       │ No (min 1)   │ No            │  │
│ │ Cold start      │ Yes ❌       │ No ✅ (warm) │ No ✅         │  │
│ │ Max timeout     │ 10 min       │ Unlimited    │ Unlimited     │  │
│ │ VNet            │ No ❌        │ Yes ✅       │ Yes ✅        │  │
│ │ Private endpoint│ No           │ Yes ✅       │ Yes ✅        │  │
│ │ Max memory      │ 1.5 GB       │ 14 GB        │ Plan-based    │  │
│ │ Max instances   │ 200          │ 100          │ 10-30         │  │
│ │ Min cost/month  │ $0 (free tier)│ ~$170       │ Plan cost     │  │
│ └────────────────┴──────────────┴──────────────┴───────────────┘  │
│                                                                       │
│ [Next: Storage >]                                                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 2: STORAGE                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Storage account: [stfuncordersprod ▼]  [Create new]               │
│ ⚡ Functions REQUIRES a storage account for:                        │
│   ├── Function code/binaries (stored in blob container)          │
│   ├── Trigger state (leases, checkpoints for queue/blob triggers)│
│   ├── Keys (host keys, function keys)                            │
│   └── Timer trigger schedules                                    │
│                                                                       │
│ ⚡ Use dedicated storage account per Function App.                 │
│   Don't share with other apps (performance isolation).          │
│                                                                       │
│ [Next: Networking >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 3: NETWORKING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Enable public access: ● On  ○ Off                                 │
│ ⚡ Off = only accessible via private endpoints or VNet.            │
│                                                                       │
│ Enable network injection:                                           │
│ ☑ Enable (Premium and Dedicated plans only!)                     │
│                                                                       │
│ Virtual network: [vnet-prod ▼]                                    │
│ Subnet: [subnet-functions (10.0.5.0/24) ▼]                      │
│ ⚡ VNet integration: Functions can access private resources       │
│   (SQL, Redis, Storage via private endpoint, etc.)              │
│   Requires Premium or Dedicated plan!                           │
│   Consumption plan does NOT support VNet.                       │
│                                                                       │
│ Private endpoint:                                                    │
│ [+ Add private endpoint]                                           │
│ Name: [pe-func-orders]                                             │
│ VNet: [vnet-prod]  Subnet: [subnet-endpoints]                   │
│ ⚡ Private endpoint = Function App accessible only from VNet.     │
│   No public URL. Internal apps call via private DNS.            │
│                                                                       │
│ [Next: Monitoring >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 4: MONITORING                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Enable Application Insights: ☑ Yes ← ALWAYS enable!             │
│ Application Insights: [appi-func-orders-prod ▼]  [Create new]   │
│                                                                       │
│ ⚡ Application Insights provides:                                   │
│   ├── Execution logs (each function invocation)                 │
│   ├── Performance metrics (duration, failures, throughput)      │
│   ├── Live metrics stream (real-time monitoring)                │
│   ├── Dependency tracking (SQL, HTTP calls, queue operations)  │
│   ├── Exception tracking with stack traces                     │
│   └── Custom metrics and events                                │
│                                                                       │
│ [Next: Deployment >]                                                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TAB 5: DEPLOYMENT                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ☐ Enable continuous deployment                                     │
│                                                                       │
│ (If checked:)                                                       │
│ Source: [GitHub ▼]                                                  │
│ Organization: [my-org]                                              │
│ Repository: [functions-orders]                                     │
│ Branch: [main]                                                      │
│ ⚡ Sets up GitHub Actions workflow automatically!                   │
│   Push to main → build → deploy to Function App.               │
│                                                                       │
│ [Review + create] → [Create]                                       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Triggers (Deep Dive)

```
┌─────────────────────────────────────────────────────────────────────┐
│           HTTP TRIGGER                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Most common trigger. Function runs when HTTP request arrives.     │
│                                                                       │
│ // JavaScript (Node.js v4 programming model)                       │
│ const { app } = require('@azure/functions');                       │
│                                                                       │
│ app.http('getUsers', {                                              │
│     methods: ['GET'],                                               │
│     authLevel: 'anonymous',  // or 'function' or 'admin'         │
│     route: 'users/{id?}',   // optional route parameter          │
│     handler: async (request, context) => {                         │
│         const id = request.params.id;                              │
│         context.log(`Getting user: ${id}`);                       │
│                                                                       │
│         if (id) {                                                   │
│             // Get specific user                                   │
│             return { jsonBody: { id, name: 'John' } };            │
│         }                                                           │
│         return { jsonBody: [{ id: 1, name: 'John' }] };          │
│     }                                                               │
│ });                                                                  │
│                                                                       │
│ URL: https://func-orders-prod.azurewebsites.net/api/users        │
│ URL: https://func-orders-prod.azurewebsites.net/api/users/123    │
│                                                                       │
│ Auth levels:                                                        │
│ ├── anonymous: No key needed (public)                            │
│ ├── function: Function-specific key in query/header             │
│ │   ?code=abc123... or x-functions-key: abc123...               │
│ ├── admin: Master key only (admin operations)                   │
│ └── ⚡ For production APIs: Use anonymous + API Management      │
│     or anonymous + Easy Auth (Azure AD / Entra ID)             │
│                                                                       │
│ # Python                                                            │
│ import azure.functions as func                                     │
│ import json                                                         │
│                                                                       │
│ app = func.FunctionApp()                                           │
│                                                                       │
│ @app.route(route="users/{id?}", methods=["GET"],                  │
│            auth_level=func.AuthLevel.ANONYMOUS)                    │
│ def get_users(req: func.HttpRequest) -> func.HttpResponse:       │
│     user_id = req.route_params.get('id')                          │
│     if user_id:                                                     │
│         return func.HttpResponse(                                  │
│             json.dumps({"id": user_id, "name": "John"}),          │
│             mimetype="application/json")                           │
│     return func.HttpResponse(                                      │
│         json.dumps([{"id": 1, "name": "John"}]),                  │
│         mimetype="application/json")                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TIMER TRIGGER (CRON)                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Runs on a schedule (like cron).                                    │
│                                                                       │
│ // JavaScript                                                       │
│ app.timer('dailyReport', {                                         │
│     schedule: '0 0 2 * * *',  // 2:00 AM UTC daily               │
│     handler: async (myTimer, context) => {                         │
│         context.log('Running daily report...');                   │
│         // Generate report, send email, etc.                      │
│     }                                                               │
│ });                                                                  │
│                                                                       │
│ CRON format: {second} {minute} {hour} {day} {month} {day-of-week}│
│ ⚡ Azure Functions uses 6-field CRON (includes seconds!)           │
│                                                                       │
│ Examples:                                                            │
│ ├── 0 */5 * * * *    → Every 5 minutes                          │
│ ├── 0 0 * * * *      → Every hour                                │
│ ├── 0 0 2 * * *      → Daily at 2 AM                             │
│ ├── 0 0 9 * * 1-5    → Weekdays at 9 AM                         │
│ ├── 0 0 0 1 * *      → First of every month at midnight        │
│ └── 0 30 9 * * 1     → Every Monday at 9:30 AM                 │
│                                                                       │
│ ⚡ Timezone: Set WEBSITE_TIME_ZONE app setting to your timezone.  │
│   Example: "India Standard Time" or "Asia/Kolkata"              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           QUEUE STORAGE TRIGGER                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Runs when a message arrives in Azure Queue Storage.                │
│                                                                       │
│ // JavaScript                                                       │
│ app.storageQueue('processOrder', {                                 │
│     queueName: 'orders',                                           │
│     connection: 'AzureWebJobsStorage',                            │
│     handler: async (queueItem, context) => {                      │
│         context.log('Processing order:', queueItem);              │
│         const order = queueItem;  // Auto-deserialized JSON      │
│         // Process the order...                                    │
│     }                                                               │
│ });                                                                  │
│                                                                       │
│ ⚡ Queue trigger behavior:                                           │
│   ├── Batch size: Processes up to 16 messages at once (default)│
│   ├── Visibility timeout: 5 min (message hidden while processing)│
│   ├── Poison queue: After 5 failures → moves to "orders-poison"│
│   ├── Polling: Exponential backoff (efficient when queue empty)│
│   └── Scaling: Functions auto-scales based on queue length     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           BLOB STORAGE TRIGGER                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Runs when a blob is created or updated in a container.             │
│                                                                       │
│ // JavaScript                                                       │
│ app.storageBlob('imageResize', {                                  │
│     path: 'uploads/{name}',                                        │
│     connection: 'AzureWebJobsStorage',                            │
│     handler: async (blob, context) => {                            │
│         context.log(`Processing blob: ${context.triggerMetadata   │
│             .name}, Size: ${blob.length} bytes`);                 │
│         // Resize image, save to thumbnails container             │
│     }                                                               │
│ });                                                                  │
│                                                                       │
│ ⚡ Blob trigger notes:                                               │
│   ├── Uses polling (not instant — can be 1-10 min delay!)       │
│   ├── For instant: Use Event Grid trigger with blob events ✅    │
│   ├── {name} in path = binding expression (filename)            │
│   └── Processes new AND updated blobs                            │
│                                                                       │
│ Better: Event Grid + Blob trigger (instant):                       │
│ app.generic('imageResizeEventGrid', {                              │
│     trigger: {                                                      │
│         type: 'eventGridTrigger'                                  │
│     },                                                              │
│     handler: async (event, context) => {                           │
│         const blobUrl = event.data.url;                            │
│         // Process immediately when blob created                  │
│     }                                                               │
│ });                                                                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           SERVICE BUS TRIGGER                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Runs when a message arrives in Service Bus queue or topic.        │
│                                                                       │
│ // JavaScript — Queue trigger                                      │
│ app.serviceBusQueue('processPayment', {                           │
│     queueName: 'payments',                                         │
│     connection: 'ServiceBusConnection',                            │
│     handler: async (message, context) => {                         │
│         context.log('Payment received:', message);                │
│         // Process payment...                                      │
│     }                                                               │
│ });                                                                  │
│                                                                       │
│ // JavaScript — Topic + subscription trigger                       │
│ app.serviceBusTopic('handleNotification', {                       │
│     topicName: 'notifications',                                    │
│     subscriptionName: 'email-handler',                            │
│     connection: 'ServiceBusConnection',                            │
│     handler: async (message, context) => {                         │
│         context.log('Notification:', message);                    │
│         // Send email...                                           │
│     }                                                               │
│ });                                                                  │
│                                                                       │
│ ⚡ Service Bus vs Queue Storage:                                     │
│   ├── Queue Storage: Simple, cheap, 64 KB messages, basic       │
│   ├── Service Bus: Enterprise, 256 KB-100 MB, topics/subs,     │
│   │   dead-letter, sessions, transactions, duplicate detection  │
│   └── Use Service Bus for production message processing ✅       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           COSMOS DB TRIGGER (Change Feed)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Runs when documents are created/updated in Cosmos DB.              │
│                                                                       │
│ // JavaScript                                                       │
│ app.cosmosDB('orderChangeHandler', {                               │
│     connectionStringSetting: 'CosmosDBConnection',                │
│     databaseName: 'ecommerce',                                     │
│     containerName: 'orders',                                       │
│     leaseContainerName: 'leases',                                 │
│     createLeaseContainerIfNotExists: true,                        │
│     handler: async (documents, context) => {                       │
│         documents.forEach(doc => {                                 │
│             context.log(`Order changed: ${doc.id}`);              │
│             // Sync to search index, send notification, etc.      │
│         });                                                        │
│     }                                                               │
│ });                                                                  │
│                                                                       │
│ ⚡ Change feed:                                                      │
│   ├── Captures inserts and updates (not deletes!)                │
│   ├── Lease container tracks processing position                 │
│   ├── Delivers in batches (not single documents)                 │
│   └── Use for: Event sourcing, data sync, notifications         │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

All trigger types summary:
┌──────────────────┬─────────────────────────────────────────────────┐
│ Trigger           │ Fires when...                                   │
├──────────────────┼─────────────────────────────────────────────────┤
│ HTTP              │ HTTP request received                           │
│ Timer             │ Scheduled time (CRON)                           │
│ Queue Storage     │ Message in Azure Queue                          │
│ Blob Storage      │ Blob created/updated (polling)                 │
│ Service Bus       │ Message in queue/topic subscription            │
│ Cosmos DB         │ Document created/updated (change feed)         │
│ Event Grid        │ Event Grid event (any Azure event)             │
│ Event Hub         │ Event received in Event Hub (streaming)        │
│ IoT Hub           │ Message from IoT device                        │
│ SignalR            │ SignalR negotiation/messages                   │
│ Kafka             │ Kafka message (preview)                        │
│ RabbitMQ          │ RabbitMQ message                                │
│ Dapr              │ Dapr binding/topic                              │
└──────────────────┴─────────────────────────────────────────────────┘
```

---

## Part 4: Bindings (Input & Output)

```
┌─────────────────────────────────────────────────────────────────────┐
│           BINDINGS                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Bindings connect your function to data WITHOUT writing SDK code.  │
│                                                                       │
│ Input bindings (read data):                                         │
│ ├── Cosmos DB: Read document by ID                               │
│ ├── Blob Storage: Read blob content                              │
│ ├── Table Storage: Read table entity                             │
│ └── SQL: Query database                                          │
│                                                                       │
│ Output bindings (write data):                                       │
│ ├── Cosmos DB: Write/update document                             │
│ ├── Blob Storage: Write blob                                     │
│ ├── Queue Storage: Send queue message                            │
│ ├── Service Bus: Send message to queue/topic                    │
│ ├── Table Storage: Write table entity                            │
│ ├── Event Hub: Send event                                        │
│ ├── Event Grid: Publish event                                    │
│ ├── SendGrid: Send email                                         │
│ ├── SignalR: Send real-time message                              │
│ └── SQL: Insert/update row                                       │
│                                                                       │
│ Example: HTTP trigger + Cosmos DB input + Queue output:           │
│                                                                       │
│ // Get order by ID from Cosmos DB, send to processing queue      │
│ app.http('getOrder', {                                              │
│     methods: ['GET'],                                               │
│     route: 'orders/{orderId}',                                     │
│     extraInputs: [                                                  │
│         input.cosmosDB({                                           │
│             connectionStringSetting: 'CosmosDBConnection',        │
│             databaseName: 'ecommerce',                             │
│             containerName: 'orders',                               │
│             id: '{orderId}',      // From route parameter        │
│             partitionKey: '{orderId}',                             │
│         })                                                          │
│     ],                                                               │
│     extraOutputs: [                                                  │
│         output.storageQueue({                                      │
│             queueName: 'order-processing',                        │
│             connection: 'AzureWebJobsStorage',                    │
│         })                                                          │
│     ],                                                               │
│     handler: async (request, context) => {                         │
│         const order = context.extraInputs.get(cosmosInput);      │
│         // Send to processing queue                                │
│         context.extraOutputs.set(queueOutput, {                   │
│             orderId: order.id,                                     │
│             action: 'process'                                     │
│         });                                                         │
│         return { jsonBody: order };                                │
│     }                                                               │
│ });                                                                  │
│                                                                       │
│ ⚡ No Cosmos DB SDK code! No Queue SDK code!                       │
│   Bindings handle all the connection, read, write operations.    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Durable Functions

```
┌─────────────────────────────────────────────────────────────────────┐
│           DURABLE FUNCTIONS                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Problem: Regular functions are stateless and short-lived.          │
│ What if you need:                                                   │
│ ├── A workflow with multiple steps?                               │
│ ├── Wait for human approval?                                     │
│ ├── Fan-out to 100 parallel tasks, then fan-in?                 │
│ ├── A long-running process (hours/days)?                        │
│ └── State between function calls?                                │
│                                                                       │
│ → Durable Functions: Stateful workflows in serverless!            │
│                                                                       │
│ Patterns:                                                            │
│                                                                       │
│ 1. Function chaining (sequential steps):                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Orchestrator:                                                │  │
│ │ Step 1: Validate order → Step 2: Charge payment →          │  │
│ │ Step 3: Send confirmation → Step 4: Update inventory       │  │
│ │                                                              │  │
│ │ If Step 2 fails → automatic retry or compensation          │  │
│ │ State persisted between steps (survives crashes!)           │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 2. Fan-out/fan-in (parallel processing):                           │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Orchestrator:                                                │  │
│ │ Fan-out: Start 100 image processing tasks in parallel      │  │
│ │ Fan-in: Wait for ALL 100 to complete → aggregate results  │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 3. Human interaction (approval workflow):                          │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Orchestrator:                                                │  │
│ │ Submit request → Send approval email → WAIT (hours/days)  │  │
│ │ → Human clicks approve → Continue processing              │  │
│ │ → Or timeout after 72 hours → escalate                    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ 4. Monitor (recurring polling):                                    │
│ ┌──────────────────────────────────────────────────────────────┐  │
│ │ Orchestrator:                                                │  │
│ │ Check status → Not ready → Wait 30 min → Check again     │  │
│ │ → Ready! → Trigger action                                  │  │
│ │ Repeats until condition met or timeout.                    │  │
│ └──────────────────────────────────────────────────────────────┘  │
│                                                                       │
│ Code example (function chaining):                                  │
│                                                                       │
│ // Orchestrator function                                           │
│ const df = require('durable-functions');                            │
│                                                                       │
│ df.app.orchestration('orderWorkflow', function* (context) {       │
│     const order = context.df.getInput();                          │
│                                                                       │
│     // Step 1: Validate                                            │
│     const validated = yield context.df                             │
│         .callActivity('validateOrder', order);                    │
│                                                                       │
│     // Step 2: Charge payment                                      │
│     const payment = yield context.df                               │
│         .callActivity('chargePayment', validated);                │
│                                                                       │
│     // Step 3: Send confirmation                                   │
│     yield context.df                                                │
│         .callActivity('sendConfirmation', payment);               │
│                                                                       │
│     // Step 4: Update inventory                                    │
│     yield context.df                                                │
│         .callActivity('updateInventory', validated.items);        │
│                                                                       │
│     return { status: 'completed', orderId: order.id };            │
│ });                                                                  │
│                                                                       │
│ // Activity functions (each step)                                  │
│ df.app.activity('validateOrder', {                                 │
│     handler: async (order) => {                                    │
│         // Validate order, check stock, etc.                      │
│         return { ...order, validated: true };                     │
│     }                                                               │
│ });                                                                  │
│                                                                       │
│ df.app.activity('chargePayment', { handler: async (order) => {   │
│     // Call payment API, charge card                              │
│     return { ...order, paymentId: 'PAY-123' };                   │
│ }});                                                                │
│                                                                       │
│ // HTTP starter (kicks off the orchestration)                      │
│ app.http('startOrder', {                                           │
│     route: 'orders',                                                │
│     methods: ['POST'],                                              │
│     extraInputs: [df.input.durableClient()],                      │
│     handler: async (request, context) => {                         │
│         const client = df.getClient(context);                     │
│         const order = await request.json();                       │
│         const instanceId = await client                           │
│             .startNew('orderWorkflow', { input: order });        │
│         return client.createCheckStatusResponse(                  │
│             request, instanceId);                                  │
│     }                                                               │
│ });                                                                  │
│                                                                       │
│ ⚡ What happens:                                                     │
│ POST /api/orders → Returns status URL                             │
│ GET /api/instances/{id} → Check progress                         │
│ Status: Running → Completed (or Failed)                          │
│ State persisted in Storage (survives restarts!)                   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Deployment Slots

```
┌─────────────────────────────────────────────────────────────────────┐
│           DEPLOYMENT SLOTS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Function App → Deployment slots (Premium/Dedicated plans only)   │
│                                                                       │
│ [+ Add slot]                                                        │
│ Name: [staging]                                                     │
│ Clone settings from: [Production ▼]                                │
│                                                                       │
│ Slots:                                                               │
│ ├── Production: func-orders-prod.azurewebsites.net              │
│ └── Staging: func-orders-prod-staging.azurewebsites.net         │
│                                                                       │
│ Workflow:                                                            │
│ 1. Deploy new code to staging slot                                │
│ 2. Test staging URL (warm up, verify)                             │
│ 3. Swap: staging ↔ production (instant, zero-downtime)          │
│ 4. Problem? Swap back (instant rollback!)                        │
│                                                                       │
│ [Swap] button: Staging → Production                               │
│                                                                       │
│ ⚡ Slot settings:                                                    │
│ ├── Slot-specific settings (sticky — don't swap):               │
│ │   DB_CONNECTION_STRING, FEATURE_FLAG, etc.                    │
│ │   Mark as "Deployment slot setting" ☑                         │
│ ├── Non-sticky settings (swap with the code):                   │
│ │   APP_VERSION, etc.                                            │
│ └── ⚡ Connection strings to staging DB = slot-specific!         │
│     Production slot → production DB                              │
│     Staging slot → staging DB                                    │
│                                                                       │
│ ⚡ Not available on Consumption plan!                               │
│   Use Premium or Dedicated for deployment slots.                 │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: Terraform & Bicep

### Terraform

```hcl
# Function App (Linux, Node.js, Consumption plan)
resource "azurerm_service_plan" "functions" {
  name                = "asp-functions-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  os_type             = "Linux"
  sku_name            = "Y1"  # Consumption plan
  # Premium: "EP1", "EP2", "EP3"
  # Dedicated: "B1", "S1", "P1v3", etc.
}

resource "azurerm_linux_function_app" "orders" {
  name                = "func-orders-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.functions.id

  storage_account_name       = azurerm_storage_account.functions.name
  storage_account_access_key = azurerm_storage_account.functions.primary_access_key

  site_config {
    application_stack {
      node_version = "20"
    }

    cors {
      allowed_origins = ["https://app.company.com"]
    }

    application_insights_connection_string = azurerm_application_insights.main.connection_string
  }

  app_settings = {
    "WEBSITE_RUN_FROM_PACKAGE"    = "1"
    "CosmosDBConnection"          = azurerm_cosmosdb_account.main.primary_sql_connection_string
    "ServiceBusConnection"        = azurerm_servicebus_namespace.main.default_primary_connection_string
    "WEBSITE_TIME_ZONE"           = "India Standard Time"
  }

  identity {
    type = "SystemAssigned"
  }

  tags = {
    environment = "prod"
    team        = "backend"
  }
}

# Storage account for Functions
resource "azurerm_storage_account" "functions" {
  name                     = "stfuncordersprod"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

# Application Insights
resource "azurerm_application_insights" "main" {
  name                = "appi-func-orders-prod"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  application_type    = "web"
}

# Premium plan with VNet integration
resource "azurerm_service_plan" "premium" {
  name                = "asp-functions-premium"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  os_type             = "Linux"
  sku_name            = "EP1"
}

resource "azurerm_linux_function_app" "premium" {
  name                = "func-api-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.premium.id

  storage_account_name       = azurerm_storage_account.functions.name
  storage_account_access_key = azurerm_storage_account.functions.primary_access_key

  virtual_network_subnet_id = azurerm_subnet.functions.id

  site_config {
    application_stack {
      node_version = "20"
    }
    vnet_route_all_enabled = true
    elastic_instance_minimum = 1  # Always warm
  }

  app_settings = {
    "WEBSITE_RUN_FROM_PACKAGE" = "1"
  }
}

# Deployment slot (Premium/Dedicated only)
resource "azurerm_linux_function_app_slot" "staging" {
  name                 = "staging"
  function_app_id      = azurerm_linux_function_app.premium.id
  storage_account_name = azurerm_storage_account.functions.name

  site_config {
    application_stack {
      node_version = "20"
    }
  }

  app_settings = {
    "WEBSITE_RUN_FROM_PACKAGE" = "1"
    "ENVIRONMENT"               = "staging"  # Slot-specific
  }
}
```

### Bicep

```bicep
resource functionApp 'Microsoft.Web/sites@2023-12-01' = {
  name: 'func-orders-prod'
  location: resourceGroup().location
  kind: 'functionapp,linux'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: hostingPlan.id
    siteConfig: {
      linuxFxVersion: 'NODE|20'
      appSettings: [
        { name: 'AzureWebJobsStorage', value: storageConnectionString }
        { name: 'FUNCTIONS_EXTENSION_VERSION', value: '~4' }
        { name: 'FUNCTIONS_WORKER_RUNTIME', value: 'node' }
        { name: 'WEBSITE_RUN_FROM_PACKAGE', value: '1' }
        { name: 'APPLICATIONINSIGHTS_CONNECTION_STRING', value: appInsights.properties.ConnectionString }
      ]
    }
  }
}

resource hostingPlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-functions-prod'
  location: resourceGroup().location
  sku: {
    name: 'Y1'   // Consumption
    tier: 'Dynamic'
  }
  properties: {
    reserved: true  // Linux
  }
}
```

---

## Part 8: az CLI Reference

```bash
# Create Function App (Consumption, Node.js, Linux)
az functionapp create \
  --resource-group rg-functions-prod \
  --name func-orders-prod \
  --consumption-plan-location centralindia \
  --runtime node \
  --runtime-version 20 \
  --os-type Linux \
  --storage-account stfuncordersprod \
  --functions-version 4 \
  --app-insights appi-func-orders-prod

# Create Function App (Premium plan)
az functionapp plan create \
  --resource-group rg-functions-prod \
  --name asp-functions-premium \
  --location centralindia \
  --sku EP1 \
  --is-linux

az functionapp create \
  --resource-group rg-functions-prod \
  --name func-api-prod \
  --plan asp-functions-premium \
  --runtime node \
  --runtime-version 20 \
  --os-type Linux \
  --storage-account stfuncordersprod \
  --functions-version 4

# Deploy function code (zip deploy)
func azure functionapp publish func-orders-prod

# Or zip deploy via CLI
az functionapp deployment source config-zip \
  --resource-group rg-functions-prod \
  --name func-orders-prod \
  --src ./deploy.zip

# App settings
az functionapp config appsettings set \
  --resource-group rg-functions-prod \
  --name func-orders-prod \
  --settings "CosmosDBConnection=AccountEndpoint=..." \
             "WEBSITE_TIME_ZONE=India Standard Time"

# List functions
az functionapp function list \
  --resource-group rg-functions-prod \
  --name func-orders-prod \
  --output table

# Get function URL
az functionapp function show \
  --resource-group rg-functions-prod \
  --name func-orders-prod \
  --function-name getUsers \
  --query invokeUrlTemplate

# Scale (Premium plan — set minimum instances)
az functionapp scale config set \
  --resource-group rg-functions-prod \
  --name func-api-prod \
  --minimum-elastic-instance-count 2 \
  --maximum-elastic-worker-count 20

# VNet integration (Premium plan)
az functionapp vnet-integration add \
  --resource-group rg-functions-prod \
  --name func-api-prod \
  --vnet vnet-prod \
  --subnet subnet-functions

# Deployment slots
az functionapp deployment slot create \
  --resource-group rg-functions-prod \
  --name func-api-prod \
  --slot staging

# Deploy to slot
func azure functionapp publish func-api-prod --slot staging

# Swap slots
az functionapp deployment slot swap \
  --resource-group rg-functions-prod \
  --name func-api-prod \
  --slot staging \
  --target-slot production

# Logs (streaming)
az functionapp log tail \
  --resource-group rg-functions-prod \
  --name func-orders-prod

# Enable managed identity
az functionapp identity assign \
  --resource-group rg-functions-prod \
  --name func-orders-prod

# Delete
az functionapp delete \
  --resource-group rg-functions-prod \
  --name func-orders-prod

# ═══ Azure Functions Core Tools (local development) ═══

# Create new project
func init MyFunctionProject --javascript

# Create a function
func new --name getUsers --template "HTTP trigger" --authlevel anonymous

# Run locally
func start

# Test locally
curl http://localhost:7071/api/getUsers

# Deploy
func azure functionapp publish func-orders-prod
```

---

## Part 9: Real-World Patterns

### Startup

```
Setup: Event-driven backend with Functions

Function App: func-backend-prod (Consumption plan)
├── Runtime: Node.js 20, Linux
├── Plan: Consumption (scale to zero, free tier!)
├── Region: Central India
└── App Insights: Enabled

Functions:
├── HTTP triggers (REST API):
│   ├── getUsers (GET /api/users)
│   ├── createUser (POST /api/users)
│   ├── getOrders (GET /api/orders)
│   └── createOrder (POST /api/orders)
│
├── Queue triggers (async processing):
│   ├── processOrder (queue: orders → validate, charge)
│   └── sendEmail (queue: emails → send via SendGrid)
│
├── Timer triggers (scheduled):
│   └── dailyReport (0 0 2 * * * → generate stats email)
│
└── Blob trigger:
    └── imageResize (uploads/ → create thumbnail → thumbnails/)

Stack:
├── Functions → Cosmos DB (data)
├── Functions → Queue Storage (async messaging)
├── Functions → Blob Storage (file uploads)
└── Custom domain via API Management (free tier)

Cost: $0-20/month (free tier covers most startup traffic)
```

### Mid-Size

```
Architecture: Microservices with Functions

Function Apps:
├── func-api-prod (Premium EP1 — always warm)
│   ├── HTTP triggers: REST API gateway
│   ├── VNet integrated → Cosmos DB, SQL, Redis
│   ├── Private endpoint (no public access)
│   ├── Deployment slots: staging + production
│   ├── Min instances: 2 (no cold start)
│   └── Custom domain: api.company.com (via API Management)
│
├── func-orders-prod (Consumption — event-driven)
│   ├── Service Bus trigger: Process orders from queue
│   ├── Cosmos DB trigger: Sync order changes to search
│   ├── Timer: Hourly order aggregation
│   └── Durable Functions: Order workflow
│       Step 1: Validate → Step 2: Pay → Step 3: Fulfill
│
├── func-notifications-prod (Consumption)
│   ├── Queue trigger: Send emails (SendGrid)
│   ├── Queue trigger: Send SMS (Twilio)
│   ├── Queue trigger: Push notifications
│   └── Event Grid: React to Azure resource events
│
└── func-analytics-prod (Consumption)
    ├── Timer: Nightly data aggregation (Cosmos DB → SQL)
    ├── Blob trigger: Process uploaded CSV files
    └── HTTP: Analytics dashboard API

API Management:
├── api.company.com → routes to func-api-prod
├── Rate limiting, JWT validation, caching
├── Developer portal for API docs
└── Policies: Transform, throttle, mock

Monitoring:
├── Application Insights: All Function Apps
├── Live metrics stream
├── Alerts: Failed executions > 10/min → PagerDuty
├── Custom dashboards in Azure Portal
└── Log Analytics workspace for cross-app queries

CI/CD:
├── GitHub Actions → Build → Test → Deploy to staging slot
├── Integration tests against staging
├── Swap staging → production (zero downtime)
└── Rollback: Swap back if issues detected

Cost: ~$300-800/month (Premium API + Consumption workers)
```

### Enterprise

```
Multi-region serverless platform:

Region 1 (Central India):
├── func-api-prod (Premium EP2, VNet, private endpoint)
│   ├── API Management (Premium) in front
│   ├── Min 3 instances, max 30
│   ├── VNet → Cosmos DB, SQL, Redis, Storage (all private)
│   ├── Managed identity → Key Vault (secrets)
│   └── Deployment slots: staging, canary, production
│
├── func-orders-prod (Premium EP1)
│   ├── Service Bus trigger: Order processing pipeline
│   ├── Durable Functions: Multi-step order workflow
│   │   Validate → Reserve inventory → Charge payment →
│   │   Create shipment → Send confirmation → Update analytics
│   ├── Compensation: If payment fails → release inventory
│   └── Human approval: Orders > $10K → manager approval
│
├── func-etl-prod (Consumption)
│   ├── Timer: Nightly ETL (Cosmos DB → Synapse)
│   ├── Blob: Process uploaded files (CSV, Excel → SQL)
│   └── Event Grid: React to resource events (auditing)
│
└── func-notifications-prod (Consumption)
    └── Multi-channel notification engine

Region 2 (West Europe):
├── Same Function Apps (geo-replicated)
├── API Management (multi-region)
├── Traffic Manager: Route to nearest region
└── Cosmos DB: Multi-region writes

Security:
├── All Premium apps: Private endpoints (no public access)
├── VNet integration: All outbound via VNet
├── Managed identity: No connection strings in app settings
├── Key Vault references: @Microsoft.KeyVault(SecretUri=...)
├── API Management: OAuth 2.0, JWT validation, rate limiting
├── Dependency injection: Structured, testable code
├── Azure Policy: Enforce HTTPS, require managed identity
├── Microsoft Defender for App Service
└── Compliance: SOC 2, ISO 27001

Monitoring:
├── Application Insights: All apps, distributed tracing
├── Smart Detection: Auto-detect anomalies
├── Availability tests: Ping API endpoints from 5 regions
├── SLOs: 99.95% availability, p99 < 500ms
├── Alerts: Adaptive thresholds, multi-condition
├── Log Analytics: Cross-region query, KQL dashboards
├── Cost alerts: Per-app budget tracking
└── Workbooks: Executive dashboards

Cost: $5,000-20,000/month (Premium plans + API Management)
```

---

## Quick Reference

| Feature | Detail |
|---------|--------|
| What | Serverless event-driven compute |
| Languages | C#, JavaScript/TS, Python, Java, PowerShell, Go |
| Triggers | HTTP, Timer, Queue, Blob, Service Bus, Cosmos DB, Event Grid, Event Hub |
| Bindings | Declarative input/output (Cosmos DB, Storage, SQL, etc.) |
| Plan: Consumption | Scale to 0, pay-per-use, 10 min timeout, free tier |
| Plan: Flex Consumption | Scale to 0, always-ready instances, VNet |
| Plan: Premium | Pre-warmed, VNet, unlimited timeout, deployment slots |
| Plan: Dedicated | App Service plan, predictable pricing |
| Durable Functions | Stateful workflows (chaining, fan-out, human interaction) |
| Deployment slots | Staging → production swap (Premium/Dedicated) |
| Cold start | Consumption: 1-3s, Premium: None (pre-warmed) |
| Free tier | 1M executions + 400K GB-s / month |
| AWS equivalent | Lambda |
| GCP equivalent | Cloud Functions |

---

## What's Next?

In the next chapter, we'll cover Azure Container Instances (ACI) — run containers without managing VMs.

→ Next: [Chapter 18: Azure Container Instances](18-aci.md)

---

*Last Updated: May 2026*
