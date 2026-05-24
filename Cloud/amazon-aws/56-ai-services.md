# Chapter 56: AWS AI/ML Services

---

## Table of Contents

- [Overview](#overview)
- [Part 1: AI/ML Services Landscape](#part-1-aiml-services-landscape)
- [Part 2: Amazon Bedrock (Generative AI)](#part-2-amazon-bedrock-generative-ai)
- [Part 3: Amazon Rekognition (Vision)](#part-3-amazon-rekognition-vision)
- [Part 4: Amazon Comprehend, Textract, Translate & Polly](#part-4-amazon-comprehend-textract-translate--polly)
- [Part 5: Terraform & CLI Examples](#part-5-terraform--cli-examples)
- [Part 6: Real-World Patterns](#part-6-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### When Would I Use AI Services vs SageMaker?

**Pre-built AI services** are like ordering food at a restaurant — someone else has already built the recipe (model), you just use it. **SageMaker** is like cooking from scratch — you build your own recipe.

**Use pre-built AI services when:**
- You want to add image recognition, text analysis, translation, or chatbot capabilities to your app
- You don't have ML expertise on your team
- The standard capabilities are good enough (they usually are!)

**Use SageMaker when:**
- You need a model trained on YOUR specific data
- Pre-built services don't cover your use case
- You need fine-grained control over the model

AWS provides pre-built AI/ML services that allow you to add intelligence to applications without building or training custom models. These services cover vision, language, speech, and generative AI.

```
What you'll learn:
├── AI/ML services landscape
├── Amazon Bedrock (generative AI, foundation models)
├── Amazon Rekognition (image/video analysis)
├── Amazon Comprehend (NLP), Textract (document processing)
├── Amazon Translate, Polly (language, speech)
├── Other AI services (Transcribe, Lex, Personalize, Forecast)
└── Real-world patterns
```

---

## Part 1: AI/ML Services Landscape

```
┌─────────────────────────────────────────────────────────────────────┐
│           AWS AI/ML SERVICES                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Generative AI:                                                       │
│ ├── Bedrock: Foundation models (Claude, Llama, Titan, etc.)     │
│ ├── CodeWhisperer: AI code generation                            │
│ └── Q: AI assistant for business/developers                     │
│                                                                       │
│ Vision:                                                               │
│ ├── Rekognition: Image/video analysis (faces, objects, text)   │
│ └── Lookout for Vision: Industrial defect detection             │
│                                                                       │
│ Language:                                                             │
│ ├── Comprehend: NLP (sentiment, entities, topics, PII)         │
│ ├── Translate: Language translation (75+ languages)             │
│ ├── Textract: Document text extraction (forms, tables)         │
│ └── Kendra: Intelligent enterprise search                       │
│                                                                       │
│ Speech:                                                               │
│ ├── Polly: Text-to-speech (lifelike voices)                    │
│ ├── Transcribe: Speech-to-text (ASR)                           │
│ └── Lex: Conversational bots (chatbots)                        │
│                                                                       │
│ Business Intelligence:                                               │
│ ├── Personalize: Recommendation engine                          │
│ ├── Forecast: Time-series forecasting                           │
│ └── Fraud Detector: Fraud detection                             │
│                                                                       │
│ All services: Pay-per-use, no ML expertise required               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Amazon Bedrock (Generative AI)

```
Console → Amazon Bedrock → Model access → Enable models

┌─────────────────────────────────────────────────────────────────┐
│           AMAZON BEDROCK                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: Fully managed service to access foundation models       │
│                                                                   │
│ Available models:                                              │
│ ├── Anthropic Claude 3.5 (text, vision)                      │
│ ├── Meta Llama 3 (text generation)                           │
│ ├── Amazon Titan (text, embeddings, image)                   │
│ ├── Stability AI SDXL (image generation)                    │
│ ├── Cohere (text, embeddings)                                │
│ └── Mistral AI (text generation)                             │
│                                                                   │
│ Key features:                                                  │
│ ├── Serverless: No infrastructure to manage                 │
│ ├── Model access: Request access per model                  │
│ ├── Playground: Test models in console                      │
│ ├── Knowledge Bases: RAG (Retrieval-Augmented Generation)  │
│ ├── Agents: Build AI agents with tool use                   │
│ ├── Guardrails: Content filtering, PII redaction           │
│ ├── Fine-tuning: Customize models with your data           │
│ └── Model evaluation: Compare models                        │
│                                                                   │
│ Playground:                                                    │
│ Console → Bedrock → Playgrounds → Text                       │
│ Model: [Anthropic Claude 3.5 Sonnet ▼]                      │
│ Prompt: [Summarize this customer review: ...]               │
│ Parameters:                                                    │
│ ├── Temperature: [0.7] (0-1, creativity)                   │
│ ├── Top P: [0.9]                                            │
│ ├── Max tokens: [512]                                       │
│ └── Stop sequences: []                                      │
│                                                                   │
│ Knowledge Bases (RAG):                                        │
│ Console → Bedrock → Knowledge bases → Create                 │
│ ├── Data source: S3 bucket with documents                  │
│ ├── Embedding model: Amazon Titan Embeddings               │
│ ├── Vector database: OpenSearch Serverless (auto-created) │
│ └── Query: Ask questions against your documents            │
│                                                                   │
│ Pricing (per 1000 tokens):                                    │
│ ├── Claude 3.5 Sonnet: $3/M input, $15/M output           │
│ ├── Llama 3 8B: $0.30/M input, $0.60/M output            │
│ ├── Titan Text: $0.30/M input, $0.40/M output             │
│ └── ⚡ ~750 words ≈ 1000 tokens                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Amazon Rekognition (Vision)

```
Console → Amazon Rekognition → Try demo

┌─────────────────────────────────────────────────────────────────┐
│           AMAZON REKOGNITION                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Image Analysis:                                                │
│ ├── Object & scene detection (car, tree, beach, etc.)       │
│ ├── Facial analysis (age, emotion, glasses, gender)        │
│ ├── Face comparison (similarity matching)                   │
│ ├── Face search (match against collection)                 │
│ ├── Celebrity recognition                                   │
│ ├── Text detection (in images)                              │
│ ├── Content moderation (unsafe content)                     │
│ └── Custom labels (train on your images)                   │
│                                                                   │
│ Video Analysis:                                                │
│ ├── Same features as image (applied to video frames)      │
│ ├── Person tracking                                         │
│ ├── Activity detection                                      │
│ └── Async processing (results via SNS notification)       │
│                                                                   │
│ API example:                                                   │
│ aws rekognition detect-labels \                              │
│   --image '{"S3Object":{"Bucket":"photos","Name":"img.jpg"}}'│
│                                                                   │
│ Response:                                                      │
│ Labels:                                                        │
│ ├── {Name: "Dog", Confidence: 99.2%}                       │
│ ├── {Name: "Animal", Confidence: 99.2%}                    │
│ ├── {Name: "Park", Confidence: 95.1%}                      │
│ └── {Name: "Grass", Confidence: 92.8%}                     │
│                                                                   │
│ Pricing:                                                       │
│ ├── Image: $1.00/1000 images (first 1M/month)            │
│ ├── Video: $0.10/minute                                    │
│ └── Face search: $0.01/1000 faces stored/month            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Amazon Comprehend, Textract, Translate & Polly

```
┌─────────────────────────────────────────────────────────────────────┐
│           COMPREHEND (NLP)                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Features:                                                            │
│ ├── Sentiment analysis: POSITIVE, NEGATIVE, NEUTRAL, MIXED     │
│ ├── Entity recognition: People, places, organizations, dates   │
│ ├── Key phrases extraction                                       │
│ ├── Language detection (100+ languages)                         │
│ ├── PII detection and redaction                                  │
│ ├── Topic modeling (group documents by topic)                  │
│ └── Custom classification/entity (train on your data)         │
│                                                                       │
│ aws comprehend detect-sentiment --text "I love this product!"  │
│ → {Sentiment: POSITIVE, Scores: {Positive: 0.99, ...}}       │
│ Pricing: $0.0001/unit (1 unit = 100 characters)                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TEXTRACT (Document Processing)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Features:                                                            │
│ ├── Text detection: Extract text from scanned documents        │
│ ├── Form extraction: Key-value pairs from forms                │
│ ├── Table extraction: Extract structured table data            │
│ ├── Invoice/receipt processing (AnalyzeExpense)               │
│ ├── ID document processing (AnalyzeID)                         │
│ └── Custom queries (ask questions about document)              │
│                                                                       │
│ aws textract analyze-document \                                     │
│   --document '{"S3Object":{"Bucket":"docs","Name":"form.pdf"}}' \│
│   --feature-types '["FORMS","TABLES"]'                             │
│ Pricing: $1.50/1000 pages (forms), $15/1000 pages (queries)    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│           TRANSLATE & POLLY                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Amazon Translate:                                                    │
│ ├── Neural machine translation (75+ languages)                  │
│ ├── Real-time and batch translation                              │
│ ├── Custom terminology (industry-specific terms)               │
│ └── Pricing: $15/million characters                             │
│                                                                       │
│ Amazon Polly (Text-to-Speech):                                     │
│ ├── Lifelike speech in 30+ languages                           │
│ ├── Standard voices + Neural voices (higher quality)          │
│ ├── SSML support (control pronunciation, pauses, emphasis)    │
│ ├── Speech marks (word timing for lip sync)                   │
│ └── Pricing: $4/million characters (standard), $16 (neural)  │
│                                                                       │
│ Amazon Transcribe (Speech-to-Text):                                │
│ ├── Automatic speech recognition (100+ languages)             │
│ ├── Real-time and batch transcription                          │
│ ├── Custom vocabulary (industry terms)                         │
│ ├── Speaker identification                                      │
│ ├── Content redaction (PII)                                     │
│ ├── Medical transcription (Amazon Transcribe Medical)         │
│ └── Pricing: $0.024/minute                                     │
│                                                                       │
│ Amazon Lex (Chatbots):                                              │
│ ├── Build conversational interfaces (same tech as Alexa)     │
│ ├── Intents, slots, fulfillment (Lambda)                      │
│ ├── Voice and text chat                                         │
│ ├── Integration: Connect, Slack, Facebook Messenger           │
│ └── Pricing: $0.004/voice request, $0.00075/text request     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Terraform & CLI Examples

```hcl
# Bedrock - invoke model (via AWS SDK, not Terraform)
# Python example:
# import boto3, json
# client = boto3.client('bedrock-runtime')
# response = client.invoke_model(
#     modelId='anthropic.claude-3-sonnet-20240229-v1:0',
#     body=json.dumps({
#         "anthropic_version": "bedrock-2023-05-31",
#         "max_tokens": 512,
#         "messages": [{"role": "user", "content": "Explain AWS Lambda"}]
#     })
# )

# Rekognition - create face collection
resource "aws_rekognition_collection" "employees" {
  collection_id = "employee-faces"
}
```

```bash
# Rekognition - detect labels
aws rekognition detect-labels \
  --image '{"S3Object":{"Bucket":"photos","Name":"image.jpg"}}' \
  --max-labels 10

# Comprehend - detect sentiment
aws comprehend detect-sentiment \
  --text "This service is amazing!" \
  --language-code en

# Comprehend - detect PII
aws comprehend detect-pii-entities \
  --text "My name is John Smith, SSN 123-45-6789" \
  --language-code en

# Textract - analyze document
aws textract analyze-document \
  --document '{"S3Object":{"Bucket":"docs","Name":"invoice.pdf"}}' \
  --feature-types '["FORMS","TABLES"]'

# Translate
aws translate translate-text \
  --text "Hello, how are you?" \
  --source-language-code en \
  --target-language-code es

# Polly - synthesize speech
aws polly synthesize-speech \
  --text "Welcome to AWS" \
  --output-format mp3 \
  --voice-id Joanna \
  output.mp3

# Bedrock - invoke model
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-3-sonnet-20240229-v1:0 \
  --body '{"anthropic_version":"bedrock-2023-05-31","max_tokens":256,"messages":[{"role":"user","content":"What is DynamoDB?"}]}' \
  --content-type application/json \
  output.json
```

---

## Part 6: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Document Processing Pipeline                             │
│ S3 (upload PDF) → Lambda → Textract (extract text/forms)         │
│                          → Comprehend (extract entities, PII)    │
│                          → DynamoDB (store structured data)      │
│ → Automate invoice processing, contract analysis                │
│                                                                       │
│ Pattern 2: Content Moderation                                       │
│ S3 (user-uploaded images) → Rekognition (detect unsafe content)  │
│ → Lambda (approve/reject)                                         │
│ → SNS (notify moderators if borderline)                          │
│                                                                       │
│ Pattern 3: RAG-based Chatbot (Bedrock)                             │
│ User Question → Bedrock Knowledge Base (RAG)                      │
│ → Vector search in company docs                                   │
│ → Foundation model generates answer with context                 │
│ → Guardrails filter response                                      │
│                                                                       │
│ Pattern 4: Multi-Language Customer Support                          │
│ Customer message → Comprehend (detect language + sentiment)      │
│ → Translate (to agent's language)                                 │
│ → Agent responds → Translate (back to customer's language)      │
│ → Polly (optional: voice response)                                │
│                                                                       │
│ Pattern 5: Sentiment Analysis Dashboard                             │
│ Reviews/Tweets → Kinesis → Lambda → Comprehend (sentiment)      │
│ → DynamoDB (store results) → QuickSight (dashboard)            │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Use pre-built AI services before building custom models     │
│ 2. Bedrock Knowledge Bases for company-specific Q&A            │
│ 3. Enable Guardrails for production GenAI applications        │
│ 4. Use async APIs for video/large document processing         │
│ 5. Comprehend PII detection for data privacy compliance       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
AI/ML Services Quick Reference:
├── Bedrock: Generative AI (Claude, Llama, Titan, SDXL)
│   ├── Knowledge Bases: RAG on your documents
│   ├── Agents: AI agents with tool use
│   └── Guardrails: Content filtering + PII redaction
├── Rekognition: Image/video analysis (faces, objects, moderation)
├── Comprehend: NLP (sentiment, entities, PII, topics)
├── Textract: Document processing (text, forms, tables, invoices)
├── Translate: Language translation (75+ languages)
├── Polly: Text-to-speech (30+ languages, neural voices)
├── Transcribe: Speech-to-text (100+ languages)
├── Lex: Chatbots (voice + text, Alexa technology)
├── Personalize: Recommendation engine
├── Forecast: Time-series forecasting
├── Fraud Detector: ML-based fraud detection
├── ⚡ All services: Pay-per-use, no ML expertise needed
├── ⚡ Use pre-built services before custom SageMaker models
└── ⚡ Bedrock for GenAI, Rekognition for vision, Comprehend for NLP
```

---

## What's Next?

In **Chapter 57: Migration Strategies**, we'll cover the 7 R's of migration, AWS Migration Hub, Database Migration Service (DMS), Application Migration Service, and migration planning.
