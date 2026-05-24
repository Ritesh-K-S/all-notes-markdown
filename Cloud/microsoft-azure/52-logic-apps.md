# Chapter 52: Azure Logic Apps

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Logic Apps Fundamentals](#part-1-logic-apps-fundamentals)
- [Part 2: Creating a Logic App (Portal Walkthrough)](#part-2-creating-a-logic-app-portal-walkthrough)
- [Part 3: Triggers & Actions](#part-3-triggers--actions)
- [Part 4: Connectors](#part-4-connectors)
- [Part 5: Control Flow & Expressions](#part-5-control-flow--expressions)
- [Part 6: Standard vs Consumption](#part-6-standard-vs-consumption)
- [Part 7: Terraform & az CLI Reference](#part-7-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure Logic Apps is a low-code/no-code workflow automation platform. You build workflows visually by connecting triggers and actions — no coding required. It has 400+ connectors to services like Office 365, Salesforce, SAP, databases, and all Azure services.

```
What you'll learn:
├── Logic Apps Fundamentals
│   ├── What is workflow automation
│   ├── Standard vs Consumption plans
│   └── Pricing models
├── Creating a Logic App (Portal)
├── Triggers & Actions
├── Connectors (400+ integrations)
├── Control Flow (conditions, loops, switches)
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: Logic Apps Fundamentals

```
Logic Apps = Visual workflow automation
"When [trigger] → Do [action1] → Then [action2] → ..."

Example workflows:
├── When email arrives with attachment → Save to Blob → Send Teams message
├── When blob uploaded → Call AI Vision API → Store results in DB
├── When HTTP request received → Insert to SQL → Send confirmation email
├── When new tweet mentions brand → Analyze sentiment → Alert team
├── Every day at 8 AM → Query database → Generate report → Email to team

How it works:
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Trigger  │ → │ Action 1 │ → │ Action 2 │ → │ Action 3 │
│ (When)   │   │ (Do)     │   │ (Do)     │   │ (Do)     │
└──────────┘   └──────────┘   └──────────┘   └──────────┘

Trigger: Starts the workflow (new email, HTTP request, schedule, etc.)
Action: A step in the workflow (send email, create blob, call API)
Connector: Integration with a service (Office 365, SQL, Salesforce)
```

---

## Part 2: Creating a Logic App (Portal Walkthrough)

```
Console → Logic Apps → Create

┌─────────────────────────────────────────────────────────────────────┐
│           CREATE LOGIC APP                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Plan type:                                                           │
│ ● Consumption (serverless, pay per execution)                     │
│ ○ Standard (dedicated, multiple workflows per app)                │
│                                                                       │
│ Subscription: [Pay-As-You-Go ▼]                                    │
│ Resource group: [rg-automation ▼]                                  │
│ Logic App name: [logic-process-orders]                             │
│ Region: [Central India ▼]                                          │
│                                                                       │
│ [Review + Create]                                                   │
│                                                                       │
│ After creation → Logic App Designer opens                         │
│                                                                       │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ LOGIC APP DESIGNER                                            │   │
│ │                                                               │   │
│ │ Start with a common trigger:                                │   │
│ │ ├── When a HTTP request is received                        │   │
│ │ ├── Recurrence (schedule)                                   │   │
│ │ ├── When a new email arrives (Outlook/Gmail)               │   │
│ │ ├── When a blob is added to Storage                        │   │
│ │ └── When a message is received in Service Bus              │   │
│ │                                                               │   │
│ │ [+ New step] ← Add actions after trigger                   │   │
│ │                                                               │   │
│ │ Search connectors: [Search 400+ connectors...]             │   │
│ │                                                               │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Example: HTTP Request → Insert to SQL → Send Email

```
Designer:

[Trigger: When a HTTP request is received]
    URL: https://prod-01.centralindia.logic.azure.com/workflows/...
    Method: POST
    JSON Schema: { "orderId": "string", "amount": "number" }
         │
         ▼
[Action: Insert row - Azure SQL]
    Server: sql-myapp.database.windows.net
    Database: orders-db
    Table: Orders
    Row: orderId = @{triggerBody()?['orderId']}
         │
         ▼
[Action: Send an email - Office 365 Outlook]
    To: ops@company.com
    Subject: New Order @{triggerBody()?['orderId']}
    Body: Amount: $@{triggerBody()?['amount']}
```

---

## Part 3: Triggers & Actions

```
Trigger types:
├── Polling: Checks periodically (every X minutes)
│   └── "When a new email arrives" (checks every 3 min)
├── Push (Webhook): Instant notification
│   └── "When a HTTP request is received"
├── Recurrence: Time-based schedule
│   └── "Every day at 8:00 AM"
└── Manual: Run on demand

Popular actions:
├── HTTP: Call any REST API
├── Parse JSON: Extract data from JSON
├── Compose: Create data objects
├── Create blob: Save to Azure Storage
├── Send email: Outlook, Gmail, SendGrid
├── Insert row: SQL, Cosmos DB
├── Send message: Service Bus, Teams, Slack
├── Execute Function: Call Azure Functions
└── Variables: Initialize, set, increment
```

---

## Part 4: Connectors

```
Built-in connectors (run in Logic Apps runtime, faster):
├── HTTP, Request/Response
├── Schedule, Recurrence
├── Azure Functions, Azure Service Bus
├── Variables, Control (condition, loop, switch)
└── Inline Code (JavaScript)

Managed connectors (Microsoft-hosted, 400+):
├── Azure: Storage, SQL, Cosmos, Event Grid, Key Vault
├── Microsoft 365: Outlook, Teams, SharePoint, OneDrive
├── Business: Salesforce, SAP, Dynamics 365, ServiceNow
├── Social: Twitter, Slack
├── Data: SQL Server, Oracle, IBM DB2, FTP, SFTP
└── AI: Azure AI, OpenAI, Cognitive Services

Standard connectors: Included in base price
Premium connectors: Additional cost per execution
├── SAP, IBM MQ, Salesforce are premium
└── ~$0.001 per premium action

Custom connectors:
├── Wrap any REST API as a Logic Apps connector
├── Define triggers, actions, authentication
└── Share across Logic Apps in your organization
```

---

## Part 5: Control Flow & Expressions

```
Control actions:
├── Condition (If/Else):
│   If @{triggerBody()?['amount']} > 1000
│   ├── Yes branch: Send approval email
│   └── No branch: Auto-approve
│
├── Switch:
│   Switch on @{triggerBody()?['status']}
│   ├── Case "new": Process new order
│   ├── Case "cancelled": Refund
│   └── Default: Log unknown status
│
├── For Each: Loop over arrays
│   For each item in @{body('Get_rows')?['value']}
│   └── Send email for each item
│
├── Until: Loop until condition is true
│   Until @{variables('retryCount')} >= 3
│   └── Try calling API, increment retryCount
│
├── Scope: Group actions together (try-catch)
│   Scope (Try) → Actions
│   Scope (Catch) → Run after "Try" has failed → Error handling
│
└── Terminate: End workflow with status (Succeeded/Failed/Cancelled)

Expressions (use in action inputs):
├── @{triggerBody()?['name']}     → Access trigger data
├── @{body('ActionName')}          → Access action output
├── @{utcNow()}                     → Current UTC time
├── @{concat('Hello ', variables('name'))} → String concat
├── @{length(body('Get_rows')?['value'])}  → Array length
└── @{if(equals(x,y),'yes','no')} → Inline if
```

---

## Part 6: Standard vs Consumption

```
┌────────────────────┬───────────────────┬───────────────────┐
│                    │ Consumption       │ Standard          │
├────────────────────┼───────────────────┼───────────────────┤
│ Hosting            │ Serverless (multi)│ Dedicated (single)│
│ Workflows per app  │ 1                 │ Multiple          │
│ Designer           │ Azure Portal      │ Portal + VS Code  │
│ VNet integration   │ No                │ Yes ✅            │
│ Running locally    │ No                │ Yes (VS Code)     │
│ Stateful/Stateless │ Stateful only     │ Both              │
│ Pricing            │ Per execution     │ App Service Plan  │
│ Built-in storage   │ Azure managed     │ Your Storage Acct │
│ ISE (legacy)       │ Replaced by Std   │ N/A               │
├────────────────────┼───────────────────┼───────────────────┤
│ Best for           │ Simple, low-vol   │ Enterprise, VNet  │
│                    │ event-driven      │ high-volume       │
└────────────────────┴───────────────────┴───────────────────┘

Consumption pricing:
├── Actions: ~$0.000025 per action execution
├── Standard connectors: ~$0.000125 per execution
├── Premium connectors: ~$0.001 per execution
├── First 4,000 actions/month FREE
└── Very cheap for low-volume workflows

Standard pricing:
├── Based on App Service Plan (WS1/WS2/WS3)
├── WS1: ~$152/month (1 vCPU, 3.5 GB RAM)
└── Better for high-volume, VNet-required scenarios
```

---

## Part 7: Terraform & az CLI Reference

### Terraform (Consumption)

```hcl
resource "azurerm_logic_app_workflow" "orders" {
  name                = "logic-process-orders"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  workflow_parameters = {
    "$connections" = jsonencode({
      defaultValue = {}
      type         = "Object"
    })
  }
}

resource "azurerm_logic_app_trigger_http_request" "trigger" {
  name         = "http-trigger"
  logic_app_id = azurerm_logic_app_workflow.orders.id
  schema       = <<SCHEMA
  {
    "type": "object",
    "properties": {
      "orderId": { "type": "string" },
      "amount": { "type": "number" }
    }
  }
  SCHEMA
}
```

### Bicep

```bicep
// Logic App (Consumption)
resource logicApp 'Microsoft.Logic/workflows@2019-05-01' = {
  name: 'logic-order-process'
  location: resourceGroup().location
  properties: {
    state: 'Enabled'
    definition: {
      '$schema': 'https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#'
      contentVersion: '1.0.0.0'
      triggers: {
        manual: {
          type: 'Request'
          kind: 'Http'
          inputs: {
            schema: {
              type: 'object'
              properties: {
                orderId: { type: 'string' }
                amount: { type: 'number' }
              }
            }
          }
        }
      }
      actions: {}
    }
  }
}

// Logic App (Standard) - runs on App Service Plan
resource logicAppStandard 'Microsoft.Web/sites@2023-01-01' = {
  name: 'logic-std-myapp'
  location: resourceGroup().location
  kind: 'functionapp,workflowapp'
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        { name: 'FUNCTIONS_EXTENSION_VERSION', value: '~4' }
        { name: 'FUNCTIONS_WORKER_RUNTIME', value: 'node' }
      ]
    }
  }
}
```
# Create Logic App (Consumption)
az logic workflow create \
  --name logic-process-orders \
  --resource-group rg-automation \
  --location centralindia

# Show workflow
az logic workflow show --name logic-process-orders --resource-group rg-automation

# List runs
az logic workflow-run list --workflow-name logic-process-orders --resource-group rg-automation

# Disable/Enable
az logic workflow update --name logic-process-orders --resource-group rg-automation --state Disabled
az logic workflow update --name logic-process-orders --resource-group rg-automation --state Enabled

# Delete
az logic workflow delete --name logic-process-orders --resource-group rg-automation --yes
```

---

## Real-World Patterns

### Pattern 1: Automated Invoice Processing

```
┌─────────────────────────────────────────────────┐
│       Invoice Processing Workflow               │
├─────────────────────────────────────────────────┤
│                                                 │
│  Email Arrives (Outlook Trigger)                │
│       │                                         │
│       ▼                                         │
│  Has Attachment? ──No──→ Skip                   │
│       │ Yes                                     │
│       ▼                                         │
│  Save PDF to SharePoint                         │
│       │                                         │
│       ▼                                         │
│  AI Builder: Extract Invoice Data               │
│  (vendor, amount, date, line items)             │
│       │                                         │
│       ▼                                         │
│  Amount > $5000? ──Yes──→ Manager Approval      │
│       │ No                    (Teams)           │
│       ▼                       │                 │
│  Create Record in Dynamics 365 ←────────────────┘
│       │                                         │
│       ▼                                         │
│  Send Confirmation Email                        │
│                                                 │
│  Connectors: Outlook, SharePoint, AI Builder,   │
│  Teams, Dynamics 365                            │
└─────────────────────────────────────────────────┘
```

### Pattern 2: Multi-System Data Sync

```
┌─────────────────────────────────────────────────┐
│       CRM → ERP Data Synchronization            │
├─────────────────────────────────────────────────┤
│                                                 │
│  Salesforce (New Deal Won)                      │
│       │                                         │
│       ▼                                         │
│  Logic App (Recurrence: every 5 min)            │
│       │                                         │
│       ├──→ Transform Data (Compose action)      │
│       │                                         │
│       ├──→ Create Customer in SAP               │
│       │                                         │
│       ├──→ Create Project in Jira               │
│       │                                         │
│       └──→ Notify Sales Team (Slack)            │
│                                                 │
│  Error Handling:                                │
│  - Retry policy: 4 retries, exponential         │
│  - Dead-letter to Storage Queue on failure      │
│  - Alert via email if 3+ failures/hour          │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Logic Apps = Low-code visual workflow automation
400+ connectors: Office 365, Salesforce, SAP, Azure services, etc.

Trigger → Action → Action → Action (visual pipeline)
Triggers: HTTP request, schedule, event, polling
Control: Condition, Switch, For Each, Until, Scope (try-catch)

Consumption: Serverless, pay per execution, simple workflows
Standard: Dedicated, VNet support, multiple workflows, VS Code dev

Expressions: @{triggerBody()?['field']}, @{body('Action')}, @{utcNow()}

Pricing (Consumption):
├── First 4,000 actions FREE
├── ~$0.000025/action
└── Premium connectors: ~$0.001/action
```

---

## What's Next?

Next chapter: [Chapter 53: API Management (APIM)](53-api-management.md) — API gateway for managing, securing, and publishing APIs to developers.
