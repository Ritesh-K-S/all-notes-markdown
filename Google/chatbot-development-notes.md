# Google Chat Bot - Development Notes

> **Project Start Date:** April 30, 2026  
> **Status:** Planning  
> **Last Updated:** April 30, 2026  
> **Platform:** Gemini Enterprise Agent Platform (formerly Vertex AI)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Google Naming / Rebranding Guide](#2-google-naming--rebranding-guide)
3. [Architecture](#3-architecture)
4. [Tools & Services Selected](#4-tools--services-selected)
5. [Phase 1 — GCP Setup](#5-phase-1--gcp-setup)
6. [Phase 2 — Conversational Agent (Dialogflow CX)](#6-phase-2--conversational-agent-dialogflow-cx)
7. [Phase 3 — Backend Webhook (Cloud Function / ADK)](#7-phase-3--backend-webhook-cloud-function--adk)
8. [Phase 4 — Connect to Google Chat](#8-phase-4--connect-to-google-chat)
9. [Phase 5 — Voice Support](#9-phase-5--voice-support)
10. [Phase 6 — Testing & Deployment](#10-phase-6--testing--deployment)
11. [API List & Intent/Tool Mapping](#11-api-list--intenttool-mapping)
12. [Issues & Troubleshooting Log](#12-issues--troubleshooting-log)
13. [Updates Log](#13-updates-log)

---

## 1. Project Overview

**Goal:** Build a Google Chat bot that takes user questions (text + voice), understands intent, calls different APIs based on the question, and responds with the result.

**Key Requirements:**
- Works inside Google Chat (text input)
- Supports voice-based input
- Routes to different APIs based on user intent
- **Conversational — bot must remember previous context within a session**
- Multi-turn conversation with follow-up questions and context carry-over

---

## 2. Google Naming / Rebranding Guide

> **IMPORTANT:** Google has heavily rebranded its AI/agent products. This section tracks old vs. new names so we don't use outdated terms.

| Old Name (Pre-2025) | Current Name (2026) | Notes |
|---------------------|---------------------|-------|
| Vertex AI | **Gemini Enterprise Agent Platform** ("Agent Platform") | The umbrella platform for all AI/ML/agent work |
| Dialogflow CX Console | **Conversational Agents Console** | New URL: `conversational-agents.cloud.google.com` (old CX console still works for now) |
| Dialogflow CX | **Conversational Agents (Dialogflow CX)** | CX engine still powers it under the hood, name is shifting |
| Intents + Flows only | **Playbooks (Generative) + Flows (Deterministic)** | Two approaches now: Playbooks use LLM reasoning, Flows use traditional intent matching |
| N/A | **Playbook Tools** | Playbooks can call OpenAPI tools, Data Store tools, Code Blocks — replaces some webhook needs |
| N/A | **Agent Development Kit (ADK)** | Code-based framework for building sophisticated agents |
| N/A | **Agent Studio** | Low-code visual canvas in the console |
| N/A | **Agent Platform Sessions** | Manages stateful context within a session |
| N/A | **Agent Platform Memory Bank** | Persistent memory ACROSS sessions (remembers users over time) |
| N/A | **Agent Runtime** | Deployment runtime with sub-second cold starts |
| N/A | **Agent Gateway** | Security/policy enforcement for agent tool calls |

### Two Agent Building Approaches

| Approach | Best For | How It Works |
|----------|----------|-------------|
| **Playbooks (Generative)** | Flexible, natural conversations; complex reasoning | LLM (Gemini) reasons through instructions + calls Tools. Less rigid. |
| **Flows (Deterministic)** | Strict, predictable routing; compliance-heavy | Traditional Intents → Pages → Webhooks. Full control over every path. |
| **Hybrid** | Best of both | Use Flows for structured paths, hand off to Playbooks for open-ended parts |

> **For our chatbot:** We can use either approach or a hybrid. Playbooks are newer and more powerful for calling different APIs dynamically. Flows give more control.

### Console URLs

| Console | URL | Status |
|---------|-----|--------|
| **Conversational Agents Console** (new) | https://conversational-agents.cloud.google.com | Active — use this |
| Dialogflow CX Console (legacy) | https://dialogflow.cloud.google.com/cx/projects | Still works, being migrated |
| Agent Platform Console | https://console.cloud.google.com/agent-platform | Active |

---

## 3. Architecture

```
User (Text/Voice)
      │
      ▼
Google Chat  ──────►  Conversational Agent        ──────►  Cloud Function / ADK (Webhook)
                       (Dialogflow CX engine)                     │
                                                                  ├── API 1 (via Playbook Tools)
                       Option A: Playbooks (Generative)           ├── API 2 (via Playbook Tools)
                         - LLM reasons through instructions       ├── API 3 (via Webhook)
                         - Calls Tools (OpenAPI / Code Block)     └── API N
                         - Session + Memory Bank                        │
                                                              ┌────────┴────────┐
                       Option B: Flows (Deterministic)        │  Agent Platform  │
                         - Intents → Pages → Webhooks         │  Sessions &      │
                         - Session Parameters                 │  Memory Bank     │
                         - Strict routing                     └─────────────────┘
                                                              Remembers context within
                                                              session AND across sessions
```

### How Context/Memory Works (Current Platform)

| Mechanism | What It Does | Scope | Available In |
|-----------|-------------|-------|--------------|
| **Agent Platform Sessions** | Manages stateful data and context within a single interaction | Per conversation session | Playbooks & Flows |
| **Agent Platform Memory Bank** | Persistent memory — recalls info across MULTIPLE sessions | Across sessions (per user) | Playbooks |
| **Session Parameters** | Stores extracted entities (e.g., city, date) across turns | Per conversation session | Flows |
| **Flows & Pages** | Tracks where the user is in a multi-step conversation | Per session | Flows |
| **Session ID** | Google Chat sends same session ID for a thread | Per chat thread | Both |
| **Webhook Session Params** | Webhook can read AND write session parameters across turns | Per session | Both |

**Example — Context in Action:**
```
User: "What's the weather in Mumbai?"
Bot:  "It's 32°C and sunny in Mumbai."
      (stores: location = Mumbai)

User: "What about tomorrow?"
Bot:  "Tomorrow in Mumbai it will be 30°C with rain."
      (remembers location = Mumbai from previous turn, only needs new param: date = tomorrow)
```

**Voice Path:**
```
Voice Input  ──►  Conversational Agent Phone Gateway / Dialogflow Messenger Widget
                        │
                        ▼
                  Speech-to-Text  ──►  Conversational Agent  ──►  Tools/Webhook  ──►  APIs
                                         (Dialogflow CX)                         ──►  Text-to-Speech
```

---

## 4. Tools & Services Selected

| Component          | Tool / Service (Current Name)                    | Purpose                              | Status      |
|--------------------|--------------------------------------------------|--------------------------------------|-------------|
| Platform           | **Gemini Enterprise Agent Platform**             | Umbrella platform (formerly Vertex AI)| Not Started |
| Chat Interface     | Google Chat API                                  | Bot lives inside Google Chat         | Not Started |
| Agent Engine       | **Conversational Agents (Dialogflow CX)**        | NLU, intent, playbooks, flows        | Not Started |
| Agent Console      | **Conversational Agents Console**                | Build/test agent (new consolidated console) | Not Started |
| Agent Building     | **Playbooks + Flows** (TBD which approach)       | Generative and/or deterministic routing | Not Started |
| Backend Logic      | Cloud Functions (Gen 2) / **Agent Dev Kit (ADK)**| Webhook or code-based agent logic    | Not Started |
| API Calling        | **Playbook Tools** (OpenAPI) + Webhooks          | Call external APIs per intent/tool   | Not Started |
| Session Memory     | **Agent Platform Sessions**                      | Context within a conversation        | Not Started |
| Persistent Memory  | **Agent Platform Memory Bank**                   | Remember user info across sessions   | Not Started |
| Voice Support      | Phone Gateway + Speech-to-Text                   | Voice input/output                   | Not Started |
| Deployment         | **Agent Runtime**                                | Deploy with sub-second cold starts   | Not Started |
| Security           | **Agent Gateway**                                | Policy enforcement for tool calls    | Not Started |
| Monitoring         | Cloud Logging + **Observability tools**          | Logs, tracing, debugging             | Not Started |

---

## 5. Phase 1 — GCP Setup

### Step 1: Create a GCP Project

1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Click the **project dropdown** at the top bar (next to "Google Cloud" logo)
3. Click **"New Project"** (top-right of the popup)
4. Enter:
   - **Project name:** `___________` (e.g., `my-chatbot`)
   - **Organization:** Select your org (or "No organization" for personal)
   - **Location:** Select folder or leave default
5. Click **"Create"**
6. Wait for the notification that the project is created
7. **Switch to the new project** using the same project dropdown

- [ ] Project created (Project ID: `___________`)

### Step 2: Set Up Billing

> APIs won't work without a billing account linked, even for free tier usage.

1. Go to **Navigation menu (☰)** → **Billing**
2. If no billing account exists:
   - Click **"Link a billing account"** → **"Create billing account"**
   - Enter payment details (credit card — you won't be charged for free tier)
3. If billing account exists:
   - Select the billing account → Click **"Set Account"**
4. New customers get **$300 free credits** + **$600 Dialogflow CX trial credit**

- [ ] Billing account linked

### Step 3: Enable APIs (one by one)

> **How to enable any API:**
> 1. Go to **Navigation menu (☰)** → **APIs & Services** → **Library**
> 2. In the search bar, type the API name
> 3. Click on the API from the results
> 4. Click the **"Enable"** button
> 5. Wait for it to enable (takes a few seconds)
>
> **OR use the direct links below** — each link opens the API page directly, just click "Enable":

#### API 1: Google Chat API
- **Search for:** `Google Chat API`
- **Direct link:** https://console.cloud.google.com/apis/library/chat.googleapis.com
- Click **Enable**
- [ ] Enabled

#### API 2: Dialogflow API
- **Search for:** `Dialogflow API`
- **Direct link:** https://console.cloud.google.com/apis/library/dialogflow.googleapis.com
- This covers Conversational Agents / Dialogflow CX
- Click **Enable**
- [ ] Enabled

#### API 3: Cloud Functions API
- **Search for:** `Cloud Functions API`
- **Direct link:** https://console.cloud.google.com/apis/library/cloudfunctions.googleapis.com
- Click **Enable**
- [ ] Enabled

#### API 4: Cloud Build API
- **Search for:** `Cloud Build API`
- **Direct link:** https://console.cloud.google.com/apis/library/cloudbuild.googleapis.com
- Required for deploying Cloud Functions
- Click **Enable**
- [ ] Enabled

#### API 5: Cloud Speech-to-Text API
- **Search for:** `Cloud Speech-to-Text API`
- **Direct link:** https://console.cloud.google.com/apis/library/speech.googleapis.com
- Required for voice support
- Click **Enable**
- [ ] Enabled

#### API 6: Cloud Text-to-Speech API (Optional — for voice responses)
- **Search for:** `Cloud Text-to-Speech API`
- **Direct link:** https://console.cloud.google.com/apis/library/texttospeech.googleapis.com
- Click **Enable**
- [ ] Enabled

#### Verify All APIs Are Enabled
1. Go to **Navigation menu (☰)** → **APIs & Services** → **Enabled APIs & Services**
2. You should see all the above APIs listed
- [ ] All verified

### Step 4: Create a Service Account

1. Go to **Navigation menu (☰)** → **IAM & Admin** → **Service Accounts**
2. Click **"+ Create Service Account"**
3. Enter:
   - **Service account name:** `chatbot-service` (or any name)
   - **Description:** `Service account for chatbot webhook`
4. Click **"Create and Continue"**
5. Grant roles (add these one by one):
   - `Dialogflow API Client` — to call Dialogflow
   - `Cloud Functions Invoker` — to invoke functions
   - `Dialogflow API Admin` — if you need to manage agents programmatically
6. Click **"Continue"** → **"Done"**
7. (Optional) Click the service account → **Keys** tab → **Add Key** → **Create new key** → **JSON** → Download and store securely

- [ ] Service account created
- [ ] Key file downloaded (if needed): stored at `___________`

### Step 5: Verify Console Access

1. Open [Conversational Agents Console](https://conversational-agents.cloud.google.com)
2. Select your project from the dropdown
3. You should see the agent builder interface
- [ ] Console access verified

### Notes

_(Add notes here as you complete this phase)_

---

## 6. Phase 2 — Conversational Agent (Dialogflow CX)

### First Decision: Playbooks vs Flows vs Hybrid

| Criteria | Playbooks (Generative) | Flows (Deterministic) |
|----------|----------------------|----------------------|
| How it routes | LLM (Gemini) reads instructions and decides | Strict intent matching → page transitions |
| API calling | **Playbook Tools** (define OpenAPI spec, LLM calls automatically) | Webhooks (you code the routing logic) |
| Flexibility | High — handles unexpected questions gracefully | Low — only handles trained intents |
| Control | Less rigid — LLM decides | Full control over every path |
| Context memory | Agent Platform Sessions + Memory Bank | Session Parameters |
| Best for | Dynamic, open-ended bots calling many APIs | Strict, compliance-heavy bots |

> **Decision:** `___________` (Playbooks / Flows / Hybrid)

### Steps (Common)

#### Step 1: Create Agent

1. Go to [Conversational Agents Console](https://conversational-agents.cloud.google.com)
2. Select your GCP project from the top dropdown
3. Click **"Create Agent"**
4. You'll see 3 options:

| Option | What it does | Choose? |
|--------|-------------|---------|
| Use a prebuilt agent | Loads a ready-made template (travel, telecom, etc.) | No — your bot has custom API logic |
| **Build your own** | Blank agent — you define your own Playbooks/Flows/Tools | **YES — select this** |
| Create Q&A agent | Data Store agent that answers from docs/websites | No — you need to call APIs, not a knowledge base |

5. Select **"Build your own"**
6. Fill in the form:
   - **Display name:** `___________` (e.g., `my-chatbot`)
   - **Location / Region:** `___________` (e.g., `us-central1`, `asia-south1` for India)
     > ⚠️ **Cannot change region later.** Pick one close to your users.
   - **Time zone:** Your local timezone
   - **Default language:** `English`
     > ⚠️ **Cannot change default language later.**
7. Click **"Create"** / **"Save"**

- [x] Agent created (Name: `___________`, Region: `___________`)

#### Step 2: Choose Approach (Playbooks vs Flows)

- [x] Approach chosen: **Playbooks (Generative)**

### Agent Console — Sidebar Reference

> After creating the agent, you land on the agent console. Here's what each sidebar section does:

| Sidebar Section | What It Is | When To Use |
|-----------------|-----------|-------------|
| **Agent Overview** | Dashboard — agent settings, default language, region, linked Gemini model | To change agent-level settings |
| **Playbooks** | Define bot behavior — goals, instructions, examples. The brain of your agent. | **NOW — start here** |
| **Flows** | Deterministic paths (Intents → Pages → Webhooks). Can work alongside Playbooks (hybrid). | Only if you need strict routing for some conversations |
| **Tools** | Define external APIs your Playbook can call — OpenAPI spec, Data Store, Code Block | **NEXT — define your APIs here** |
| **Prebuilts** | Ready-made components (e.g., prebuilt intents: "yes", "no", "cancel", small talk) | Optional — can add later for common utterances |
| | | |
| **Test Cases** | Saved test conversations for regression testing your agent | After building — for QA |
| **Conversation History** | Logs of real user conversations with the bot | After deployment — for debugging & improvement |
| | | |
| **_Deploy Section_** | | |
| **Environments** | Separate deployments: `draft` (dev), `staging`, `production` | When deploying — to separate dev from prod |
| **Versions** | Snapshots of your agent at a point in time (like git commits) | Before deploying — save a stable version to promote |
| | | |
| **_Integration Section_** | | |
| **Integrations** | Connect agent to channels: **Google Chat**, Dialogflow Messenger widget, Phone Gateway, etc. | Phase 4 — to make bot available in Google Chat |
| **Conversation Profiles** | Config for Contact Center AI (CCAI) — STT/TTS settings, agent assist suggestions, speech models | Phase 5 (voice) — only needed for call center / telephony setups |

### What To Do Next (In Order)

```
Step 3: Playbooks  → Create main playbook (goal + instructions)
Step 4: Tools      → Define the APIs the playbook should call
Step 5: Test       → Use the built-in simulator (chat icon, right side)
Step 6: Integrate  → Connect to Google Chat (Phase 4)
Step 7: Voice      → Set up voice support (Phase 5)
```

### Steps — If Using Playbooks (Generative)

#### Step 3: Create Your Main Playbook

1. Click **"Playbooks"** in the sidebar
2. You'll see a default playbook (usually `Default Generative Agent`) — click it, or create a new one
3. Fill in:
   - **Goal:** A clear one-line statement of what this playbook does
     > Example: *"Help users by answering questions, calling APIs for weather, reports, and calendar, and responding with the results."*
   - **Instructions:** Step-by-step rules the LLM follows (natural language)
     > Example:
     > ```
     > - Greet the user and ask how you can help
     > - If the user asks about weather, use the weather tool to get the answer
     > - If the user asks about reports, use the reports tool
     > - Always confirm the result with the user
     > - If you don't understand, ask the user to clarify
     > - Remember context from previous messages in the conversation
     > ```
4. Click **"Save"**

- [ ] Main playbook created

#### Step 4: Define Tools (APIs)

1. Click **"Tools"** in the sidebar
2. Click **"+ Create"**
3. Choose tool type:
   - **OpenAPI** — provide your API's OpenAPI/Swagger spec (YAML or JSON). The LLM will call it automatically based on instructions.
   - **Data Store** — if you want to query a document/website knowledge base
   - **Code Block** — write inline Python/JavaScript for custom logic
4. For each API you want the bot to call, create a separate Tool
5. After creating tools, go back to your Playbook and **reference the tools in the instructions**

- [ ] Tools defined (list them in Section 11)

#### Step 5: Add Examples (Optional but Recommended)

1. In your Playbook, go to the **"Examples"** tab
2. Add sample conversations showing the ideal bot behavior
3. This helps the LLM understand exactly how to respond

- [ ] Examples added

### Steps — If Using Flows (Deterministic)

- [ ] Define **Intents** (one per API/action)
- [ ] Add **Training Phrases** for each intent (10-20 examples each)
- [ ] Define **Entity Types** for parameters (dates, names, locations, etc.)
- [ ] Create **Flows** and **Pages** for conversation structure
- [ ] Set up **Default Welcome Intent** and **Fallback Intent**
- [ ] Configure **Session Parameters** for context carry-over between turns
- [ ] Set up **Conditional Routes** that use session params from previous turns
- [ ] Configure **Webhooks** for API fulfillment
- [ ] Test multi-turn conversations (follow-up questions using previous context)

### Intents / Playbook Tools Created

| Name | Type (Intent / Playbook Tool) | Description | Status |
|------|-------------------------------|-------------|--------|
| _(add here)_ | | | |

### Notes

_(Add notes here as you complete this phase)_

---

## 7. Phase 3 — Test Your Agent

### Step 7: Test in Console Simulator

> The Conversational Agents Console has a built-in chat simulator on the **right side** of the screen (speech bubble icon).

1. Click the **"Test Agent"** button / chat icon on the right panel
2. Type a message that should trigger one of your tools
3. Check:
   - Did the agent understand the intent correctly?
   - Did it call the right Tool?
   - Did it pass the right parameters to the API?
   - Is the response formatted well?
4. Test **multi-turn** — ask a follow-up that relies on previous context
5. Test **edge cases** — gibberish input, missing parameters, ambiguous questions
6. Check the **"Agent Response"** section in the simulator to see the full trace:
   - Which Playbook was triggered
   - Which Tool was called
   - What parameters were sent
   - What the API returned

#### Test Scenarios to Try

| # | Test Input | Expected Behavior | Pass? |
|---|-----------|-------------------|-------|
| 1 | _(your test message)_ | _(expected tool call + response)_ | |
| 2 | Follow-up referencing previous context | Should remember previous params | |
| 3 | Unclear/ambiguous input | Should ask for clarification | |
| 4 | Completely off-topic input | Should handle gracefully (fallback) | |

- [ ] Basic single-turn tests passing
- [ ] Multi-turn context tests passing
- [ ] Edge case / fallback tests passing

### Notes

_(Add test results and observations here)_

---

## 8. Phase 4 — Connect to Google Chat (Python)

> There are **two ways** to connect your agent to Google Chat:
>
> **Option A:** Direct Dialogflow Integration (no code — Google Chat talks to Dialogflow directly)
> **Option B:** Python webhook (you write a service that receives Google Chat messages, sends them to Dialogflow CX API, and returns the response)
>
> **Option A** is simpler. **Option B** gives you more control (logging, custom formatting, pre/post processing).

### Option A — Direct Integration (No Code)

#### Step 8A: Configure Google Chat API

1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Make sure your project is selected
3. Go to **Navigation menu (☰)** → **APIs & Services** → **Enabled APIs & Services**
4. Find **"Google Chat API"** in the list → **Click on it**
   > ⚠️ Do NOT go to API Library (that only shows "Manage" / "Try this API").
   > You need the **Enabled API detail page** which has the **Configuration** tab.
5. Click the **"Configuration"** tab at the top
   > **Direct URL:** `https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat`
6. Fill in:

| Field | Value |
|-------|-------|
| **App name** | `___________` (the name users will see in Google Chat) |
| **Avatar URL** | _(optional — URL to a bot icon)_ |
| **Description** | `___________` |
| **Enable Interactive features** | **ON** (toggle this ON) |
| **Functionality** | Check: ✅ `Receive 1:1 messages` and optionally ✅ `Join spaces and group conversations` |
| **Connection settings** | Select **"Dialogflow"** |
| **Dialogflow agent resource name** | Select your agent from the dropdown (it should auto-populate) |
| **Visibility** | Choose who can see the bot: everyone in your org / specific people |

6. Click **"Save"**

- [ ] Google Chat API configured

#### Step 8A-2: Test in Google Chat

1. Open [Google Chat](https://chat.google.com)
2. Click **"+ New chat"** or **"Find apps"**
3. Search for your bot name
4. Send a message — it should respond via your Dialogflow Playbook + Tools

- [ ] Bot responding in Google Chat

---

### Option B — Python Webhook (More Control)

> Use this if you need custom logic between Google Chat and Dialogflow (logging, message formatting, auth checks, etc.)

#### Step 8B-1: Set Up Python Project

```
google-chat-bot/
├── main.py              # Webhook handler
├── requirements.txt     # Dependencies
├── .env                 # Environment variables (DO NOT commit)
└── README.md
```

#### Step 8B-2: Install Dependencies

```bash
pip install flask google-cloud-dialogflow-cx google-auth gunicorn
```

**requirements.txt:**
```
flask==3.1.*
google-cloud-dialogflow-cx==1.*
google-auth==2.*
gunicorn==22.*
```

#### Step 8B-3: Python Webhook Code

```python
"""
Google Chat Bot → Dialogflow CX Webhook
Receives messages from Google Chat, sends to Dialogflow CX, returns response.
"""

import os
import uuid
from flask import Flask, request, jsonify
from google.cloud import dialogflowcx_v3 as dialogflow

app = Flask(__name__)

# --- Configuration ---
PROJECT_ID = os.environ.get("GCP_PROJECT_ID", "your-project-id")
LOCATION = os.environ.get("DIALOGFLOW_LOCATION", "us-central1")  # your agent's region
AGENT_ID = os.environ.get("DIALOGFLOW_AGENT_ID", "your-agent-id")
LANGUAGE_CODE = "en"

def get_dialogflow_response(user_message: str, session_id: str) -> str:
    """Send user message to Dialogflow CX and get response."""
    
    client = dialogflow.SessionsClient(
        client_options={
            "api_endpoint": f"{LOCATION}-dialogflow.googleapis.com"
        }
    )

    session_path = client.session_path(
        project=PROJECT_ID,
        location=LOCATION,
        agent=AGENT_ID,
        session=session_id,
    )

    text_input = dialogflow.TextInput(text=user_message)
    query_input = dialogflow.QueryInput(
        text=text_input,
        language_code=LANGUAGE_CODE,
    )

    response = client.detect_intent(
        request=dialogflow.DetectIntentRequest(
            session=session_path,
            query_input=query_input,
        )
    )

    # Collect all response messages
    response_texts = []
    for msg in response.query_result.response_messages:
        if msg.text:
            response_texts.extend(msg.text.text)

    return "\n".join(response_texts) if response_texts else "Sorry, I didn't understand that."


@app.route("/", methods=["POST"])
def handle_chat():
    """Handle incoming Google Chat messages."""
    
    event = request.get_json()
    event_type = event.get("type", "")

    # --- Handle different event types ---
    if event_type == "ADDED_TO_SPACE":
        return jsonify({"text": "Thanks for adding me! How can I help you?"})

    if event_type == "REMOVED_FROM_SPACE":
        return jsonify({})

    if event_type == "MESSAGE":
        user_message = event.get("message", {}).get("text", "").strip()
        
        if not user_message:
            return jsonify({"text": "Please send a text message."})

        # Use the Chat space/thread as session ID for context continuity
        space_name = event.get("space", {}).get("name", "")
        thread_name = event.get("message", {}).get("thread", {}).get("name", "")
        session_id = thread_name if thread_name else space_name
        # Clean session ID (Dialogflow only allows alphanumeric, hyphens, underscores)
        session_id = session_id.replace("/", "-").replace(" ", "")
        if not session_id:
            session_id = str(uuid.uuid4())

        bot_response = get_dialogflow_response(user_message, session_id)
        return jsonify({"text": bot_response})

    return jsonify({"text": ""})


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)), debug=True)
```

#### Step 8B-4: Deploy to Cloud Run (Recommended for Python)

```bash
# Set your project
gcloud config set project YOUR_PROJECT_ID

# Deploy to Cloud Run
gcloud run deploy google-chat-bot \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars "GCP_PROJECT_ID=your-project-id,DIALOGFLOW_LOCATION=us-central1,DIALOGFLOW_AGENT_ID=your-agent-id"
```

> After deployment, you'll get a URL like: `https://google-chat-bot-xxxxx-uc.a.run.app`

- [ ] Deployed to Cloud Run
- [ ] Deployment URL: `___________`

#### Step 8B-5: Configure Google Chat API to Use Your Webhook

1. Go to [Google Cloud Console](https://console.cloud.google.com) → Search **"Google Chat API"** → **Configuration**
2. Fill in:

| Field | Value |
|-------|-------|
| **App name** | `___________` |
| **Enable Interactive features** | **ON** |
| **Functionality** | ✅ Receive 1:1 messages |
| **Connection settings** | Select **"HTTP endpoint URL"** (NOT Dialogflow) |
| **HTTP endpoint URL** | Paste your Cloud Run URL: `https://google-chat-bot-xxxxx-uc.a.run.app` |
| **Visibility** | Your org / specific people |

3. Click **"Save"**

- [ ] Google Chat API configured with webhook URL

#### Step 8B-6: Test in Google Chat

1. Open [Google Chat](https://chat.google.com)
2. Search for your bot → Send a message
3. It should go: **Google Chat → your Python webhook → Dialogflow CX → Tools/APIs → response back**

- [ ] Bot responding in Google Chat via Python webhook

### Chosen Option: `___________` (A / B)

### Configuration Details

| Setting         | Value          |
|-----------------|----------------|
| App Name        | _(fill in)_    |
| Connection Type | _(Direct Dialogflow / HTTP Webhook)_ |
| Webhook URL     | _(if Option B)_ |
| Visibility      | _(fill in)_    |

### Notes

_(Add notes here as you complete this phase)_

---

## 9. Phase 5 — Voice Support

### Option A — Phone Gateway (Recommended for phone calls)

- [ ] In Conversational Agents Console → Sidebar → **Integrations**
- [ ] Click **Phone Gateway**
- [ ] Provision a phone number
- [ ] Test by calling the number

### Option B — Web Widget with Mic (Recommended for browser)

- [ ] In Conversational Agents Console → Sidebar → **Integrations**
- [ ] Click **Dialogflow Messenger**
- [ ] Enable microphone button in widget config
- [ ] Copy the embed `<script>` tag
- [ ] Embed widget on website / internal portal

### Option C — Python + Speech-to-Text (Programmatic Voice)

> If you want voice in your own app (not a phone line or web widget), you can use Speech-to-Text API to convert audio to text, then send to Dialogflow CX.

```python
"""
Voice input → Speech-to-Text → Dialogflow CX → Response
"""

from google.cloud import speech_v1 as speech

def transcribe_audio(audio_file_path: str) -> str:
    """Convert audio file to text using Google Speech-to-Text."""
    
    client = speech.SpeechClient()

    with open(audio_file_path, "rb") as audio_file:
        content = audio_file.read()

    audio = speech.RecognitionAudio(content=content)
    config = speech.RecognitionConfig(
        encoding=speech.RecognitionConfig.AudioEncoding.LINEAR16,
        sample_rate_hertz=16000,
        language_code="en-US",
    )

    response = client.recognize(config=config, audio=audio)

    transcript = ""
    for result in response.results:
        transcript += result.alternatives[0].transcript

    return transcript


# Usage: transcribe → send to Dialogflow
# audio_text = transcribe_audio("user_audio.wav")
# bot_response = get_dialogflow_response(audio_text, session_id)  # reuse function from Phase 4
```

### Chosen Approach: `___________`

### Notes

_(Add notes here as you complete this phase)_

---

## 10. Phase 6 — Testing & Deployment

### Testing Checklist

- [ ] Test in Conversational Agents Console built-in simulator
- [ ] Test all intents with sample inputs
- [ ] Test in Google Chat (direct message to bot)
- [ ] Test voice input (phone gateway / web widget)
- [ ] Test edge cases and fallback responses
- [ ] Verify Cloud Logging for errors
- [ ] Load test (if applicable)

### Go-Live Checklist

- [ ] All intents working correctly
- [ ] All API integrations tested
- [ ] Error handling in place
- [ ] Bot visibility configured for target users
- [ ] Monitoring and alerting set up

### Notes

_(Add notes here as you complete this phase)_

---

## 11. API List & Intent/Tool Mapping

| # | API Name | Base URL | Mapped To (Intent / Playbook Tool) | Auth Method | Status |
|---|----------|----------|-------------------------------------|-------------|--------|
| 1 | _(add)_  |          |                                     |             |        |
| 2 | _(add)_  |          |                                     |             |        |
| 3 | _(add)_  |          |                                     |             |        |

---

## 12. Issues & Troubleshooting Log

| Date | Issue | Root Cause | Resolution | Status |
|------|-------|------------|------------|--------|
| _(add as encountered)_ | | | | |

---

## 13. Updates Log

| Date           | Update Description                              |
|----------------|--------------------------------------------------|
| April 30, 2026 | Created initial development plan and notes file  |
| April 30, 2026 | Added conversational context/memory requirement — bot must remember previous turns |
| April 30, 2026 | **Major update:** Corrected all naming to current Google branding — Gemini Enterprise Agent Platform, Conversational Agents Console, Playbooks vs Flows, Agent Platform Sessions & Memory Bank, ADK, Agent Runtime, etc. Added rebranding guide section. |
| April 30, 2026 | Added detailed step-by-step guide for Phase 1: Create project, enable each API (with direct links), set up billing, create service account, verify console access |
| April 30, 2026 | Phase 2: Added agent creation steps — select "Build your own" (not prebuilt, not Q&A). Added form fields and warnings about region/language lock-in. |
| April 30, 2026 | Agent created with Playbooks approach. Added sidebar reference table, next steps order (Playbook → Tools → Test → Integrate), and detailed Steps 3-5 for Playbook setup. |
| April 30, 2026 | Rewrote Phase 3 (testing), Phase 4 (Google Chat integration with 2 options: Direct Dialogflow vs Python webhook), updated Phase 5 (voice with Python STT code). All code now in Python. |
| April 30, 2026 | Fixed Phase 4 Step 8A: clarified that Configuration tab is under Enabled APIs detail page, NOT the API Library page. Added direct URL. |

---
