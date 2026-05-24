# Chapter 61: Azure AI Services

---

## Table of Contents

- [Overview](#overview)
- [Part 1: AI Services Overview](#part-1-ai-services-overview)
- [Part 2: Azure OpenAI Service](#part-2-azure-openai-service)
- [Part 3: Vision Services](#part-3-vision-services)
- [Part 4: Speech Services](#part-4-speech-services)
- [Part 5: Language Services](#part-5-language-services)
- [Part 6: Azure AI Search](#part-6-azure-ai-search)
- [Part 7: Azure Bot Service](#part-7-azure-bot-service)
- [Part 8: Terraform & az CLI Reference](#part-8-terraform--az-cli-reference)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Azure AI Services are pre-built AI APIs that add intelligence to your apps without building ML models. Just call an API — no data science expertise needed. Includes Azure OpenAI (GPT-4, DALL-E), Vision, Speech, Language, and more.

```
What you'll learn:
├── AI Services Overview (pre-built AI vs custom ML)
├── Azure OpenAI Service (GPT-4, ChatGPT, DALL-E, Embeddings)
├── Vision (image analysis, OCR, face detection)
├── Speech (speech-to-text, text-to-speech, translation)
├── Language (sentiment, key phrases, translation, QnA)
├── Azure AI Search (intelligent search)
├── Bot Service (chatbots)
├── Terraform, az CLI
└── Quick reference
```

---

## Part 1: AI Services Overview

```
Pre-built AI vs Custom ML:
├── Pre-built (AI Services): Call an API, instant results
│   "Analyze this image" → "It's a cat on a sofa"
│   No training, no data needed
│
└── Custom (Azure ML): Train your own model
    Need your own data, ML expertise
    Better for domain-specific problems

AI Services categories:
┌──────────────────────────────────────────────────────┐
│ Azure AI Services                                      │
├──────────────┬───────────────────────────────────────┤
│ OpenAI       │ GPT-4, GPT-4o, ChatGPT, DALL-E,     │
│              │ Embeddings, Whisper                    │
├──────────────┼───────────────────────────────────────┤
│ Vision       │ Image analysis, OCR, Face, Custom     │
│              │ Vision, Video Indexer                  │
├──────────────┼───────────────────────────────────────┤
│ Speech       │ Speech-to-Text, Text-to-Speech,      │
│              │ Translation, Speaker Recognition      │
├──────────────┼───────────────────────────────────────┤
│ Language     │ Sentiment, Key Phrases, NER, QnA,    │
│              │ Translation, Summarization            │
├──────────────┼───────────────────────────────────────┤
│ Decision     │ Content Safety, Anomaly Detector,    │
│              │ Personalizer                          │
├──────────────┼───────────────────────────────────────┤
│ Search       │ Azure AI Search (cognitive search)    │
└──────────────┴───────────────────────────────────────┘

Deployment options:
├── Multi-service resource: One resource, all APIs
├── Single-service resource: One resource, one API
└── Best practice: Separate resource for OpenAI (different pricing)
```

---

## Part 2: Azure OpenAI Service

```
Console → Azure OpenAI → Create

┌─────────────────────────────────────────────────────────────────────┐
│           AZURE OPENAI SERVICE                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Name: [oai-mycompany]                                               │
│ Region: [East US ▼] (not all regions available)                    │
│ Pricing tier: [Standard S0]                                        │
│                                                                       │
│ After creation → Go to Azure OpenAI Studio                        │
│ URL: https://oai.azure.com                                         │
│                                                                       │
│ Deploy a model:                                                      │
│ Deployments → [+ Create new deployment]                            │
│ Model: [gpt-4o ▼]                                                 │
│ Deployment name: [gpt-4o-deployment]                               │
│ Tokens per minute: [10000]                                         │
│                                                                       │
│ Available models:                                                    │
│ ├── GPT-4o: Latest, multimodal (text + images)                   │
│ ├── GPT-4 Turbo: Powerful reasoning                               │
│ ├── GPT-3.5 Turbo: Fast, cheap                                   │
│ ├── DALL-E 3: Image generation                                    │
│ ├── Whisper: Speech-to-text                                       │
│ └── text-embedding-ada-002: Vector embeddings                    │
│                                                                       │
│ Call the API:                                                        │
│ POST https://oai-mycompany.openai.azure.com/openai/deployments/   │
│      gpt-4o-deployment/chat/completions?api-version=2024-02-15    │
│                                                                       │
│ {                                                                     │
│   "messages": [                                                      │
│     {"role": "system", "content": "You are a helpful assistant"},  │
│     {"role": "user", "content": "What is Azure?"}                  │
│   ],                                                                 │
│   "max_tokens": 800                                                 │
│ }                                                                     │
│                                                                       │
│ Content filtering: Built-in safety (hate, violence, self-harm)    │
│ RAG pattern: Combine with AI Search for your own data             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

Pricing (pay per token):
├── GPT-4o: ~$5/1M input tokens, ~$15/1M output tokens
├── GPT-3.5 Turbo: ~$0.50/1M input, ~$1.50/1M output
├── DALL-E 3: ~$0.04/image (Standard)
└── Embeddings: ~$0.10/1M tokens
```

---

## Part 3: Vision Services

```
Console → AI Services → Computer Vision → Create

Image Analysis:
  POST /computervision/imageanalysis:analyze?features=caption,tags,objects
  Body: {"url": "https://example.com/photo.jpg"}
  Response: {
    "caption": "a person riding a bicycle on a city street",
    "tags": ["bicycle", "person", "street", "outdoor"],
    "objects": [{"name": "bicycle", "boundingBox": {...}}]
  }

OCR (Read text from images):
  POST /computervision/imageanalysis:analyze?features=read
  ├── Extracts printed and handwritten text
  ├── Supports 164 languages
  └── Great for: receipts, documents, signs, forms

Face API:
  ├── Face detection (age, emotion, glasses, head pose)
  ├── Face verification (is this the same person?)
  ├── Face grouping (cluster similar faces)
  └── ⚠️ Requires approval for identification features

Custom Vision:
  ├── Train your own image classifier or object detector
  ├── Upload labeled images → Train → Deploy
  ├── Example: "Is this product defective?" (quality inspection)
  └── Portal: https://customvision.ai
```

---

## Part 4: Speech Services

```
Console → AI Services → Speech → Create

Speech-to-Text:
  ├── Real-time transcription (streaming audio → text)
  ├── Batch transcription (audio files → text)
  ├── Custom Speech: Train on your vocabulary
  └── 100+ languages supported

  const speechConfig = SpeechConfig.fromSubscription(key, region);
  const recognizer = new SpeechRecognizer(speechConfig);
  recognizer.recognizeOnceAsync(result => {
    console.log("You said:", result.text);
  });

Text-to-Speech:
  ├── 400+ neural voices, 140+ languages
  ├── Custom Neural Voice: Clone a voice (with consent)
  ├── SSML for fine control (speed, pitch, emphasis)
  └── Real-time streaming or file output

  const synthesizer = new SpeechSynthesizer(speechConfig);
  synthesizer.speakTextAsync("Hello, welcome to Azure!");

Speech Translation:
  ├── Real-time speech-to-speech translation
  └── Speak English → Hear Hindi, Spanish, etc.

Pricing:
├── Speech-to-Text: $1/audio hour (standard)
├── Text-to-Speech: $16/1M characters (neural)
└── Free tier: 5 hours STT, 0.5M chars TTS per month
```

---

## Part 5: Language Services

```
Console → AI Services → Language → Create

Sentiment Analysis:
  Input: "The food was amazing but the service was terrible"
  Output: { positive: 0.1, neutral: 0.1, negative: 0.8 }
  Per-sentence: "food was amazing" (positive), "service was terrible" (negative)

Key Phrase Extraction:
  Input: "Azure Machine Learning provides tools for training models"
  Output: ["Azure Machine Learning", "tools", "training models"]

Named Entity Recognition (NER):
  Input: "Microsoft was founded in Redmond in 1975"
  Output: [
    {"text": "Microsoft", "category": "Organization"},
    {"text": "Redmond", "category": "Location"},
    {"text": "1975", "category": "DateTime"}
  ]

PII Detection:
  Detects and redacts personal information
  "Call me at 9876543210" → "Call me at ██████████"

Question Answering (formerly QnA Maker):
  ├── Upload documents/FAQs → Creates knowledge base
  ├── Users ask questions → Returns best answer
  └── Great for: FAQ bots, help desk automation

Text Translation (Translator):
  ├── 100+ languages, real-time translation
  ├── Custom Translator: Train on your domain terminology
  └── Document translation: Translate entire documents

Text Summarization:
  ├── Extractive: Picks key sentences from text
  └── Abstractive: Generates new summary (uses GPT)
```

---

## Part 6: Azure AI Search

```
Console → AI Search → Create

Azure AI Search = Intelligent search engine for your data

Name: [srch-mycompany]
Pricing tier: [Basic] (~$75/month)

How it works:
  Data sources (Blob, SQL, Cosmos)
       ↓
  Indexer (crawls and extracts)
       ↓ AI enrichment (optional):
       ├── OCR (extract text from images)
       ├── Key phrases, entities, sentiment
       ├── Custom skills (Azure Function)
       └── Vector embeddings (for semantic search)
       ↓
  Search Index (fast, queryable)
       ↓
  Search API
  GET /indexes/products/docs?search=laptop&$filter=price lt 50000

RAG (Retrieval-Augmented Generation):
├── Your documents → AI Search index
├── User asks question → Search finds relevant docs
├── Relevant docs + question → Send to GPT-4
├── GPT-4 answers using YOUR data (not hallucinating)
└── "On Your Data" feature in Azure OpenAI Studio
```

---

## Part 7: Azure Bot Service

```
Console → Bot Services → Create → Azure Bot

Bot Service = Build and deploy chatbots

Frameworks:
├── Bot Framework SDK (C#, JavaScript, Python)
├── Power Virtual Agents (no-code, now Copilot Studio)
└── Bot Framework Composer (visual, low-code)

Channels (deploy bot to):
├── Microsoft Teams
├── Web Chat (embed in website)
├── Slack, Facebook Messenger
├── Twilio (SMS)
├── Direct Line (custom apps)
└── Email

Architecture:
  User (Teams/Web) → Bot Service → Your Bot Code (App Service)
                                         │
                                    ├── Language Understanding
                                    ├── QnA Knowledge Base
                                    └── Azure OpenAI (GPT-4)
```

---

## Part 8: Terraform & az CLI Reference

### Terraform

```hcl
resource "azurerm_cognitive_account" "openai" {
  name                = "oai-mycompany"
  resource_group_name = azurerm_resource_group.main.name
  location            = "eastus"
  kind                = "OpenAI"
  sku_name            = "S0"
}

resource "azurerm_cognitive_deployment" "gpt4o" {
  name                 = "gpt-4o-deployment"
  cognitive_account_id = azurerm_cognitive_account.openai.id
  model {
    format  = "OpenAI"
    name    = "gpt-4o"
    version = "2024-05-13"
  }
  sku {
    name     = "Standard"
    capacity = 10
  }
}

resource "azurerm_search_service" "main" {
  name                = "srch-mycompany"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "basic"
}
```

### Bicep

```bicep
// Azure OpenAI Service
resource openai 'Microsoft.CognitiveServices/accounts@2023-10-01-preview' = {
  name: 'oai-mycompany-prod'
  location: resourceGroup().location
  kind: 'OpenAI'
  sku: { name: 'S0' }
  properties: {
    customSubDomainName: 'oai-mycompany-prod'
    publicNetworkAccess: 'Enabled'
  }
}

// GPT-4 Deployment
resource gpt4 'Microsoft.CognitiveServices/accounts/deployments@2023-10-01-preview' = {
  parent: openai
  name: 'gpt-4'
  sku: { name: 'Standard', capacity: 10 }
  properties: {
    model: { format: 'OpenAI', name: 'gpt-4', version: '0613' }
  }
}

// Azure AI Search
resource search 'Microsoft.Search/searchServices@2023-11-01' = {
  name: 'srch-mycompany'
  location: resourceGroup().location
  sku: { name: 'basic' }
  properties: { replicaCount: 1, partitionCount: 1 }
}
```
az cognitiveservices account create \
  --name oai-mycompany \
  --resource-group rg-ai \
  --kind OpenAI --sku S0 \
  --location eastus

# Deploy GPT-4o model
az cognitiveservices account deployment create \
  --name oai-mycompany \
  --resource-group rg-ai \
  --deployment-name gpt-4o-deployment \
  --model-name gpt-4o --model-version 2024-05-13 \
  --model-format OpenAI \
  --sku-name Standard --sku-capacity 10

# Create AI Search
az search service create \
  --name srch-mycompany \
  --resource-group rg-ai \
  --sku basic

# Get OpenAI key
az cognitiveservices account keys list \
  --name oai-mycompany --resource-group rg-ai

# Delete
az cognitiveservices account delete --name oai-mycompany --resource-group rg-ai
```

---

## Real-World Patterns

### Pattern 1: RAG-Based Enterprise Chatbot

```
┌─────────────────────────────────────────────────┐
│  RAG (Retrieval-Augmented Generation)           │
├─────────────────────────────────────────────────┤
│                                                 │
│  Company Docs ─→ AI Search (index + embed)      │
│  (PDFs, wikis)                                  │
│                                                 │
│  User Question                                  │
│       │                                         │
│       ▼                                         │
│  AI Search: Find relevant docs (vector search) │
│       │                                         │
│       ▼                                         │
│  Azure OpenAI (GPT-4):                          │
│  "Answer based on these docs: [context]"        │
│       │                                         │
│       ▼                                         │
│  Grounded Answer with Citations                 │
│                                                 │
│  Benefits: No hallucination (grounded in docs), │
│  always up-to-date, company-specific knowledge  │
└─────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Azure AI Services = Pre-built AI APIs (no ML expertise needed)

OpenAI: GPT-4o, GPT-3.5, DALL-E 3, Embeddings, Whisper
Vision: Image analysis, OCR, Face, Custom Vision
Speech: Speech-to-Text, Text-to-Speech, Translation (400+ voices)
Language: Sentiment, Key Phrases, NER, PII, QnA, Translation, Summarization
Search: Azure AI Search (intelligent search + RAG with OpenAI)
Bot: Bot Framework + Channels (Teams, Web, Slack)

RAG Pattern: Your docs → AI Search → GPT-4 answers with your data

Pricing: Pay per API call / token / character
Free tiers available for most services

Azure OpenAI Studio: https://oai.azure.com
Custom Vision: https://customvision.ai
```

---

## What's Next?

Next chapter: [Chapter 62: Azure Migrate](62-azure-migrate.md) — Tools and strategies for migrating workloads from on-premises to Azure.
