# Spring AI — Complete Notes

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Project Setup](#2-project-setup)
3. [Core Abstractions](#3-core-abstractions)
4. [ChatClient — Fluent API (Recommended)](#4-chatclient--fluent-api-recommended)
5. [Prompt Engineering](#5-prompt-engineering)
6. [Structured Output](#6-structured-output)
7. [Chat Options & Model Parameters](#7-chat-options--model-parameters)
8. [Multimodal (Vision, Audio)](#8-multimodal-vision-audio)
9. [Function / Tool Calling](#9-function--tool-calling)
10. [Advisors (Middleware/Interceptors)](#10-advisors-middlewareinterceptors)
11. [Chat Memory (Conversation History)](#11-chat-memory-conversation-history)
12. [Embeddings](#12-embeddings)
13. [Vector Stores](#13-vector-stores)
14. [Document Loading (ETL Pipeline)](#14-document-loading-etl-pipeline)
15. [RAG (Retrieval Augmented Generation)](#15-rag-retrieval-augmented-generation)
16. [Model Evaluation](#16-model-evaluation)
17. [Error Handling & Retry](#17-error-handling--retry)
18. [Observability & Monitoring](#18-observability--monitoring)
19. [Multiple Models & Providers](#19-multiple-models--providers)
20. [Ollama (Local Models)](#20-ollama-local-models)
21. [Common Patterns & Recipes](#21-common-patterns--recipes)
22. [Testing Spring AI](#22-testing-spring-ai)
23. [Best Practices](#23-best-practices)
24. [Quick Reference Cheat Sheet](#24-quick-reference-cheat-sheet)

---

## 1. INTRODUCTION

### What is Spring AI?
- **Spring Framework module** for integrating **AI/ML models** into Java applications.
- Provides a **portable, unified API** across multiple AI providers.
- Created by **Spring team (VMware/Broadcom)**, first stable release 2024.
- Follows Spring principles — DI, auto-configuration, abstraction.
- Supports **chat models, embeddings, image generation, audio, vector stores, RAG, function calling, and more**.

### Why Spring AI?
| Without Spring AI | With Spring AI |
|-------------------|----------------|
| Direct HTTP calls to each AI API | Unified abstraction layer |
| Different SDK per provider | Single API, swap providers |
| Manual prompt construction | Prompt templates & management |
| Custom RAG implementation | Built-in RAG support |
| No Spring integration | Full Spring Boot auto-config |
| Manual JSON parsing | Structured output mapping |

### Supported AI Providers
| Provider | Chat | Embedding | Image | Audio | Moderation |
|----------|------|-----------|-------|-------|------------|
| **OpenAI** (GPT-4, GPT-4o) | ✅ | ✅ | ✅ (DALL-E) | ✅ (Whisper, TTS) | ✅ |
| **Azure OpenAI** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Anthropic** (Claude) | ✅ | — | — | — | — |
| **Google Vertex AI** (Gemini) | ✅ | ✅ | ✅ | — | — |
| **Google AI** (Gemini) | ✅ | ✅ | — | — | — |
| **Amazon Bedrock** | ✅ | ✅ | ✅ | — | — |
| **Mistral AI** | ✅ | ✅ | — | — | — |
| **Ollama** (local models) | ✅ | ✅ | — | — | — |
| **HuggingFace** | ✅ | ✅ | — | — | — |
| **Groq** | ✅ | — | — | — | — |
| **DeepSeek** | ✅ | — | — | — | — |

### Spring AI Architecture
```
Your Application
    ↓
Spring AI Abstraction Layer (ChatModel, EmbeddingModel, etc.)
    ↓
Provider-Specific Implementation (OpenAiChatModel, AnthropicChatModel, etc.)
    ↓
AI Provider API (OpenAI, Anthropic, Ollama, etc.)
```

---

## 2. PROJECT SETUP

### Maven Dependencies
```xml
<!-- Spring AI BOM (Bill of Materials) — version management -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Pick ONE chat model provider -->

<!-- OpenAI -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>

<!-- Anthropic (Claude) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
</dependency>

<!-- Ollama (local models) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
</dependency>

<!-- Azure OpenAI -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-azure-openai-spring-boot-starter</artifactId>
</dependency>

<!-- Google Vertex AI (Gemini) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-vertex-ai-gemini-spring-boot-starter</artifactId>
</dependency>

<!-- Mistral AI -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-mistralai-spring-boot-starter</artifactId>
</dependency>

<!-- Amazon Bedrock -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-bedrock-ai-spring-boot-starter</artifactId>
</dependency>
```

### Spring Initializr
```
https://start.spring.io/
→ Add dependency: "OpenAI" or "Ollama" or "Anthropic" etc.
→ Spring AI starters appear under "AI" category
```

### Repository (if needed for milestones/snapshots)
```xml
<repositories>
    <repository>
        <id>spring-milestones</id>
        <url>https://repo.spring.io/milestone</url>
    </repository>
    <repository>
        <id>spring-snapshots</id>
        <url>https://repo.spring.io/snapshot</url>
    </repository>
</repositories>
```

### Application Properties
```yaml
# application.yml

# === OpenAI ===
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}        # use env variable — never hardcode!
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
      embedding:
        options:
          model: text-embedding-3-small
      image:
        options:
          model: dall-e-3

# === Anthropic ===
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-20250514
          max-tokens: 4096

# === Ollama (local — no API key needed) ===
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3
      embedding:
        options:
          model: nomic-embed-text

# === Azure OpenAI ===
spring:
  ai:
    azure:
      openai:
        api-key: ${AZURE_OPENAI_API_KEY}
        endpoint: https://my-resource.openai.azure.com/
        chat:
          options:
            deployment-name: gpt-4o
            temperature: 0.7
```

---

## 3. CORE ABSTRACTIONS

### Key Interfaces
```
ChatModel          → Send prompts, get responses (text generation)
StreamingChatModel → Stream responses token by token
EmbeddingModel     → Convert text to vector embeddings
ImageModel         → Generate images from text
AudioModel         → Text-to-speech / speech-to-text
ModerationModel    → Content moderation
ChatClient         → High-level fluent API for chat (recommended)
```

### ChatModel Interface
```java
public interface ChatModel extends Model<Prompt, ChatResponse> {
    ChatResponse call(Prompt prompt);
    default String call(String message) { ... }
}

// ChatResponse contains:
// - List<Generation> results
// - ChatResponseMetadata (usage, rate limits)

// Generation contains:
// - AssistantMessage output
// - ChatGenerationMetadata (finish reason)
```

### Message Types
```java
// Messages represent conversation turns
Message userMsg    = new UserMessage("What is Spring AI?");
Message systemMsg  = new SystemMessage("You are a Java expert.");
Message assistMsg  = new AssistantMessage("Spring AI is...");

// Message hierarchy:
// Message (interface)
// ├── UserMessage        → user input
// ├── SystemMessage      → system instructions
// ├── AssistantMessage   → AI response
// └── ToolResponseMessage → function/tool result
```

---

## 4. ChatClient — FLUENT API (RECOMMENDED)

### What is ChatClient?
- **High-level**, **fluent builder** API for interacting with AI models.
- Recommended over calling `ChatModel` directly.
- Supports prompts, system messages, parameters, advisors, output conversion.

### Basic Usage
```java
@Service
public class AiService {

    private final ChatClient chatClient;

    // Inject ChatClient.Builder, build with defaults
    public AiService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    // Simple call
    public String ask(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();    // returns String response
    }
}
```

### With System Message
```java
public String askExpert(String question) {
    return chatClient.prompt()
        .system("You are a senior Java developer. Answer concisely.")
        .user(question)
        .call()
        .content();
}

// System message from resource file
public String askExpert(String question) {
    return chatClient.prompt()
        .system(s -> s.text(new ClassPathResource("prompts/system.txt")))
        .user(question)
        .call()
        .content();
}
```

### Default System Message (Applied to All Calls)
```java
@Service
public class AiService {
    private final ChatClient chatClient;

    public AiService(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are a helpful assistant for a Java learning platform.")
            .build();
    }

    // Every call() now includes the default system message
    public String ask(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
```

### With Parameters (Model Options)
```java
public String ask(String question) {
    return chatClient.prompt()
        .user(question)
        .options(ChatOptionsBuilder.builder()
            .withModel("gpt-4o")
            .withTemperature(0.3)
            .withMaxTokens(500)
            .build())
        .call()
        .content();
}
```

### Full ChatResponse (Metadata)
```java
public ChatResponse askFull(String question) {
    ChatResponse response = chatClient.prompt()
        .user(question)
        .call()
        .chatResponse();

    // Access metadata
    String content = response.getResult().getOutput().getText();
    Usage usage = response.getMetadata().getUsage();
    long promptTokens = usage.getPromptTokens();
    long completionTokens = usage.getCompletionTokens();
    long totalTokens = usage.getTotalTokens();

    return response;
}
```

### Streaming Responses
```java
public Flux<String> askStream(String question) {
    return chatClient.prompt()
        .user(question)
        .stream()
        .content();    // Flux<String> — token by token
}

// In a REST controller
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> streamAnswer(@RequestParam String question) {
    return chatClient.prompt()
        .user(question)
        .stream()
        .content();
}
```

### ChatClient Bean Configuration
```java
@Configuration
public class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder
            .defaultSystem("You are a helpful coding assistant.")
            .defaultOptions(ChatOptionsBuilder.builder()
                .withModel("gpt-4o")
                .withTemperature(0.7)
                .build())
            .build();
    }
}
```

---

## 5. PROMPT ENGINEERING

### Prompt Object
```java
// Simple prompt
Prompt prompt = new Prompt("What is Spring AI?");

// With messages
Prompt prompt = new Prompt(List.of(
    new SystemMessage("You are a Java expert."),
    new UserMessage("Explain dependency injection.")
));

// With options
Prompt prompt = new Prompt(
    "Explain Spring Boot",
    OpenAiChatOptions.builder()
        .withModel("gpt-4o")
        .withTemperature(0.5)
        .build()
);
```

### PromptTemplate — Dynamic Prompts
```java
// Inline template
PromptTemplate template = new PromptTemplate(
    "Explain the concept of {topic} in {language} programming. "
    + "Give {count} examples."
);

Prompt prompt = template.create(Map.of(
    "topic", "polymorphism",
    "language", "Java",
    "count", "3"
));

ChatResponse response = chatModel.call(prompt);
```

### Template from Resource File
```java
// src/main/resources/prompts/explain.st
// Explain the concept of {topic} in {language} programming.
// Provide {count} practical examples.
// Format the response as markdown.

@Value("classpath:prompts/explain.st")
private Resource explainPrompt;

public String explain(String topic, String language) {
    PromptTemplate template = new PromptTemplate(explainPrompt);
    Prompt prompt = template.create(Map.of(
        "topic", topic,
        "language", language,
        "count", "3"
    ));
    return chatModel.call(prompt).getResult().getOutput().getText();
}
```

### System + User Template with ChatClient
```java
public String codeReview(String code) {
    return chatClient.prompt()
        .system("You are a senior code reviewer. Be concise and actionable.")
        .user(u -> u.text("Review this code and suggest improvements:\n\n{code}")
                     .param("code", code))
        .call()
        .content();
}
```

### Roles Pattern (Multi-turn Conversation)
```java
public String chat(List<Message> conversationHistory, String userInput) {
    List<Message> messages = new ArrayList<>(conversationHistory);
    messages.add(new UserMessage(userInput));

    ChatResponse response = chatModel.call(new Prompt(messages));
    String reply = response.getResult().getOutput().getText();

    // Add assistant reply to history for next turn
    messages.add(new AssistantMessage(reply));

    return reply;
}
```

### Prompt Best Practices
```
1. Be specific and clear in instructions
2. Use system messages for persona/behavior/rules
3. Use templates for reusable prompts
4. Keep templates in resource files (not hardcoded)
5. Set temperature: 0.0-0.3 for factual, 0.7-1.0 for creative
6. Set max_tokens to control response length and cost
7. Use few-shot examples in the prompt when needed
8. Separate instructions from data with clear delimiters
```

---

## 6. STRUCTURED OUTPUT

### What is Structured Output?
- Parse AI responses into **Java objects** instead of raw strings.
- Uses `BeanOutputConverter`, `ListOutputConverter`, `MapOutputConverter`.
- Spring AI appends format instructions to the prompt automatically.

### Entity (Single Object)
```java
// Define your record/class
public record BookRecommendation(
    String title,
    String author,
    int year,
    String genre,
    String summary
) {}

// Using ChatClient — entity()
public BookRecommendation getRecommendation(String topic) {
    return chatClient.prompt()
        .user("Recommend a book about " + topic)
        .call()
        .entity(BookRecommendation.class);     // auto-parses into object
}
```

### List of Entities
```java
public List<BookRecommendation> getRecommendations(String topic) {
    return chatClient.prompt()
        .user("Recommend 5 books about " + topic)
        .call()
        .entity(new ParameterizedTypeReference<List<BookRecommendation>>() {});
}
```

### Using BeanOutputConverter Directly
```java
public BookRecommendation getBook(String topic) {
    BeanOutputConverter<BookRecommendation> converter =
        new BeanOutputConverter<>(BookRecommendation.class);

    String format = converter.getFormat();    // JSON schema instructions

    Prompt prompt = new Prompt(
        "Recommend a book about " + topic + "\n" + format
    );

    ChatResponse response = chatModel.call(prompt);
    return converter.convert(response.getResult().getOutput().getText());
}
```

### ListOutputConverter
```java
ListOutputConverter converter = new ListOutputConverter(new DefaultConversionService());

String format = converter.getFormat();    // instructs AI to return comma-separated list

Prompt prompt = new Prompt(
    "List 10 programming languages.\n" + format
);

List<String> languages = converter.convert(
    chatModel.call(prompt).getResult().getOutput().getText()
);
```

### MapOutputConverter
```java
MapOutputConverter converter = new MapOutputConverter();

String format = converter.getFormat();

Prompt prompt = new Prompt(
    "Provide a JSON map of 5 countries and their capitals.\n" + format
);

Map<String, Object> countries = converter.convert(
    chatModel.call(prompt).getResult().getOutput().getText()
);
```

### Complex Nested Types
```java
public record CourseOutline(
    String title,
    String description,
    List<Module> modules
) {
    public record Module(
        String name,
        int durationHours,
        List<String> topics
    ) {}
}

public CourseOutline generateCourse(String subject) {
    return chatClient.prompt()
        .system("You are a course designer.")
        .user("Design a comprehensive course outline for: " + subject)
        .call()
        .entity(CourseOutline.class);
}
```

---

## 7. CHAT OPTIONS & MODEL PARAMETERS

### Common Parameters
| Parameter | Description | Range |
|-----------|-------------|-------|
| `model` | Model name | `gpt-4o`, `claude-sonnet-4-20250514`, etc. |
| `temperature` | Randomness/creativity | 0.0 (deterministic) – 2.0 (creative) |
| `maxTokens` | Max response length | 1 – model max |
| `topP` | Nucleus sampling | 0.0 – 1.0 |
| `topK` | Top-K sampling | 1 – vocab size |
| `frequencyPenalty` | Penalize repeated tokens | -2.0 – 2.0 |
| `presencePenalty` | Penalize already-used tokens | -2.0 – 2.0 |
| `stop` | Stop sequences | list of strings |
| `seed` | Deterministic output | integer |

### Setting Options
```java
// Via application.yml
spring:
  ai:
    openai:
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
          max-tokens: 2000

// Via code — per request
chatClient.prompt()
    .user("Explain polymorphism")
    .options(OpenAiChatOptions.builder()
        .withModel("gpt-4o-mini")
        .withTemperature(0.3)
        .withMaxTokens(500)
        .withTopP(0.9)
        .build())
    .call()
    .content();

// Portable options (provider-agnostic)
chatClient.prompt()
    .options(ChatOptionsBuilder.builder()
        .withModel("gpt-4o")
        .withTemperature(0.5)
        .build())
    .call();
```

### Temperature Guide
```
0.0       → Deterministic, factual, consistent (code generation, data extraction)
0.1 - 0.3 → Focused, reliable (Q&A, summarization)
0.4 - 0.7 → Balanced (general conversation, explanations)
0.8 - 1.0 → Creative (stories, brainstorming)
1.0 - 2.0 → Very random (experimental, poetry)
```

---

## 8. MULTIMODAL (Vision, Audio)

### Image Input (Vision)
```java
// Send image to AI for analysis
public String analyzeImage(String imageUrl) {
    UserMessage userMessage = new UserMessage(
        "Describe what you see in this image in detail.",
        new Media(MimeTypeUtils.IMAGE_PNG, new URL(imageUrl))
    );

    ChatResponse response = chatModel.call(new Prompt(userMessage));
    return response.getResult().getOutput().getText();
}

// From file
public String analyzeLocalImage(Resource imageFile) {
    UserMessage userMessage = new UserMessage(
        "What's in this image?",
        new Media(MimeTypeUtils.IMAGE_JPEG, imageFile)
    );

    return chatModel.call(new Prompt(userMessage))
        .getResult().getOutput().getText();
}

// With ChatClient
public String describeImage(String imageUrl) {
    return chatClient.prompt()
        .user(u -> u.text("Describe this image.")
                     .media(MimeTypeUtils.IMAGE_PNG, new URL(imageUrl)))
        .call()
        .content();
}

// Multiple images
public String compareImages(String url1, String url2) {
    return chatClient.prompt()
        .user(u -> u.text("Compare these two images.")
                     .media(MimeTypeUtils.IMAGE_PNG, new URL(url1))
                     .media(MimeTypeUtils.IMAGE_PNG, new URL(url2)))
        .call()
        .content();
}
```

### Image Generation (DALL-E, etc.)
```java
@Autowired
private ImageModel imageModel;

public String generateImage(String description) {
    ImagePrompt prompt = new ImagePrompt(
        description,
        OpenAiImageOptions.builder()
            .withModel("dall-e-3")
            .withQuality("hd")
            .withN(1)                         // number of images
            .withHeight(1024)
            .withWidth(1024)
            .withResponseFormat("url")        // "url" or "b64_json"
            .build()
    );

    ImageResponse response = imageModel.call(prompt);
    return response.getResult().getOutput().getUrl();    // image URL
}
```

### Audio — Text-to-Speech (TTS)
```java
@Autowired
private OpenAiAudioSpeechModel speechModel;

public byte[] textToSpeech(String text) {
    OpenAiAudioSpeechOptions options = OpenAiAudioSpeechOptions.builder()
        .withModel("tts-1")
        .withVoice(OpenAiAudioApi.SpeechRequest.Voice.ALLOY)
        .withResponseFormat(OpenAiAudioApi.SpeechRequest.AudioResponseFormat.MP3)
        .withSpeed(1.0f)
        .build();

    SpeechPrompt prompt = new SpeechPrompt(text, options);
    SpeechResponse response = speechModel.call(prompt);
    return response.getResult().getOutput();    // audio bytes
}
```

### Audio — Speech-to-Text (Transcription)
```java
@Autowired
private OpenAiAudioTranscriptionModel transcriptionModel;

public String transcribe(Resource audioFile) {
    OpenAiAudioTranscriptionOptions options = OpenAiAudioTranscriptionOptions.builder()
        .withModel("whisper-1")
        .withLanguage("en")
        .withResponseFormat(OpenAiAudioApi.TranscriptResponseFormat.TEXT)
        .build();

    AudioTranscriptionPrompt prompt = new AudioTranscriptionPrompt(audioFile, options);
    AudioTranscriptionResponse response = transcriptionModel.call(prompt);
    return response.getResult().getOutput();
}
```

---

## 9. FUNCTION / TOOL CALLING

### What is Function Calling?
- AI model can **request to call functions** you define.
- Model doesn't execute code — it returns function name + arguments.
- Spring AI executes the function and sends the result back to the model.
- Enables AI to **access real-time data, APIs, databases, calculations**.

### How It Works
```
1. You register functions with the model
2. User asks a question
3. Model determines a function call is needed
4. Model returns function name + arguments (JSON)
5. Spring AI calls your function with those arguments
6. Function result is sent back to model
7. Model generates final response using the result
```

### Defining Functions
```java
// Method 1: @Bean with java.util.function.Function
@Configuration
public class AiFunctions {

    // Simple function
    @Bean
    @Description("Get the current weather for a given city")
    public Function<WeatherRequest, WeatherResponse> currentWeather() {
        return request -> {
            // Call real weather API here
            return new WeatherResponse(request.city(), 22.5, "Sunny");
        };
    }

    public record WeatherRequest(String city) {}
    public record WeatherResponse(String city, double temperature, String condition) {}

    // Another function
    @Bean
    @Description("Search for products by name and return results")
    public Function<ProductSearchRequest, List<Product>> searchProducts() {
        return request -> {
            return productService.search(request.query(), request.maxResults());
        };
    }

    public record ProductSearchRequest(String query, int maxResults) {}
}
```

### Using Functions with ChatClient
```java
// Register functions in the call
public String askWithTools(String question) {
    return chatClient.prompt()
        .user(question)
        .functions("currentWeather", "searchProducts")    // bean names
        .call()
        .content();
}

// Example:
// User: "What's the weather in Mumbai?"
// → Model calls currentWeather("Mumbai")
// → Gets WeatherResponse(Mumbai, 32.0, Sunny)
// → Responds: "The current weather in Mumbai is 32°C and sunny."
```

### Functions with ChatModel Directly
```java
OpenAiChatOptions options = OpenAiChatOptions.builder()
    .withModel("gpt-4o")
    .withFunctions(Set.of("currentWeather"))
    .build();

UserMessage userMessage = new UserMessage("What's the weather in Delhi?");
ChatResponse response = chatModel.call(new Prompt(List.of(userMessage), options));
```

### Method 2: FunctionCallback (More Control)
```java
@Bean
public FunctionCallback weatherFunction() {
    return FunctionCallback.builder()
        .function("getWeather", (WeatherRequest req) -> {
            return new WeatherResponse(req.city(), 25.0, "Cloudy");
        })
        .description("Get current weather for a city")
        .inputType(WeatherRequest.class)
        .build();
}
```

### Default Functions (Always Available)
```java
@Bean
ChatClient chatClient(ChatClient.Builder builder) {
    return builder
        .defaultSystem("You are a helpful assistant.")
        .defaultFunctions("currentWeather", "searchProducts")   // always available
        .build();
}
```

### Real-World Function Examples
```java
// Database lookup
@Bean
@Description("Find user information by user ID")
public Function<UserLookupRequest, UserInfo> findUser(UserRepository repo) {
    return req -> {
        User user = repo.findById(req.userId()).orElseThrow();
        return new UserInfo(user.getName(), user.getEmail(), user.getRole());
    };
}

// API call
@Bean
@Description("Get current stock price for a ticker symbol")
public Function<StockRequest, StockPrice> getStockPrice(StockApiClient client) {
    return req -> client.getPrice(req.ticker());
}

// Calculation
@Bean
@Description("Calculate compound interest given principal, rate, and years")
public Function<InterestRequest, InterestResult> calculateInterest() {
    return req -> {
        double amount = req.principal() * Math.pow(1 + req.rate() / 100, req.years());
        return new InterestResult(amount, amount - req.principal());
    };
}
```

---

## 10. ADVISORS (Middleware/Interceptors)

### What are Advisors?
- **Interceptors** that augment the prompt or response.
- Added to the ChatClient call chain.
- Built-in advisors: chat memory, RAG, logging, safety.
- Custom advisors for cross-cutting concerns.

### Advisor Types
| Advisor | Purpose |
|---------|---------|
| `MessageChatMemoryAdvisor` | Add conversation history to prompt |
| `PromptChatMemoryAdvisor` | Add history as system message |
| `VectorStoreChatMemoryAdvisor` | Store/retrieve history from vector store |
| `QuestionAnswerAdvisor` | RAG — retrieve documents + augment prompt |
| `SafeGuardAdvisor` | Content safety filtering |
| `SimpleLoggerAdvisor` | Log prompts and responses |

### Using Advisors
```java
@Bean
ChatClient chatClient(ChatClient.Builder builder,
                       ChatMemory chatMemory,
                       VectorStore vectorStore) {
    return builder
        .defaultSystem("You are a helpful assistant.")
        .defaultAdvisors(
            new MessageChatMemoryAdvisor(chatMemory),     // conversation memory
            new QuestionAnswerAdvisor(vectorStore),        // RAG
            new SimpleLoggerAdvisor()                     // logging
        )
        .build();
}
```

### Per-Request Advisors
```java
public String ask(String question, String conversationId) {
    return chatClient.prompt()
        .user(question)
        .advisors(a -> a
            .param(ChatMemory.CONVERSATION_ID, conversationId)
            .param(ChatMemory.CHAT_MEMORY_RETRIEVE_SIZE, 10))
        .call()
        .content();
}
```

### Custom Advisor
```java
public class TimingAdvisor implements CallAroundAdvisor {

    @Override
    public AdvisedResponse aroundCall(AdvisedRequest request, CallAroundAdvisorChain chain) {
        long start = System.currentTimeMillis();
        AdvisedResponse response = chain.nextAroundCall(request);
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("AI call took " + elapsed + "ms");
        return response;
    }

    @Override
    public String getName() {
        return "TimingAdvisor";
    }

    @Override
    public int getOrder() {
        return 0;    // execution order
    }
}

// Register
chatClient = builder
    .defaultAdvisors(new TimingAdvisor())
    .build();
```

---

## 11. CHAT MEMORY (Conversation History)

### Why Chat Memory?
- AI models are **stateless** — each call is independent.
- Chat memory stores conversation history and adds it to each prompt.
- Enables **multi-turn conversations**.

### In-Memory Chat Memory
```java
@Bean
ChatMemory chatMemory() {
    return new InMemoryChatMemory();
}

@Bean
ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory) {
    return builder
        .defaultSystem("You are a helpful assistant.")
        .defaultAdvisors(new MessageChatMemoryAdvisor(chatMemory))
        .build();
}
```

### Using with Conversation IDs
```java
@Service
public class ChatService {

    private final ChatClient chatClient;

    public ChatService(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    // Each user/session gets a unique conversationId
    public String chat(String conversationId, String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .advisors(a -> a
                .param(ChatMemory.CONVERSATION_ID, conversationId)
                .param(ChatMemory.CHAT_MEMORY_RETRIEVE_SIZE, 20))   // last N messages
            .call()
            .content();
    }
}

// REST Controller
@RestController
@RequestMapping("/chat")
public class ChatController {

    private final ChatService chatService;

    @PostMapping
    public String chat(@RequestParam String sessionId, @RequestBody String message) {
        return chatService.chat(sessionId, message);
    }
}
```

### Memory Advisor Types
```java
// MessageChatMemoryAdvisor — adds history as user/assistant messages (recommended)
new MessageChatMemoryAdvisor(chatMemory)

// PromptChatMemoryAdvisor — adds history as system message text
new PromptChatMemoryAdvisor(chatMemory)

// VectorStoreChatMemoryAdvisor — stores in vector store, retrieves relevant history
new VectorStoreChatMemoryAdvisor(vectorStore)
```

### Persistent Chat Memory (Database-backed)
```java
// Use JDBC-backed memory
@Bean
ChatMemory chatMemory(JdbcTemplate jdbcTemplate) {
    return new JdbcChatMemory(jdbcTemplate);
}

// Or implement ChatMemory interface for custom storage
public class RedisChatMemory implements ChatMemory {

    @Override
    public void add(String conversationId, List<Message> messages) {
        // store in Redis
    }

    @Override
    public List<Message> get(String conversationId, int lastN) {
        // retrieve from Redis
    }

    @Override
    public void clear(String conversationId) {
        // delete from Redis
    }
}
```

---

## 12. EMBEDDINGS

### What are Embeddings?
- **Vector representations** of text — numbers capturing semantic meaning.
- Similar texts → similar vectors (close in vector space).
- Used for: **semantic search, RAG, similarity matching, clustering**.
- Typical dimensions: 384, 768, 1536, 3072.

### EmbeddingModel Interface
```java
public interface EmbeddingModel extends Model<EmbeddingRequest, EmbeddingResponse> {
    EmbeddingResponse call(EmbeddingRequest request);
    float[] embed(String text);
    float[] embed(Document document);
    List<float[]> embed(List<String> texts);
}
```

### Using EmbeddingModel
```java
@Service
public class EmbeddingService {

    private final EmbeddingModel embeddingModel;

    public EmbeddingService(EmbeddingModel embeddingModel) {
        this.embeddingModel = embeddingModel;
    }

    // Single text
    public float[] getEmbedding(String text) {
        return embeddingModel.embed(text);
    }

    // Batch
    public List<float[]> getEmbeddings(List<String> texts) {
        return embeddingModel.embed(texts);
    }

    // Full response
    public EmbeddingResponse getFullResponse(String text) {
        return embeddingModel.call(
            new EmbeddingRequest(List.of(text), EmbeddingOptionsBuilder.builder().build())
        );
    }

    // Cosine similarity
    public double similarity(String text1, String text2) {
        float[] v1 = embeddingModel.embed(text1);
        float[] v2 = embeddingModel.embed(text2);
        return cosineSimilarity(v1, v2);
    }

    private double cosineSimilarity(float[] a, float[] b) {
        double dot = 0, normA = 0, normB = 0;
        for (int i = 0; i < a.length; i++) {
            dot += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        return dot / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

### Embedding Models Configuration
```yaml
# OpenAI
spring.ai.openai.embedding.options.model: text-embedding-3-small

# Ollama
spring.ai.ollama.embedding.options.model: nomic-embed-text

# Azure
spring.ai.azure.openai.embedding.options.deployment-name: text-embedding-ada-002
```

---

## 13. VECTOR STORES

### What is a Vector Store?
- **Database optimized** for storing and searching vector embeddings.
- Enables **similarity search** — find documents closest to a query.
- Foundation for **RAG (Retrieval Augmented Generation)**.

### Supported Vector Stores
| Vector Store | Type |
|-------------|------|
| `SimpleVectorStore` | In-memory (dev/testing) |
| `PgVectorStore` | PostgreSQL + pgvector |
| `ChromaVectorStore` | Chroma DB |
| `MilvusVectorStore` | Milvus |
| `PineconeVectorStore` | Pinecone (cloud) |
| `WeaviateVectorStore` | Weaviate |
| `QdrantVectorStore` | Qdrant |
| `RedisVectorStore` | Redis Stack |
| `Neo4jVectorStore` | Neo4j |
| `ElasticsearchVectorStore` | Elasticsearch |
| `MongoDBAtlasVectorStore` | MongoDB Atlas |

### VectorStore Interface
```java
public interface VectorStore {
    void add(List<Document> documents);
    Optional<Boolean> delete(List<String> idList);
    List<Document> similaritySearch(String query);
    List<Document> similaritySearch(SearchRequest request);
}
```

### SimpleVectorStore (In-Memory — Dev/Testing)
```java
@Bean
VectorStore vectorStore(EmbeddingModel embeddingModel) {
    return new SimpleVectorStore(embeddingModel);
}
```

### PgVector (PostgreSQL — Production)
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
</dependency>
```
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: postgres
    password: secret
  ai:
    vectorstore:
      pgvector:
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536
```

### Adding Documents
```java
@Service
public class DocumentService {

    private final VectorStore vectorStore;

    public DocumentService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public void addDocuments() {
        List<Document> documents = List.of(
            new Document("Spring AI provides a unified API for AI models.",
                Map.of("source", "docs", "chapter", "1")),

            new Document("Vector stores enable similarity search.",
                Map.of("source", "docs", "chapter", "2")),

            new Document("RAG combines retrieval with generation.",
                Map.of("source", "docs", "chapter", "3"))
        );

        vectorStore.add(documents);    // embeds + stores
    }
}
```

### Similarity Search
```java
// Simple search
List<Document> results = vectorStore.similaritySearch("How does RAG work?");

// With options
List<Document> results = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query("How does RAG work?")
        .topK(5)                                          // top 5 results
        .similarityThreshold(0.7)                         // min similarity (0.0-1.0)
        .filterExpression("source == 'docs' && chapter == '3'")  // metadata filter
        .build()
);

for (Document doc : results) {
    System.out.println("Content: " + doc.getText());
    System.out.println("Metadata: " + doc.getMetadata());
    System.out.println("Score: " + doc.getMetadata().get("distance"));
}
```

### Filter Expressions
```java
// Metadata filtering — SQL-like syntax
"author == 'Ritesh'"
"year > 2023"
"category IN ['java', 'spring']"
"author == 'Ritesh' && year >= 2024"
"NOT(category == 'deprecated')"
"country == 'India' || country == 'USA'"
```

---

## 14. DOCUMENT LOADING (ETL Pipeline)

### ETL Pipeline: Read → Transform → Write
```
Document Readers     →     Document Transformers     →     Document Writers
(PDF, HTML, JSON)          (Split, Enrich)                 (VectorStore)
```

### Document Readers
```java
// Text files
TextReader reader = new TextReader("classpath:data/article.txt");
List<Document> docs = reader.get();

// JSON
JsonReader reader = new JsonReader(
    new ClassPathResource("data/products.json"),
    "name", "description", "category"         // keys to extract
);

// PDF
// Requires: spring-ai-pdf-document-reader
PagePdfDocumentReader reader = new PagePdfDocumentReader("classpath:data/manual.pdf");
List<Document> docs = reader.get();

// Tika (Word, Excel, PowerPoint, HTML, etc.)
// Requires: spring-ai-tika-document-reader
TikaDocumentReader reader = new TikaDocumentReader("classpath:data/document.docx");
List<Document> docs = reader.get();

// HTML from URL
// Requires: spring-ai-jsoup-document-reader
JsoupDocumentReader reader = new JsoupDocumentReader("https://example.com/article");
```

### Document Transformers (Text Splitting)
```java
// TokenTextSplitter — split by token count
TokenTextSplitter splitter = new TokenTextSplitter(
    800,     // default chunk size (tokens)
    350,     // min chunk size
    5,       // min chars per chunk
    10000,   // max num chunks
    true     // keep separator
);
List<Document> chunks = splitter.apply(documents);

// TextSplitter with custom settings
TextSplitter splitter = new TokenTextSplitter();
List<Document> chunks = splitter.split(documents);
```

### Full ETL Pipeline
```java
@Component
public class DocumentIngestionService {

    private final VectorStore vectorStore;

    public DocumentIngestionService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public void ingestPdf(Resource pdfFile) {
        // 1. READ
        PagePdfDocumentReader reader = new PagePdfDocumentReader(pdfFile);
        List<Document> documents = reader.get();

        // 2. TRANSFORM (split into chunks)
        TokenTextSplitter splitter = new TokenTextSplitter();
        List<Document> chunks = splitter.apply(documents);

        // 3. WRITE (embed + store)
        vectorStore.add(chunks);
    }

    public void ingestTextFiles(List<Resource> files) {
        for (Resource file : files) {
            TextReader reader = new TextReader(file);
            reader.getCustomMetadata().put("source", file.getFilename());

            List<Document> docs = reader.get();
            List<Document> chunks = new TokenTextSplitter().apply(docs);
            vectorStore.add(chunks);
        }
    }
}
```

---

## 15. RAG (Retrieval Augmented Generation)

### What is RAG?
- **Retrieval Augmented Generation** — enhance AI with your private data.
- Instead of relying only on model's training data, **retrieve relevant documents** first.
- Prevents hallucinations, provides up-to-date information.

### RAG Flow
```
1. User asks a question
2. Question is embedded (converted to vector)
3. Similar documents retrieved from vector store
4. Retrieved documents added to the prompt as context
5. AI generates answer using the context
```

### QuestionAnswerAdvisor (Built-in RAG)
```java
@Bean
ChatClient chatClient(ChatClient.Builder builder, VectorStore vectorStore) {
    return builder
        .defaultSystem("Answer questions based on the provided context. "
            + "If you don't know the answer from the context, say so.")
        .defaultAdvisors(new QuestionAnswerAdvisor(vectorStore))
        .build();
}

// Usage — RAG happens automatically
public String askAboutDocs(String question) {
    return chatClient.prompt()
        .user(question)
        .call()
        .content();
    // Behind the scenes:
    // 1. Embeds the question
    // 2. Retrieves similar documents from vectorStore
    // 3. Adds them to the prompt as context
    // 4. AI answers using the context
}
```

### QuestionAnswerAdvisor with Options
```java
QuestionAnswerAdvisor ragAdvisor = new QuestionAnswerAdvisor(
    vectorStore,
    SearchRequest.builder()
        .topK(5)                      // retrieve top 5 documents
        .similarityThreshold(0.7)     // minimum relevance
        .build()
);

// Custom prompt template for RAG
QuestionAnswerAdvisor ragAdvisor = new QuestionAnswerAdvisor(
    vectorStore,
    SearchRequest.builder().topK(5).build(),
    """
    Context information:
    ---------------------
    {question_answer_context}
    ---------------------
    Given the context above, answer the following question.
    If the answer is not in the context, say "I don't have that information."

    Question: {question}
    Answer:
    """
);
```

### Full RAG Application Example
```java
@Configuration
public class RagConfig {

    @Bean
    VectorStore vectorStore(EmbeddingModel embeddingModel) {
        return new SimpleVectorStore(embeddingModel);
    }

    @Bean
    ChatClient chatClient(ChatClient.Builder builder, VectorStore vectorStore) {
        return builder
            .defaultSystem("You are a documentation assistant. "
                + "Answer only from the provided context.")
            .defaultAdvisors(
                new QuestionAnswerAdvisor(vectorStore,
                    SearchRequest.builder().topK(5).similarityThreshold(0.7).build()),
                new MessageChatMemoryAdvisor(new InMemoryChatMemory())
            )
            .build();
    }
}

@Service
public class DocsService {

    private final VectorStore vectorStore;
    private final ChatClient chatClient;

    public DocsService(VectorStore vectorStore, ChatClient chatClient) {
        this.vectorStore = vectorStore;
        this.chatClient = chatClient;
    }

    // Ingest documents
    public void loadDocs(Resource pdfFile) {
        List<Document> docs = new PagePdfDocumentReader(pdfFile).get();
        List<Document> chunks = new TokenTextSplitter().apply(docs);
        vectorStore.add(chunks);
    }

    // Ask questions about docs
    public String ask(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}

@RestController
@RequestMapping("/api/docs")
public class DocsController {

    private final DocsService docsService;

    @PostMapping("/upload")
    public String upload(@RequestParam MultipartFile file) throws IOException {
        docsService.loadDocs(file.getResource());
        return "Documents ingested successfully";
    }

    @GetMapping("/ask")
    public String ask(@RequestParam String question) {
        return docsService.ask(question);
    }
}
```

---

## 16. MODEL EVALUATION

### What is Evaluation?
- Measure how **accurate** and **relevant** AI responses are.
- Spring AI provides built-in evaluators.

### RelevancyEvaluator
```java
@Service
public class EvalService {

    private final ChatClient chatClient;
    private final RelevancyEvaluator evaluator;

    public EvalService(ChatClient chatClient, ChatClient.Builder builder) {
        this.chatClient = chatClient;
        this.evaluator = new RelevancyEvaluator(builder);
    }

    public boolean isRelevant(String question, String answer, List<Document> context) {
        EvaluationRequest request = new EvaluationRequest(
            question,
            context,
            answer
        );
        EvaluationResponse response = evaluator.evaluate(request);
        return response.isPass();
    }
}
```

---

## 17. ERROR HANDLING & RETRY

### Retry Configuration
```yaml
# Auto-configured retry for transient failures
spring:
  ai:
    retry:
      max-attempts: 3
      backoff:
        initial-interval: 2000       # ms
        multiplier: 2
        max-interval: 30000          # ms
      on-client-errors: false        # retry on 4xx?
      on-http-codes: 429, 500, 503  # retry on these codes
```

### Exception Handling
```java
@Service
public class AiService {

    private final ChatClient chatClient;

    public String safAsk(String question) {
        try {
            return chatClient.prompt()
                .user(question)
                .call()
                .content();
        } catch (NonTransientAiException e) {
            // 4xx errors — bad request, auth failure
            throw new AiServiceException("Invalid request: " + e.getMessage(), e);
        } catch (TransientAiException e) {
            // 5xx, rate limits — retries exhausted
            throw new AiServiceException("AI service unavailable", e);
        }
    }
}

// Global exception handler
@RestControllerAdvice
public class AiExceptionHandler {

    @ExceptionHandler(NonTransientAiException.class)
    public ResponseEntity<String> handleAiError(NonTransientAiException e) {
        return ResponseEntity.status(500).body("AI processing error: " + e.getMessage());
    }

    @ExceptionHandler(TransientAiException.class)
    public ResponseEntity<String> handleTransientError(TransientAiException e) {
        return ResponseEntity.status(503).body("AI service temporarily unavailable");
    }
}
```

---

## 18. OBSERVABILITY & MONITORING

### Micrometer Integration
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing</artifactId>
</dependency>
```

```yaml
# Enable AI observability
management:
  tracing:
    enabled: true
  observations:
    ai:
      enabled: true
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus

spring:
  ai:
    chat:
      observations:
        include-prompt: true          # log prompts (careful in production!)
        include-completion: true      # log completions
```

### Logged Metrics
```
spring.ai.chat.model.call         → chat call duration, count
spring.ai.chat.model.tokens       → token usage (prompt, completion)
spring.ai.embedding.model.call    → embedding call duration
spring.ai.image.model.call        → image generation metrics
spring.ai.vectorstore.add         → document addition
spring.ai.vectorstore.search      → similarity search
```

### SimpleLoggerAdvisor
```java
// Log all prompts and responses
chatClient = builder
    .defaultAdvisors(new SimpleLoggerAdvisor())
    .build();

// Set log level
// application.yml:
logging:
  level:
    org.springframework.ai.chat.client.advisor: DEBUG
```

---

## 19. MULTIPLE MODELS & PROVIDERS

### Using Multiple Providers
```java
// When multiple providers are configured, qualify which to use

@Configuration
public class MultiModelConfig {

    @Bean("openaiClient")
    ChatClient openaiClient(
            @Qualifier("openAiChatModel") ChatModel openAiModel) {
        return ChatClient.builder(openAiModel)
            .defaultSystem("You are a general assistant.")
            .build();
    }

    @Bean("claudeClient")
    ChatClient claudeClient(
            @Qualifier("anthropicChatModel") ChatModel claudeModel) {
        return ChatClient.builder(claudeModel)
            .defaultSystem("You are a code expert.")
            .build();
    }
}

@Service
public class SmartAiService {

    @Qualifier("openaiClient")
    private final ChatClient openai;

    @Qualifier("claudeClient")
    private final ChatClient claude;

    public SmartAiService(
            @Qualifier("openaiClient") ChatClient openai,
            @Qualifier("claudeClient") ChatClient claude) {
        this.openai = openai;
        this.claude = claude;
    }

    // Route to best model based on task
    public String ask(String question, String taskType) {
        return switch (taskType) {
            case "code" -> claude.prompt().user(question).call().content();
            case "general" -> openai.prompt().user(question).call().content();
            default -> openai.prompt().user(question).call().content();
        };
    }
}
```

### Fallback Pattern
```java
public String askWithFallback(String question) {
    try {
        return openai.prompt().user(question).call().content();
    } catch (Exception e) {
        // Fallback to another provider
        return claude.prompt().user(question).call().content();
    }
}
```

---

## 20. OLLAMA (LOCAL MODELS)

### What is Ollama?
- Run **open-source AI models locally** — no API key, no cost, full privacy.
- Models: Llama 3, Mistral, Gemma, CodeLlama, Phi, DeepSeek, etc.

### Setup Ollama
```bash
# Install Ollama
# macOS/Linux
curl -fsSL https://ollama.com/install.sh | sh

# Windows — download from https://ollama.com/download

# Pull a model
ollama pull llama3
ollama pull mistral
ollama pull nomic-embed-text    # for embeddings
ollama pull codellama            # for code

# List models
ollama list

# Run (starts server on localhost:11434)
ollama serve
```

### Spring AI + Ollama
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
</dependency>
```

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3
          temperature: 0.7
          num-predict: 2000        # max tokens
      embedding:
        options:
          model: nomic-embed-text
```

```java
@Service
public class LocalAiService {

    private final ChatClient chatClient;

    public LocalAiService(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are a helpful coding assistant.")
            .build();
    }

    public String ask(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
// Same code works for OpenAI, Ollama, Claude — just change the starter + config
```

---

## 21. COMMON PATTERNS & RECIPES

### REST Controller for Chat
```java
@RestController
@RequestMapping("/api/ai")
public class AiController {

    private final ChatClient chatClient;

    public AiController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    // Simple Q&A
    @GetMapping("/ask")
    public String ask(@RequestParam String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }

    // Streaming
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> stream(@RequestParam String question) {
        return chatClient.prompt()
            .user(question)
            .stream()
            .content();
    }

    // Structured output
    @GetMapping("/recommend")
    public BookRecommendation recommend(@RequestParam String topic) {
        return chatClient.prompt()
            .user("Recommend a book about " + topic)
            .call()
            .entity(BookRecommendation.class);
    }

    // Chat with memory
    @PostMapping("/chat")
    public String chat(@RequestParam String sessionId, @RequestBody String message) {
        return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, sessionId))
            .call()
            .content();
    }
}
```

### Code Review Assistant
```java
@Service
public class CodeReviewService {

    private final ChatClient chatClient;

    public CodeReviewService(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("""
                You are a senior code reviewer.
                Review the given code for:
                1. Bugs and potential issues
                2. Security vulnerabilities
                3. Performance improvements
                4. Code style and best practices
                Be concise and actionable.
                """)
            .build();
    }

    public record CodeReview(
        List<Issue> issues,
        String overallRating,
        String summary
    ) {
        public record Issue(String type, String severity, String description, String suggestion) {}
    }

    public CodeReview review(String code, String language) {
        return chatClient.prompt()
            .user(u -> u.text("Review this {language} code:\n```\n{code}\n```")
                         .param("language", language)
                         .param("code", code))
            .call()
            .entity(CodeReview.class);
    }
}
```

### Summarization Service
```java
@Service
public class SummarizationService {

    private final ChatClient chatClient;

    public SummarizationService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public String summarize(String text, int maxSentences) {
        return chatClient.prompt()
            .system("Summarize the following text in exactly " + maxSentences + " sentences.")
            .user(text)
            .options(ChatOptionsBuilder.builder()
                .withTemperature(0.2)
                .build())
            .call()
            .content();
    }

    public record Summary(
        String title,
        String summary,
        List<String> keyPoints,
        String sentiment
    ) {}

    public Summary structuredSummary(String text) {
        return chatClient.prompt()
            .system("Analyze and summarize the provided text.")
            .user(text)
            .call()
            .entity(Summary.class);
    }
}
```

### Chatbot with RAG + Memory + Functions
```java
@Configuration
public class ChatbotConfig {

    @Bean
    ChatClient chatbot(ChatClient.Builder builder,
                       VectorStore vectorStore,
                       ChatMemory chatMemory) {
        return builder
            .defaultSystem("""
                You are a product support assistant for TechCorp.
                Use the provided context to answer questions.
                If asked about orders, use the lookupOrder function.
                If you don't know, say "I'll connect you to a human agent."
                """)
            .defaultAdvisors(
                new MessageChatMemoryAdvisor(chatMemory),
                new QuestionAnswerAdvisor(vectorStore,
                    SearchRequest.builder().topK(5).build())
            )
            .defaultFunctions("lookupOrder", "getProductInfo")
            .build();
    }

    @Bean
    ChatMemory chatMemory() {
        return new InMemoryChatMemory();
    }

    @Bean
    @Description("Look up order status by order ID")
    Function<OrderRequest, OrderInfo> lookupOrder(OrderService orderService) {
        return req -> orderService.getOrderInfo(req.orderId());
    }

    public record OrderRequest(String orderId) {}
    public record OrderInfo(String orderId, String status, String estimatedDelivery) {}
}
```

---

## 22. TESTING SPRING AI

### Unit Testing (Mock ChatClient)
```java
@ExtendWith(MockitoExtension.class)
class AiServiceTest {

    @Mock
    private ChatModel chatModel;

    @Test
    void shouldReturnResponse() {
        // Arrange
        AssistantMessage assistantMsg = new AssistantMessage("Spring AI is great!");
        Generation generation = new Generation(assistantMsg);
        ChatResponse response = new ChatResponse(List.of(generation));

        when(chatModel.call(any(Prompt.class))).thenReturn(response);

        // Act
        AiService service = new AiService(chatModel);
        String result = service.ask("What is Spring AI?");

        // Assert
        assertEquals("Spring AI is great!", result);
        verify(chatModel).call(any(Prompt.class));
    }
}
```

### Integration Testing with Testcontainers + Ollama
```java
@SpringBootTest
@Testcontainers
class AiIntegrationTest {

    @Container
    static OllamaContainer ollama = new OllamaContainer("ollama/ollama:latest")
        .withModel("llama3");

    @DynamicPropertySource
    static void configureOllama(DynamicPropertyRegistry registry) {
        registry.add("spring.ai.ollama.base-url", ollama::getEndpoint);
        registry.add("spring.ai.ollama.chat.options.model", () -> "llama3");
    }

    @Autowired
    private ChatClient chatClient;

    @Test
    void shouldRespondToQuestion() {
        String response = chatClient.prompt()
            .user("What is 2 + 2?")
            .call()
            .content();

        assertNotNull(response);
        assertTrue(response.contains("4"));
    }
}
```

### Testing with Profiles
```yaml
# application-test.yml — use cheap/fast model for tests
spring:
  ai:
    openai:
      chat:
        options:
          model: gpt-4o-mini         # cheaper model for tests
          temperature: 0.0           # deterministic
          max-tokens: 100            # limit costs
```

```java
@SpringBootTest
@ActiveProfiles("test")
class AiServiceTest { ... }
```

---

## 23. BEST PRACTICES

### API Key Security
```
1. NEVER hardcode API keys in source code
2. Use environment variables: ${OPENAI_API_KEY}
3. Use Spring Vault or AWS Secrets Manager in production
4. Add application-*.yml to .gitignore if it contains keys
5. Use different API keys for dev/staging/prod
```

### Cost Optimization
```
1. Use smaller models for simple tasks (gpt-4o-mini vs gpt-4o)
2. Set max_tokens to limit response length
3. Cache frequent responses (Spring @Cacheable)
4. Use embeddings only when needed (expensive for large datasets)
5. Batch embedding requests when possible
6. Use Ollama for development/testing (free, local)
7. Monitor token usage via actuator metrics
8. Use temperature 0.0 for deterministic caching
```

### Prompt Engineering
```
1. Be specific and clear in system messages
2. Use role/persona in system prompts
3. Provide examples (few-shot learning)
4. Use structured output for reliable parsing
5. Keep prompts in resource files, not Java code
6. Version control your prompts
7. Test prompts systematically
```

### Architecture
```
1. Use ChatClient (not ChatModel directly)
2. Keep AI logic in @Service layer (not controllers)
3. Use advisors for cross-cutting concerns (logging, memory, RAG)
4. Design for provider portability (avoid provider-specific APIs)
5. Use @Qualifier when multiple models configured
6. Implement retry and fallback strategies
7. Use streaming for long responses
8. Make AI calls async when possible
```

### RAG Best Practices
```
1. Chunk documents appropriately (not too large, not too small)
2. Use metadata for filtering (source, date, category)
3. Set similarity threshold to filter low-relevance results
4. Include "I don't know" instructions in system prompt
5. Monitor retrieval quality (are relevant docs found?)
6. Re-index when source documents change
7. Use different embedding models for different domains
```

---

## 24. QUICK REFERENCE CHEAT SHEET

### Dependencies
```xml
<!-- Pick ONE provider -->
spring-ai-openai-spring-boot-starter
spring-ai-anthropic-spring-boot-starter
spring-ai-ollama-spring-boot-starter
spring-ai-azure-openai-spring-boot-starter

<!-- Vector Store (pick one) -->
spring-ai-pgvector-store-spring-boot-starter
spring-ai-chroma-store-spring-boot-starter

<!-- Document Reading -->
spring-ai-pdf-document-reader
spring-ai-tika-document-reader
```

### Minimal App
```java
@SpringBootApplication
public class App { public static void main(String[] a) { SpringApplication.run(App.class, a); } }

@RestController
class AiController {
    private final ChatClient chatClient;
    AiController(ChatClient.Builder b) { this.chatClient = b.build(); }

    @GetMapping("/ask")
    String ask(@RequestParam String q) {
        return chatClient.prompt().user(q).call().content();
    }
}
```
```yaml
spring.ai.openai.api-key: ${OPENAI_API_KEY}
```

### ChatClient Patterns
```java
// Simple call
chatClient.prompt().user("question").call().content();

// With system message
chatClient.prompt().system("persona").user("question").call().content();

// Streaming
chatClient.prompt().user("question").stream().content();    // Flux<String>

// Structured output
chatClient.prompt().user("question").call().entity(MyClass.class);

// With functions
chatClient.prompt().user("question").functions("func1").call().content();

// With options
chatClient.prompt().user("q").options(ChatOptionsBuilder.builder()
    .withModel("gpt-4o").withTemperature(0.5).build()).call().content();

// With advisors (memory, RAG)
chatClient.prompt().user("q")
    .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, "session1"))
    .call().content();
```

### Key Interfaces
```
ChatModel          → chatModel.call(prompt) → ChatResponse
EmbeddingModel     → embeddingModel.embed("text") → float[]
VectorStore        → vectorStore.add(docs) / similaritySearch("query")
ImageModel         → imageModel.call(imagePrompt) → ImageResponse
ChatClient         → fluent API wrapping ChatModel (recommended)
ChatMemory         → conversation history storage
```

### RAG in 3 Steps
```java
// 1. LOAD documents
vectorStore.add(new TokenTextSplitter().apply(reader.get()));

// 2. CONFIGURE advisor
new QuestionAnswerAdvisor(vectorStore, SearchRequest.builder().topK(5).build())

// 3. ASK questions
chatClient.prompt().user(question).call().content();
```

---

*Complete Spring AI Reference — All 24 Topics*
*Last updated: April 2026*
