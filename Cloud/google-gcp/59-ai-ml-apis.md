# Chapter 59 — AI/ML APIs

---

## Table of Contents

- [Overview](#overview)
- [Part 1: AI/ML APIs Fundamentals](#part-1--aiml-apis-fundamentals)
- [Part 2: Cloud Vision API](#part-2--cloud-vision-api)
- [Part 3: Cloud Natural Language API](#part-3--cloud-natural-language-api)
- [Part 4: Cloud Translation API](#part-4--cloud-translation-api)
- [Part 5: Cloud Speech-to-Text](#part-5--cloud-speech-to-text)
- [Part 6: Cloud Text-to-Speech](#part-6--cloud-text-to-speech)
- [Part 7: Document AI](#part-7--document-ai)
- [Part 8: Video Intelligence API](#part-8--video-intelligence-api)
- [Part 9: Recommendations AI](#part-9--recommendations-ai)
- [Part 10: Dialogflow CX](#part-10--dialogflow-cx)
- [Part 11: Gemini API (Generative AI)](#part-11--gemini-api-generative-ai)
- [Part 12: Imagen API (Image Generation)](#part-12--imagen-api-image-generation)
- [Part 13: Embedding APIs](#part-13--embedding-apis)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Patterns](#part-15--real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

Google Cloud provides a rich set of pre-trained AI/ML APIs that let you add intelligence to applications without building custom models. From computer vision and natural language processing to speech recognition and document understanding, these APIs are accessible via REST/gRPC calls and client libraries. Combined with Gemini's generative AI capabilities, they cover virtually every AI use case — from extracting text from invoices to generating images and building conversational agents.

---

## Part 1 — AI/ML APIs Fundamentals

### API Landscape

```
┌────────────────────────────────────────────────────────────────────┐
│         GOOGLE CLOUD AI/ML APIs                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  VISION & IMAGE                                                     │
│  ┌──────────────────┐  ┌──────────────────┐                       │
│  │ Cloud Vision API │  │ Imagen           │                       │
│  │ (analyze images) │  │ (generate images)│                       │
│  └──────────────────┘  └──────────────────┘                       │
│                                                                      │
│  LANGUAGE & TEXT                                                    │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │ Natural Language │  │ Translation      │  │ Gemini        │  │
│  │ (NLP analysis)   │  │ (130+ languages) │  │ (generative)  │  │
│  └──────────────────┘  └──────────────────┘  └───────────────┘  │
│                                                                      │
│  SPEECH & AUDIO                                                     │
│  ┌──────────────────┐  ┌──────────────────┐                       │
│  │ Speech-to-Text   │  │ Text-to-Speech   │                       │
│  │ (transcription)  │  │ (voice synthesis)│                       │
│  └──────────────────┘  └──────────────────┘                       │
│                                                                      │
│  DOCUMENTS & STRUCTURED DATA                                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │ Document AI      │  │ Recommendations  │  │ Dialogflow CX │  │
│  │ (OCR, extraction)│  │ AI               │  │ (chatbots)    │  │
│  └──────────────────┘  └──────────────────┘  └───────────────┘  │
│                                                                      │
│  VIDEO                                                              │
│  ┌──────────────────┐                                              │
│  │ Video Intelligence│                                             │
│  │ (label, track)    │                                             │
│  └──────────────────┘                                              │
│                                                                      │
│  All APIs: REST/gRPC, client libraries (Python/Java/Go/Node)     │
│  No ML expertise required — just send data, get predictions       │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Category | GCP | AWS | Azure |
|----------|-----|-----|-------|
| Vision | Cloud Vision API | Rekognition | Computer Vision |
| NLP | Natural Language API | Comprehend | Text Analytics |
| Translation | Cloud Translation | Translate | Translator |
| Speech-to-Text | Cloud Speech-to-Text | Transcribe | Speech Service |
| Text-to-Speech | Cloud Text-to-Speech | Polly | Speech Service |
| Document processing | Document AI | Textract | Form Recognizer |
| Video analysis | Video Intelligence | Rekognition Video | Video Indexer |
| Chatbot | Dialogflow CX | Lex | Bot Service |
| Recommendations | Recommendations AI | Personalize | Personalizer |
| Generative AI | Gemini | Bedrock (Claude) | Azure OpenAI (GPT) |

---

## Part 2 — Cloud Vision API

### Image Analysis

```python
from google.cloud import vision

client = vision.ImageAnnotatorClient()

# Analyze image from GCS
image = vision.Image(
    source=vision.ImageSource(gcs_image_uri='gs://my-bucket/photo.jpg')
)

# Label detection — what's in the image?
response = client.label_detection(image=image)
for label in response.label_annotations:
    print(f"{label.description}: {label.score:.2f}")
# Output: "Dog: 0.98", "Animal: 0.97", "Outdoors: 0.92"

# Text detection (OCR)
response = client.text_detection(image=image)
for text in response.text_annotations:
    print(text.description)

# Face detection
response = client.face_detection(image=image)
for face in response.face_annotations:
    print(f"Joy: {face.joy_likelihood.name}")
    print(f"Anger: {face.anger_likelihood.name}")
    print(f"Confidence: {face.detection_confidence:.2f}")

# Object localization
response = client.object_localization(image=image)
for obj in response.localized_object_annotations:
    print(f"{obj.name}: {obj.score:.2f}")
    for vertex in obj.bounding_poly.normalized_vertices:
        print(f"  ({vertex.x:.2f}, {vertex.y:.2f})")

# Safe search detection (content moderation)
response = client.safe_search_detection(image=image)
safe = response.safe_search_annotation
print(f"Adult: {safe.adult.name}")
print(f"Violence: {safe.violence.name}")

# Logo detection
response = client.logo_detection(image=image)

# Landmark detection
response = client.landmark_detection(image=image)

# Web detection (find similar images online)
response = client.web_detection(image=image)
```

### Vision API Features Summary

| Feature | Description | Use Case |
|---------|-------------|----------|
| Label detection | Identify objects, scenes | Image categorization |
| Text detection (OCR) | Extract text from images | Receipt scanning |
| Face detection | Detect faces + emotions | Photo apps |
| Object localization | Bounding boxes around objects | Inventory counting |
| Safe search | Content moderation | User-uploaded content |
| Logo detection | Identify brand logos | Brand monitoring |
| Landmark detection | Identify places | Travel apps |
| Web detection | Find similar images online | Reverse image search |
| Product search | Find similar products | E-commerce |

### Pricing

| Feature | Cost |
|---------|------|
| Label, face, landmark, logo | $1.50/1K images (first 1K free) |
| Text detection (OCR) | $1.50/1K images |
| Object localization | $2.25/1K images |
| Safe search | $1.50/1K images |

---

## Part 3 — Cloud Natural Language API

### Text Analysis

```python
from google.cloud import language_v2

client = language_v2.LanguageServiceClient()

text = "Google Cloud Platform is amazing. The AI services are fast and accurate. I love using BigQuery for analytics."

document = language_v2.Document(
    content=text,
    type_=language_v2.Document.Type.PLAIN_TEXT,
)

# Sentiment analysis
response = client.analyze_sentiment(document=document)
print(f"Document sentiment: {response.document_sentiment.score:.2f}")
# Score: -1.0 (negative) to 1.0 (positive)
# Magnitude: 0.0 (neutral) to inf (strong)

for sentence in response.sentences:
    print(f"  '{sentence.text.content}' → {sentence.sentiment.score:.2f}")

# Entity analysis
response = client.analyze_entities(document=document)
for entity in response.entities:
    print(f"{entity.name} ({entity.type_.name}): salience={entity.salience:.2f}")
# Output: "Google Cloud Platform (ORGANIZATION): salience=0.45"
#         "BigQuery (CONSUMER_GOOD): salience=0.25"

# Entity sentiment analysis
response = client.analyze_entity_sentiment(document=document)
for entity in response.entities:
    print(f"{entity.name}: sentiment={entity.sentiment.score:.2f}")

# Syntax analysis (POS tagging, dependency tree)
response = client.analyze_syntax(document=document)
for token in response.tokens:
    print(f"{token.text.content}: {token.part_of_speech.tag.name}")

# Text classification
response = client.classify_text(document=document)
for category in response.categories:
    print(f"{category.name}: {category.confidence:.2f}")
# Output: "/Computers & Electronics/Cloud Computing: 0.95"
```

### Pricing

| Feature | Cost |
|---------|------|
| Sentiment analysis | $1.00/1K records (first 5K free) |
| Entity analysis | $1.00/1K records |
| Syntax analysis | $0.50/1K records |
| Classification | $2.00/1K records |

---

## Part 4 — Cloud Translation API

### Translation

```python
from google.cloud import translate_v2 as translate

client = translate.Client()

# Basic translation (v2)
result = client.translate('Hello, how are you?', target_language='es')
print(result['translatedText'])        # "Hola, ¿cómo estás?"
print(result['detectedSourceLanguage'])# "en"

# Detect language
result = client.detect_language('Bonjour le monde')
print(result['language'])              # "fr"
print(result['confidence'])           # 0.98

# List supported languages
languages = client.get_languages()
# 130+ languages supported
```

```python
# Translation API v3 (Advanced — glossaries, batch, models)
from google.cloud import translate_v3

client = translate_v3.TranslationServiceClient()
parent = f"projects/my-project/locations/us-central1"

# Translate with v3
response = client.translate_text(
    parent=parent,
    contents=['Hello world', 'Machine learning is powerful'],
    target_language_code='ja',
    source_language_code='en',
)
for translation in response.translations:
    print(translation.translated_text)

# Batch translation (large volumes)
operation = client.batch_translate_text(
    parent=parent,
    source_language_code='en',
    target_language_codes=['fr', 'de', 'ja'],
    input_configs=[{
        'gcs_source': {'input_uri': 'gs://my-bucket/input/*.txt'},
    }],
    output_config={
        'gcs_destination': {'output_uri_prefix': 'gs://my-bucket/output/'},
    },
)
result = operation.result()  # wait for completion
```

### Pricing

| Feature | Cost |
|---------|------|
| Translation (v2) | $20/million characters |
| Translation (v3 — NMT) | $20/million characters |
| Language detection | $20/million characters |
| Batch translation | $20/million characters |
| Free tier | 500K characters/month |

---

## Part 5 — Cloud Speech-to-Text

### Audio Transcription

```python
from google.cloud import speech

client = speech.SpeechClient()

# Transcribe audio file from GCS
audio = speech.RecognitionAudio(uri='gs://my-bucket/audio/meeting.wav')
config = speech.RecognitionConfig(
    encoding=speech.RecognitionConfig.AudioEncoding.LINEAR16,
    sample_rate_hertz=16000,
    language_code='en-US',
    enable_automatic_punctuation=True,
    enable_word_time_offsets=True,        # word-level timestamps
    model='latest_long',                  # best for long audio
)

# Short audio (< 1 min)
response = client.recognize(config=config, audio=audio)
for result in response.results:
    print(result.alternatives[0].transcript)
    print(f"Confidence: {result.alternatives[0].confidence:.2f}")

# Long audio (> 1 min) — async
operation = client.long_running_recognize(config=config, audio=audio)
response = operation.result(timeout=600)
for result in response.results:
    print(result.alternatives[0].transcript)

# Streaming (real-time)
import io

def stream_audio():
    with io.open('audio.wav', 'rb') as f:
        while True:
            data = f.read(4096)
            if not data:
                break
            yield speech.StreamingRecognizeRequest(audio_content=data)

streaming_config = speech.StreamingRecognitionConfig(
    config=config,
    interim_results=True,
)

responses = client.streaming_recognize(
    config=streaming_config,
    requests=stream_audio(),
)
for response in responses:
    for result in response.results:
        print(f"{'Final' if result.is_final else 'Interim'}: {result.alternatives[0].transcript}")
```

### Speech-to-Text v2 (Chirp)

```python
from google.cloud.speech_v2 import SpeechClient
from google.cloud.speech_v2.types import cloud_speech

client = SpeechClient()

# Chirp model — 100+ languages, higher accuracy
config = cloud_speech.RecognitionConfig(
    auto_decoding_config=cloud_speech.AutoDetectDecodingConfig(),
    language_codes=['en-US'],
    model='chirp',                       # Google's latest model
)

request = cloud_speech.RecognizeRequest(
    recognizer=f"projects/my-project/locations/us-central1/recognizers/_",
    config=config,
    uri='gs://my-bucket/audio/meeting.wav',
)

response = client.recognize(request=request)
```

### Pricing

| Feature | Cost |
|---------|------|
| Standard model | $0.006/15 seconds |
| Enhanced/Chirp | $0.009/15 seconds |
| Free tier | 60 minutes/month |

---

## Part 6 — Cloud Text-to-Speech

### Voice Synthesis

```python
from google.cloud import texttospeech

client = texttospeech.TextToSpeechClient()

# Basic text-to-speech
synthesis_input = texttospeech.SynthesisInput(text='Hello, welcome to Google Cloud!')

# WaveNet voice (high quality, natural sounding)
voice = texttospeech.VoiceSelectionParams(
    language_code='en-US',
    name='en-US-Neural2-F',               # Neural2 voice
    ssml_gender=texttospeech.SsmlVoiceGender.FEMALE,
)

audio_config = texttospeech.AudioConfig(
    audio_encoding=texttospeech.AudioEncoding.MP3,
    speaking_rate=1.0,                     # 0.25 to 4.0
    pitch=0.0,                             # -20.0 to 20.0
)

response = client.synthesize_speech(
    input=synthesis_input,
    voice=voice,
    audio_config=audio_config,
)

with open('output.mp3', 'wb') as out:
    out.write(response.audio_content)

# SSML input (fine control over pronunciation)
ssml_input = texttospeech.SynthesisInput(
    ssml="""
    <speak>
        <p>Welcome to <emphasis level="strong">Google Cloud</emphasis>.</p>
        <p>Your order number is <say-as interpret-as="characters">ABC123</say-as>.</p>
        <break time="500ms"/>
        <p>Thank you for choosing us.</p>
    </speak>
    """
)

# List available voices
voices = client.list_voices(language_code='en-US')
for voice in voices.voices:
    print(f"{voice.name} ({voice.ssml_gender.name})")
```

### Voice Types

| Type | Quality | Cost |
|------|---------|------|
| Standard | Basic | $4/million characters |
| WaveNet | High quality | $16/million characters |
| Neural2 | Most natural | $16/million characters |
| Studio | Premium | $160/million characters |
| Free tier | — | 1M standard / 100K WaveNet chars/month |

---

## Part 7 — Document AI

### Intelligent Document Processing

```
┌────────────────────────────────────────────────────────────────────┐
│         DOCUMENT AI                                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Extract structured data from unstructured documents:             │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Input: PDF, image, scanned document                      │     │
│  │       │                                                    │     │
│  │       ▼                                                    │     │
│  │  Document AI Processor                                    │     │
│  │  ├── OCR Processor       → extract all text              │     │
│  │  ├── Form Parser         → key-value pairs               │     │
│  │  ├── Invoice Parser      → invoice-specific fields       │     │
│  │  ├── Receipt Parser      → receipt-specific fields       │     │
│  │  ├── ID Parser           → identity document fields      │     │
│  │  ├── Contract Parser     → contract clauses              │     │
│  │  ├── W-2 / 1099 Parser   → tax form fields               │     │
│  │  ├── Lending Parser      → mortgage/loan documents       │     │
│  │  └── Custom Extractor    → train on your own documents   │     │
│  │       │                                                    │     │
│  │       ▼                                                    │     │
│  │  Output: structured JSON with extracted fields            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```python
from google.cloud import documentai

client = documentai.DocumentProcessorServiceClient()
processor_name = f"projects/my-project/locations/us/processors/PROCESSOR_ID"

# Process a document
with open('invoice.pdf', 'rb') as f:
    content = f.read()

request = documentai.ProcessRequest(
    name=processor_name,
    raw_document=documentai.RawDocument(
        content=content,
        mime_type='application/pdf',
    ),
)

result = client.process_document(request=request)
document = result.document

# Get full text
print(document.text)

# Get entities (structured fields)
for entity in document.entities:
    print(f"{entity.type_}: {entity.mention_text}")
    # Output for invoice:
    # "invoice_id: INV-2024-001"
    # "total_amount: $1,234.56"
    # "due_date: 2024-02-15"
    # "vendor_name: Acme Corp"
    # "line_item/description: Widget A"
    # "line_item/quantity: 10"
    # "line_item/unit_price: $123.45"

# Batch processing (large volumes)
operation = client.batch_process_documents(
    request=documentai.BatchProcessRequest(
        name=processor_name,
        input_documents=documentai.BatchDocumentsInputConfig(
            gcs_prefix=documentai.GcsPrefix(
                gcs_uri_prefix='gs://my-bucket/invoices/'
            )
        ),
        document_output_config=documentai.DocumentOutputConfig(
            gcs_output_config=documentai.DocumentOutputConfig.GcsOutputConfig(
                gcs_uri='gs://my-bucket/output/'
            )
        ),
    )
)
result = operation.result()
```

---

## Part 8 — Video Intelligence API

### Video Analysis

```python
from google.cloud import videointelligence

client = videointelligence.VideoIntelligenceServiceClient()

# Label detection (what's in the video)
operation = client.annotate_video(
    request={
        'input_uri': 'gs://my-bucket/video.mp4',
        'features': [videointelligence.Feature.LABEL_DETECTION],
    }
)

result = operation.result(timeout=300)
for annotation in result.annotation_results[0].segment_label_annotations:
    print(f"Label: {annotation.entity.description}")
    for segment in annotation.segments:
        print(f"  {segment.segment.start_time_offset} → {segment.segment.end_time_offset}")
        print(f"  Confidence: {segment.confidence:.2f}")

# Shot change detection
operation = client.annotate_video(
    request={
        'input_uri': 'gs://my-bucket/video.mp4',
        'features': [videointelligence.Feature.SHOT_CHANGE_DETECTION],
    }
)

# Object tracking
operation = client.annotate_video(
    request={
        'input_uri': 'gs://my-bucket/video.mp4',
        'features': [videointelligence.Feature.OBJECT_TRACKING],
    }
)

# Speech transcription from video
operation = client.annotate_video(
    request={
        'input_uri': 'gs://my-bucket/video.mp4',
        'features': [videointelligence.Feature.SPEECH_TRANSCRIPTION],
        'video_context': {
            'speech_transcription_config': {
                'language_code': 'en-US',
                'enable_automatic_punctuation': True,
            }
        },
    }
)

# Text detection in video (OCR on video frames)
operation = client.annotate_video(
    request={
        'input_uri': 'gs://my-bucket/video.mp4',
        'features': [videointelligence.Feature.TEXT_DETECTION],
    }
)

# Person detection
operation = client.annotate_video(
    request={
        'input_uri': 'gs://my-bucket/video.mp4',
        'features': [videointelligence.Feature.PERSON_DETECTION],
        'video_context': {
            'person_detection_config': {
                'include_bounding_boxes': True,
                'include_pose_landmarks': True,
                'include_attributes': True,
            }
        },
    }
)
```

---

## Part 9 — Recommendations AI

### Product Recommendations

```
┌────────────────────────────────────────────────────────────────────┐
│         RECOMMENDATIONS AI                                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Personalized product recommendations for e-commerce:             │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  1. Import catalog (products)                             │     │
│  │  2. Import user events (views, clicks, purchases)        │     │
│  │  3. Train recommendation model (automatic)               │     │
│  │  4. Serve predictions via API                            │     │
│  │                                                            │     │
│  │  Recommendation types:                                    │     │
│  │  • "Recommended for you" — personalized per user        │     │
│  │  • "Others also viewed" — item-item similarity          │     │
│  │  • "Frequently bought together" — co-purchase           │     │
│  │  • "Recently viewed" — user history                     │     │
│  │  • "Similar items" — content-based similarity           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Part of Vertex AI Search & Conversation (Retail)                 │
│  Pricing: $0.27/1K prediction requests                            │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 10 — Dialogflow CX

### Conversational AI

```
┌────────────────────────────────────────────────────────────────────┐
│         DIALOGFLOW CX                                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Build advanced conversational agents (chatbots/voicebots):       │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Agent                                                     │     │
│  │  ├── Flows (conversation paths)                           │     │
│  │  │    ├── Pages (conversation states)                    │     │
│  │  │    │    ├── Intents (user wants to do X)             │     │
│  │  │    │    ├── Parameters (extracted data)               │     │
│  │  │    │    ├── Fulfillment (webhook/response)           │     │
│  │  │    │    └── Transitions (→ next page)                │     │
│  │  │    └── Route Groups                                   │     │
│  │  ├── Entity Types (custom entities)                      │     │
│  │  ├── Webhooks (backend logic)                            │     │
│  │  └── Environments (draft/prod)                           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Channels: Web, Telephony, Messenger, Slack, etc.                 │
│  Languages: 30+ supported                                         │
│  Generative: Integrates with Gemini for generative responses     │
│                                                                      │
│  CX vs ES:                                                         │
│  • Dialogflow ES: simpler, flat intent structure                  │
│  • Dialogflow CX: flow-based, enterprise, visual builder         │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Pricing

| Feature | Cost |
|---------|------|
| Text requests | $0.007/request |
| Audio input | $0.001/second |
| Audio output (TTS) | $0.001/second |
| Design-time (API calls) | Free |

---

## Part 11 — Gemini API (Generative AI)

### Text, Code, and Multimodal Generation

```python
import vertexai
from vertexai.generative_models import GenerativeModel, Part, SafetySetting

vertexai.init(project='my-project', location='us-central1')

# ─── Text Generation ──────────────────────────────────────────
model = GenerativeModel('gemini-1.5-pro')

response = model.generate_content(
    'Write a Python function that finds the longest palindromic substring.',
    generation_config={
        'temperature': 0.2,
        'max_output_tokens': 2048,
        'top_p': 0.8,
        'top_k': 40,
    },
)
print(response.text)

# ─── Multimodal (Image + Text) ────────────────────────────────
image = Part.from_uri('gs://my-bucket/chart.png', mime_type='image/png')
response = model.generate_content([
    'Analyze this chart and summarize the key trends.',
    image,
])

# ─── Multimodal (Video + Text) ────────────────────────────────
video = Part.from_uri('gs://my-bucket/demo.mp4', mime_type='video/mp4')
response = model.generate_content([
    'Describe what happens in this video.',
    video,
])

# ─── Multimodal (Audio + Text) ────────────────────────────────
audio = Part.from_uri('gs://my-bucket/meeting.mp3', mime_type='audio/mp3')
response = model.generate_content([
    'Transcribe and summarize this meeting recording.',
    audio,
])

# ─── Structured Output (JSON) ─────────────────────────────────
response = model.generate_content(
    'Extract the product name, price, and description from this text: '
    '"The new iPhone 15 Pro Max costs $1199 and features a titanium design."',
    generation_config={
        'response_mime_type': 'application/json',
    },
)

# ─── Function Calling ─────────────────────────────────────────
from vertexai.generative_models import FunctionDeclaration, Tool

get_weather = FunctionDeclaration(
    name='get_weather',
    description='Get current weather for a location',
    parameters={
        'type': 'object',
        'properties': {
            'location': {'type': 'string', 'description': 'City name'},
            'unit': {'type': 'string', 'enum': ['celsius', 'fahrenheit']},
        },
        'required': ['location'],
    },
)

tool = Tool(function_declarations=[get_weather])
model = GenerativeModel('gemini-1.5-pro', tools=[tool])
response = model.generate_content("What's the weather in Tokyo?")
# Response includes function call to execute
```

### Model Comparison

| Model | Best For | Input | Output | Speed |
|-------|---------|-------|--------|-------|
| Gemini 1.5 Pro | Complex reasoning | Text, image, video, audio | Text | Medium |
| Gemini 1.5 Flash | Fast responses | Text, image, video, audio | Text | Fast |
| Gemini 1.0 Pro | General purpose | Text | Text | Fast |
| Gemini 1.0 Pro Vision | Image understanding | Text + image | Text | Medium |

---

## Part 12 — Imagen API (Image Generation)

### Generate and Edit Images

```python
import vertexai
from vertexai.preview.vision_models import ImageGenerationModel

vertexai.init(project='my-project', location='us-central1')

model = ImageGenerationModel.from_pretrained('imagen-3.0-generate-001')

# Generate image from text prompt
images = model.generate_images(
    prompt='A serene mountain lake at sunset with pine trees, photorealistic',
    number_of_images=4,
    aspect_ratio='16:9',
    safety_filter_level='block_some',
)

# Save generated images
for i, image in enumerate(images):
    image.save(f'generated_{i}.png')

# Edit image (inpainting)
from vertexai.preview.vision_models import Image

base_image = Image.load_from_file('photo.png')
mask_image = Image.load_from_file('mask.png')

edited = model.edit_image(
    base_image=base_image,
    mask=mask_image,
    prompt='Replace the sky with a dramatic sunset',
)
edited.save('edited.png')
```

---

## Part 13 — Embedding APIs

### Text & Multimodal Embeddings

```python
from vertexai.language_models import TextEmbeddingModel

# Text embeddings (for semantic search, RAG, classification)
model = TextEmbeddingModel.from_pretrained('text-embedding-005')

embeddings = model.get_embeddings([
    'How to set up a GKE cluster',
    'Kubernetes cluster creation guide',
    'Best Italian restaurants in New York',
])

for embedding in embeddings:
    print(f"Dimensions: {len(embedding.values)}")       # 768
    print(f"First 5 values: {embedding.values[:5]}")

# Multimodal embeddings (text + image in same space)
from vertexai.vision_models import MultiModalEmbeddingModel, Image

mm_model = MultiModalEmbeddingModel.from_pretrained('multimodalembedding')

# Embed image
image = Image.load_from_file('product.jpg')
img_embedding = mm_model.get_embeddings(image=image)

# Embed text
txt_embedding = mm_model.get_embeddings(
    contextual_text='red running shoes'
)

# Compare cosine similarity between image and text embeddings
# Use for cross-modal search (text → image, image → text)
```

### Pricing

| Model | Cost |
|-------|------|
| text-embedding-005 | $0.000025/1K characters |
| multimodalembedding | $0.0001/image, $0.000025/1K chars |
| Free tier | Generous free quotas |

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Enable APIs ──────────────────────────────────────────────
resource "google_project_service" "vision" {
  project = var.project_id
  service = "vision.googleapis.com"
}

resource "google_project_service" "language" {
  project = var.project_id
  service = "language.googleapis.com"
}

resource "google_project_service" "translate" {
  project = var.project_id
  service = "translate.googleapis.com"
}

resource "google_project_service" "speech" {
  project = var.project_id
  service = "speech.googleapis.com"
}

resource "google_project_service" "texttospeech" {
  project = var.project_id
  service = "texttospeech.googleapis.com"
}

resource "google_project_service" "documentai" {
  project = var.project_id
  service = "documentai.googleapis.com"
}

resource "google_project_service" "videointelligence" {
  project = var.project_id
  service = "videointelligence.googleapis.com"
}

resource "google_project_service" "aiplatform" {
  project = var.project_id
  service = "aiplatform.googleapis.com"
}

# ─── Document AI Processor ───────────────────────────────────
resource "google_document_ai_processor" "invoice" {
  location     = "us"
  display_name = "invoice-parser"
  type         = "INVOICE_PROCESSOR"
  project      = var.project_id
}

# ─── Service Account for AI APIs ─────────────────────────────
resource "google_service_account" "ai_service" {
  account_id   = "ai-api-sa"
  display_name = "AI/ML API Service Account"
  project      = var.project_id
}

resource "google_project_iam_member" "ai_roles" {
  for_each = toset([
    "roles/aiplatform.user",
    "roles/documentai.viewer",
    "roles/storage.objectViewer",
  ])
  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.ai_service.email}"
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# ENABLE APIs
# ═══════════════════════════════════════════════════════════════
gcloud services enable vision.googleapis.com
gcloud services enable language.googleapis.com
gcloud services enable translate.googleapis.com
gcloud services enable speech.googleapis.com
gcloud services enable texttospeech.googleapis.com
gcloud services enable documentai.googleapis.com
gcloud services enable videointelligence.googleapis.com
gcloud services enable aiplatform.googleapis.com

# ═══════════════════════════════════════════════════════════════
# VISION
# ═══════════════════════════════════════════════════════════════
gcloud ml vision detect-labels gs://bucket/image.jpg
gcloud ml vision detect-text gs://bucket/image.jpg
gcloud ml vision detect-faces gs://bucket/image.jpg
gcloud ml vision detect-objects gs://bucket/image.jpg
gcloud ml vision detect-logos gs://bucket/image.jpg
gcloud ml vision suggest-crop gs://bucket/image.jpg

# ═══════════════════════════════════════════════════════════════
# LANGUAGE
# ═══════════════════════════════════════════════════════════════
gcloud ml language analyze-sentiment --content="Great product!"
gcloud ml language analyze-entities --content="Google Cloud in NYC"
gcloud ml language classify-text --content="Article about AI..."

# ═══════════════════════════════════════════════════════════════
# TRANSLATE
# ═══════════════════════════════════════════════════════════════
gcloud ml translate translate-text --content="Hello" --target-language=es

# ═══════════════════════════════════════════════════════════════
# SPEECH
# ═══════════════════════════════════════════════════════════════
gcloud ml speech recognize gs://bucket/audio.wav \
    --language-code=en-US \
    --enable-automatic-punctuation

# ═══════════════════════════════════════════════════════════════
# DOCUMENT AI
# ═══════════════════════════════════════════════════════════════
gcloud document-ai processors list --location=us
gcloud document-ai processors create --type=INVOICE_PROCESSOR \
    --display-name=NAME --location=us
```

---

## Part 15 — Real-World Patterns

### Pattern 1: Intelligent Document Processing Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: DOCUMENT PROCESSING                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Email with invoice attachment arrives                     │        │
│  │       │                                                    │        │
│  │       ▼ Eventarc (GCS upload trigger)                    │        │
│  │  ┌──────────────────────────────────────────────┐        │        │
│  │  │ Cloud Function: classify-document             │        │        │
│  │  │ → Vision API: detect text + classify type    │        │        │
│  │  │ → Route to appropriate processor              │        │        │
│  │  └──────────────┬──────────────┬────────────────┘        │        │
│  │                  │              │                           │        │
│  │          ┌───────▼──┐  ┌───────▼──────┐                  │        │
│  │          │ Invoice  │  │ Receipt      │                  │        │
│  │          │ Parser   │  │ Parser       │                  │        │
│  │          │(Doc AI)  │  │(Doc AI)      │                  │        │
│  │          └────┬─────┘  └──────┬───────┘                  │        │
│  │               └───────┬───────┘                           │        │
│  │                       ▼                                    │        │
│  │              ┌──────────────────┐                         │        │
│  │              │ Cloud Function:  │                         │        │
│  │              │ validate + store │                         │        │
│  │              │ → BigQuery       │                         │        │
│  │              │ → Firestore      │                         │        │
│  │              └──────────────────┘                         │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Volume: 10,000+ invoices/month auto-processed                     │
│  Accuracy: 95%+ field extraction                                    │
│  Human review: only flagged low-confidence extractions             │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Multilingual Content Platform

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: MULTILINGUAL CONTENT                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Content creator writes article (English)                  │        │
│  │       │                                                    │        │
│  │       ▼                                                    │        │
│  │  Cloud Function: content-pipeline                         │        │
│  │  ├── NLP API: extract entities + sentiment               │        │
│  │  ├── NLP API: classify content category                  │        │
│  │  ├── Translation API: translate to 10 languages          │        │
│  │  ├── Gemini: generate localized SEO metadata             │        │
│  │  ├── Text-to-Speech: generate audio version              │        │
│  │  │   (Neural2 voices in each language)                   │        │
│  │  ├── Vision API: analyze/tag article images              │        │
│  │  └── Imagen: generate article thumbnail                  │        │
│  │       │                                                    │        │
│  │       ▼                                                    │        │
│  │  ┌──────────────────────────────────────────────┐        │        │
│  │  │ Output:                                       │        │        │
│  │  │ ├── 10 translated articles (GCS)             │        │        │
│  │  │ ├── 10 audio files (GCS)                     │        │        │
│  │  │ ├── SEO metadata (Firestore)                 │        │        │
│  │  │ ├── Entity/category tags (BigQuery)          │        │        │
│  │  │ └── Thumbnail image (GCS)                    │        │        │
│  │  └──────────────────────────────────────────────┘        │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: AI-Powered Customer Support

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: CUSTOMER SUPPORT                                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Customer contacts support (chat/voice/email):            │        │
│  │                                                            │        │
│  │  Voice call:                                              │        │
│  │  ├── Speech-to-Text (Chirp) → transcribe                │        │
│  │  ├── NLP API → detect intent + sentiment                 │        │
│  │  ├── Dialogflow CX → route conversation                  │        │
│  │  │   ├── Simple queries → Gemini + RAG answer           │        │
│  │  │   ├── Order status → webhook → order API             │        │
│  │  │   └── Complex → transfer to human agent              │        │
│  │  └── Text-to-Speech → respond to caller                  │        │
│  │                                                            │        │
│  │  Chat:                                                    │        │
│  │  ├── Dialogflow CX → same flow as voice                 │        │
│  │  ├── Gemini → context-aware responses                    │        │
│  │  └── Grounding → search knowledge base (Vector Search)  │        │
│  │                                                            │        │
│  │  Email:                                                   │        │
│  │  ├── NLP API → classify ticket category + priority       │        │
│  │  ├── Translation API → detect language, translate        │        │
│  │  ├── Document AI → extract attachments (receipts, etc.)  │        │
│  │  ├── Gemini → draft response for agent review            │        │
│  │  └── Route to appropriate support team                   │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  All interactions logged to BigQuery for analysis                   │
│  NLP sentiment tracked to measure customer satisfaction             │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| API | Use Case | gcloud Command |
|-----|----------|---------------|
| Vision | Image analysis, OCR | `gcloud ml vision detect-labels gs://...` |
| NLP | Sentiment, entities | `gcloud ml language analyze-sentiment --content=...` |
| Translation | 130+ languages | `gcloud ml translate translate-text --target-language=es` |
| Speech-to-Text | Audio transcription | `gcloud ml speech recognize gs://...` |
| Text-to-Speech | Voice synthesis | Python SDK only |
| Document AI | Invoice/receipt extraction | `gcloud document-ai processors list` |
| Video Intelligence | Video labels/tracking | Python SDK only |
| Dialogflow CX | Chatbots/voicebots | Console + API |
| Gemini | Generative AI | `vertexai.GenerativeModel('gemini-1.5-pro')` |
| Imagen | Image generation | Python SDK only |
| Embeddings | Semantic search, RAG | `TextEmbeddingModel.get_embeddings()` |

---

## Getting Started with AI APIs (Beginner Guide)

Google Cloud AI APIs are **pre-trained models you can call with a simple REST request** — no ML expertise needed. You send data (an image, text, audio), and the API returns predictions (labels, sentiment, translation). Here's how to get started from zero.

### Step 1: Enable the API

Before using any AI API, you must enable it in your project. Each API is a separate service:

```bash
# Enable the APIs you need
gcloud services enable vision.googleapis.com           # Vision API
gcloud services enable language.googleapis.com         # Natural Language API
gcloud services enable translate.googleapis.com        # Translation API
gcloud services enable speech.googleapis.com           # Speech-to-Text
gcloud services enable texttospeech.googleapis.com     # Text-to-Speech
gcloud services enable documentai.googleapis.com       # Document AI
gcloud services enable videointelligence.googleapis.com # Video Intelligence

# Check which APIs are enabled
gcloud services list --enabled --filter="name:vision OR name:language OR name:translate"
```

```
Console → APIs & Services → Library → Search for "Vision" → ENABLE

┌─────────────────────────────────────────────────────────────────┐
│           CLOUD VISION API                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Cloud Vision API                                                │
│ Google Enterprise API                                           │
│                                                                   │
│ Integrates Google Vision features, including image labeling,    │
│ face and landmark detection, OCR, and explicit content detection │
│                                                                   │
│ Status: [ENABLED ✓]                                             │
│                                                                   │
│ ⚡ Each API must be enabled separately.                          │
│   Enabling is free — you only pay when you make API calls.      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Set Up Authentication

You need credentials to call the APIs. Two common approaches:

```bash
# Option A: Use your own credentials (for local development/testing)
gcloud auth application-default login
# This stores credentials locally — client libraries pick them up automatically

# Option B: Create a service account (for production/automation)
gcloud iam service-accounts create ai-api-caller \
    --display-name="AI API Service Account"

# Grant it permission to use AI APIs
gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:ai-api-caller@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/aiplatform.user"

# Download the key (for non-GCP environments)
gcloud iam service-accounts keys create key.json \
    --iam-account=ai-api-caller@PROJECT_ID.iam.gserviceaccount.com

# Set the environment variable so client libraries find the key
export GOOGLE_APPLICATION_CREDENTIALS="key.json"
```

> **On GCP services** (Cloud Run, Compute Engine, Cloud Functions), you don't need a key file — the default service account credentials are available automatically.

### Step 3: Make Your First API Call

Here's a simple curl example to analyze an image with the Vision API:

```bash
# Analyze an image using Vision API with curl
curl -X POST \
  "https://vision.googleapis.com/v1/images:annotate" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "requests": [{
      "image": {
        "source": {
          "imageUri": "gs://cloud-samples-data/vision/label/wakeupcat.jpg"
        }
      },
      "features": [{
        "type": "LABEL_DETECTION",
        "maxResults": 5
      }]
    }]
  }'

# Response (simplified):
# {
#   "responses": [{
#     "labelAnnotations": [
#       { "description": "Cat", "score": 0.98 },
#       { "description": "Whiskers", "score": 0.95 },
#       { "description": "Small to medium-sized cats", "score": 0.93 }
#     ]
#   }]
# }
```

```bash
# Analyze sentiment with Natural Language API
curl -X POST \
  "https://language.googleapis.com/v2/documents:analyzeSentiment" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
    "document": {
      "type": "PLAIN_TEXT",
      "content": "Google Cloud is amazing! I love the AI APIs."
    }
  }'

# Response: score=0.9 (very positive), magnitude=1.8 (strong feeling)
```

### Step 4: Test in the Console ("Try this API")

Google provides a built-in API explorer for every API — no code needed:

```
How to find it:
1. Go to the API's documentation page (e.g., cloud.google.com/vision/docs)
2. Look for "Try this API" or "API Explorer" links
3. Or go directly: Console → APIs & Services → Click the enabled API → "Try this API"

┌─────────────────────────────────────────────────────────────────┐
│           TRY THIS API — Vision API                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ The API Explorer lets you:                                      │
│ • Fill in request parameters in a form                         │
│ • Click EXECUTE to make a real API call                        │
│ • See the full JSON response                                   │
│ • Copy the equivalent curl command                             │
│                                                                   │
│ ⚡ Great for testing before writing code!                        │
│   Uses your logged-in Google account credentials.              │
│   No setup needed — just fill in the form and click Execute.   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Testing AI APIs

### Testing Vision API from Console

```
Console → APIs & Services → Library → Cloud Vision API → TRY THIS API
Or: Console → Vertex AI → Vision (under "Deploy and Use")

┌─────────────────────────────────────────────────────────────────┐
│           VISION API — DRAG & DROP DEMO                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ The Vision API page has a built-in demo:                        │
│                                                                   │
│ 1. Go to: cloud.google.com/vision#demo                         │
│ 2. Drag and drop any image (or paste a URL)                    │
│ 3. Instantly see results:                                      │
│                                                                   │
│    ┌──────────────────┐  Results:                              │
│    │                  │  ├── Labels: Cat (98%), Animal (97%)   │
│    │   [Your Image]   │  ├── Objects: 1 cat detected          │
│    │                  │  ├── Text: (any text found via OCR)    │
│    │                  │  ├── Faces: (emotions detected)        │
│    └──────────────────┘  └── Safe Search: all safe             │
│                                                                   │
│ ⚡ This uses your project quota — but the free tier covers it.  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# CLI: Quick test — detect labels on a public sample image
gcloud ml vision detect-labels gs://cloud-samples-data/vision/label/wakeupcat.jpg

# CLI: Detect text (OCR) in an image
gcloud ml vision detect-text gs://my-bucket/receipt.jpg

# CLI: Detect objects with bounding boxes
gcloud ml vision detect-objects gs://my-bucket/photo.jpg
```

### Testing Natural Language API from Console

```
Console → APIs & Services → Library → Cloud Natural Language API → TRY THIS API
Or: cloud.google.com/natural-language#demo

┌─────────────────────────────────────────────────────────────────┐
│           NATURAL LANGUAGE API — DEMO                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ 1. Go to: cloud.google.com/natural-language#demo               │
│ 2. Paste any text or enter a URL                               │
│ 3. Click ANALYZE                                               │
│                                                                   │
│ Input: "Google Cloud is an amazing platform. The support       │
│         team is very helpful and responsive."                  │
│                                                                   │
│ Results:                                                       │
│ ├── Sentiment:                                                 │
│ │   Score: 0.8 (positive) | Magnitude: 1.6 (strong)          │
│ │                                                              │
│ ├── Entities:                                                  │
│ │   • Google Cloud (ORGANIZATION) — salience: 0.65            │
│ │   • platform (OTHER) — salience: 0.20                       │
│ │   • support team (OTHER) — salience: 0.15                   │
│ │                                                              │
│ ├── Syntax:                                                    │
│ │   Parts of speech for each word (noun, verb, adj, etc.)     │
│ │                                                              │
│ └── Categories:                                                │
│     /Computers & Electronics/Cloud Computing: 0.95            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# CLI: Analyze sentiment
gcloud ml language analyze-sentiment --content="This product is great! I love it."

# CLI: Analyze entities
gcloud ml language analyze-entities --content="Google Cloud in San Francisco"

# CLI: Classify text
gcloud ml language classify-text --content="Machine learning is transforming healthcare..."
```

### Testing Translation API from Console

```
Console → APIs & Services → Library → Cloud Translation API → TRY THIS API
Or: Console → Translation (under AI & Machine Learning)

┌─────────────────────────────────────────────────────────────────┐
│           TRANSLATION API — QUICK TEST                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ The easiest way to test Translation API:                       │
│                                                                   │
│ 1. Use the API Explorer:                                       │
│    Console → APIs & Services → Cloud Translation API           │
│    → "Try this API" button                                     │
│                                                                   │
│ 2. Or use gcloud CLI:                                          │
│                                                                   │
│    $ gcloud ml translate translate-text \                      │
│        --content="Hello, how are you?" \                       │
│        --target-language=es                                    │
│                                                                   │
│    Result: "Hola, ¿cómo estás?"                                │
│                                                                   │
│ 3. Supported languages: 130+                                   │
│    Common codes: es (Spanish), fr (French), de (German),       │
│    ja (Japanese), zh (Chinese), ko (Korean), hi (Hindi),       │
│    ar (Arabic), pt (Portuguese), ru (Russian)                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# CLI: Translate text
gcloud ml translate translate-text \
    --content="Machine learning is the future" \
    --target-language=ja

# CLI: Detect language
gcloud ml translate detect-language --content="Bonjour le monde"
# Result: language=fr, confidence=1.0
```

### Enable/Disable API Commands Reference

```bash
# ═══════════════════════════════════════════════════════════════
# ENABLE APIs
# ═══════════════════════════════════════════════════════════════
gcloud services enable vision.googleapis.com
gcloud services enable language.googleapis.com
gcloud services enable translate.googleapis.com
gcloud services enable speech.googleapis.com
gcloud services enable texttospeech.googleapis.com
gcloud services enable documentai.googleapis.com
gcloud services enable videointelligence.googleapis.com

# ═══════════════════════════════════════════════════════════════
# DISABLE APIs (when no longer needed — stops any charges)
# ═══════════════════════════════════════════════════════════════
gcloud services disable vision.googleapis.com
gcloud services disable language.googleapis.com
gcloud services disable translate.googleapis.com
gcloud services disable speech.googleapis.com
gcloud services disable texttospeech.googleapis.com
gcloud services disable documentai.googleapis.com
gcloud services disable videointelligence.googleapis.com

# ═══════════════════════════════════════════════════════════════
# CHECK STATUS
# ═══════════════════════════════════════════════════════════════
# List all enabled APIs
gcloud services list --enabled

# Check if a specific API is enabled
gcloud services list --enabled --filter="name:vision.googleapis.com"

# List available (not yet enabled) AI APIs
gcloud services list --available --filter="name:vision OR name:language OR name:translate"
```

> **Tip:** Disabling an API does NOT delete any data or resources — it just prevents new API calls. You can re-enable it anytime. This is useful for ensuring you're not accidentally billed for APIs you're no longer using.

---

## What's Next?

Continue to **Chapter 60: Migration Strategies** → `60-migration-strategies.md`
