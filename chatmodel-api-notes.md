# Chat Model API Notes — OpenAI, Google Gemini, Anthropic Claude

---

## Table of Contents

1. [OpenAI Chat API](#section-1-openai-chat-api)
2. [Google Gemini Chat API](#section-2-google-gemini-chat-api)
3. [Anthropic Claude Chat API](#section-3-anthropic-claude-chat-api)
4. [Tool Calling / Function Calling (All 3 Providers)](#section-4-tool-calling--function-calling-all-3-providers)
5. [Vision / Image Input (All 3 Providers)](#section-5-vision--image-input-all-3-providers)
6. [Streaming (All 3 Providers)](#section-6-streaming-all-3-providers)
7. [Structured Output / JSON Mode](#section-7-structured-output--json-mode)
8. [Multiple / Parallel Tool Calls](#section-8-multiple--parallel-tool-calls)

---

# Section 1: OpenAI Chat API

## Base URL

```
https://api.openai.com/v1/chat/completions
```

## Authentication

- Header: `Authorization: Bearer YOUR_API_KEY`
- Get your key from: https://platform.openai.com/api-keys

---

## Request (POST)

### Headers

| Header         | Value                  |
| -------------- | ---------------------- |
| Content-Type   | application/json       |
| Authorization  | Bearer YOUR_API_KEY    |

### Request Body (JSON)

```json
{
  "model": "gpt-4o",
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "What is Java?" }
  ],
  "temperature": 0.7,
  "max_tokens": 500,
  "top_p": 1,
  "frequency_penalty": 0,
  "presence_penalty": 0,
  "stop": ["\n"],
  "n": 1,
  "stream": false
}
```

### Parameters Explained

| Parameter            | Type     | Required | Default | Description                                                                                             |
| -------------------- | -------- | -------- | ------- | ------------------------------------------------------------------------------------------------------- |
| `model`              | string   | Yes      | —       | Model ID to use. Examples: `gpt-4o`, `gpt-4o-mini`, `gpt-4-turbo`, `gpt-3.5-turbo`                    |
| `messages`           | array    | Yes      | —       | Array of message objects. Each has `role` and `content`. This is the conversation history.               |
| `messages[].role`    | string   | Yes      | —       | `system` = sets behavior, `user` = your prompt, `assistant` = model's previous reply                   |
| `messages[].content` | string   | Yes      | —       | The actual text of the message                                                                          |
| `temperature`        | number   | No       | 1       | Randomness. `0` = deterministic, `2` = very random. Lower = focused, Higher = creative                 |
| `max_tokens`         | integer  | No       | model's max | Max tokens in the response. 1 token ≈ 4 characters in English                                      |
| `top_p`              | number   | No       | 1       | Nucleus sampling. `0.1` = only top 10% probable tokens. Alternative to temperature (don't use both)    |
| `frequency_penalty`  | number   | No       | 0       | `-2.0` to `2.0`. Positive = penalize repeated tokens. Reduces word repetition                           |
| `presence_penalty`   | number   | No       | 0       | `-2.0` to `2.0`. Positive = penalize tokens that appeared at all. Encourages new topics                |
| `stop`               | string/array | No   | null    | Up to 4 sequences where the model will stop generating. Example: `["\n", "END"]`                        |
| `n`                  | integer  | No       | 1       | How many response choices to generate. `n=3` gives 3 different answers                                  |
| `stream`             | boolean  | No       | false   | `true` = response comes as Server-Sent Events (SSE) chunk by chunk (like typing effect)                 |
| `response_format`    | object   | No       | —       | Force JSON output: `{ "type": "json_object" }`. Model will return valid JSON                            |
| `seed`               | integer  | No       | —       | For reproducible outputs. Same seed + same params = same result (mostly)                                |
| `tools`              | array    | No       | —       | Function calling. Define functions the model can call. Model returns function name + args               |
| `tool_choice`        | string/object | No  | auto    | `auto` = model decides, `none` = no tools, `required` = must call a tool                               |
| `logprobs`           | boolean  | No       | false   | Return log probabilities of output tokens                                                               |
| `user`               | string   | No       | —       | Unique user ID for abuse monitoring                                                                     |

---

### Response Body

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1713100000,
  "model": "gpt-4o",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Java is a high-level, object-oriented programming language..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 20,
    "completion_tokens": 85,
    "total_tokens": 105
  }
}
```

### Response Fields Explained

| Field                          | Description                                                                 |
| ------------------------------ | --------------------------------------------------------------------------- |
| `id`                           | Unique ID for this completion                                               |
| `object`                       | Always `chat.completion`                                                    |
| `created`                      | Unix timestamp when the response was generated                              |
| `model`                        | The model that was actually used                                            |
| `choices`                      | Array of responses (length = `n` from request)                              |
| `choices[].index`              | Index of this choice (0, 1, 2...)                                           |
| `choices[].message.role`       | Always `assistant`                                                          |
| `choices[].message.content`    | The actual generated text                                                   |
| `choices[].finish_reason`      | `stop` = natural end, `length` = hit max_tokens, `tool_calls` = tool used  |
| `usage.prompt_tokens`          | Tokens used in the input                                                    |
| `usage.completion_tokens`      | Tokens used in the output                                                   |
| `usage.total_tokens`           | Total tokens consumed (you're billed on this)                               |

---

### Postman Setup (OpenAI)

```
Method: POST
URL:    https://api.openai.com/v1/chat/completions

Headers:
  Content-Type: application/json
  Authorization: Bearer sk-xxxxxxxxxxxxxxxxxxxxxxxx

Body (raw JSON):
{
  "model": "gpt-4o-mini",
  "messages": [
    { "role": "user", "content": "Explain REST API in 2 lines" }
  ],
  "temperature": 0.5,
  "max_tokens": 100
}
```

### Common Models

| Model           | Context Window | Notes                                |
| --------------- | -------------- | ------------------------------------ |
| `gpt-4o`        | 128K tokens    | Latest flagship, fast + smart        |
| `gpt-4o-mini`   | 128K tokens    | Cheap + fast, good for simple tasks  |
| `gpt-4-turbo`   | 128K tokens    | Older but still powerful             |
| `gpt-3.5-turbo` | 16K tokens     | Budget model, fastest                |

---
---

# Section 2: Google Gemini Chat API

## Base URL

```
https://generativelanguage.googleapis.com/v1beta/models/{model}:generateContent?key=YOUR_API_KEY
```

Example:
```
https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=AIza...
```

## Authentication

- Query parameter: `?key=YOUR_API_KEY`
- Get your key from: https://aistudio.google.com/apikey

---

## Request (POST)

### Headers

| Header       | Value            |
| ------------ | ---------------- |
| Content-Type | application/json |

### Request Body (JSON)

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [
        { "text": "What is Java?" }
      ]
    }
  ],
  "systemInstruction": {
    "parts": [
      { "text": "You are a helpful assistant." }
    ]
  },
  "generationConfig": {
    "temperature": 0.7,
    "maxOutputTokens": 500,
    "topP": 0.95,
    "topK": 40,
    "candidateCount": 1,
    "stopSequences": ["END"],
    "responseMimeType": "text/plain"
  },
  "safetySettings": [
    {
      "category": "HARM_CATEGORY_HARASSMENT",
      "threshold": "BLOCK_MEDIUM_AND_ABOVE"
    }
  ]
}
```

### Parameters Explained

| Parameter                              | Type     | Required | Default     | Description                                                                                   |
| -------------------------------------- | -------- | -------- | ----------- | --------------------------------------------------------------------------------------------- |
| `contents`                             | array    | Yes      | —           | The conversation messages. Array of content objects                                            |
| `contents[].role`                      | string   | Yes      | —           | `user` = your message, `model` = previous AI reply (note: NOT "assistant" like OpenAI)        |
| `contents[].parts`                     | array    | Yes      | —           | Array of parts. Each part can be `text`, `inlineData` (image), etc.                           |
| `contents[].parts[].text`             | string   | Yes*     | —           | The actual text content of this part                                                          |
| `systemInstruction`                    | object   | No       | —           | System-level instruction (like OpenAI's system role). Sets model behavior                     |
| `generationConfig`                     | object   | No       | —           | Controls how the model generates output                                                       |
| `generationConfig.temperature`         | number   | No       | 1.0         | Randomness. `0` = deterministic, `2` = very creative                                          |
| `generationConfig.maxOutputTokens`     | integer  | No       | model's max | Max tokens in the response                                                                    |
| `generationConfig.topP`               | number   | No       | 0.95        | Nucleus sampling. Similar to OpenAI's top_p                                                   |
| `generationConfig.topK`               | integer  | No       | 40          | Only sample from the top K most likely tokens. Lower = more focused (Gemini-specific)         |
| `generationConfig.candidateCount`     | integer  | No       | 1           | Number of response candidates to return (like OpenAI's `n`)                                   |
| `generationConfig.stopSequences`      | array    | No       | —           | Up to 5 strings. Model stops when it generates any of these                                   |
| `generationConfig.responseMimeType`   | string   | No       | text/plain  | `text/plain` or `application/json` (forces JSON output)                                       |
| `generationConfig.responseSchema`     | object   | No       | —           | JSON Schema to enforce structured JSON output (use with `application/json` mime type)          |
| `safetySettings`                       | array    | No       | —           | Configure content filtering thresholds                                                        |
| `safetySettings[].category`           | string   | No       | —           | `HARM_CATEGORY_HARASSMENT`, `HARM_CATEGORY_HATE_SPEECH`, `HARM_CATEGORY_SEXUALLY_EXPLICIT`, `HARM_CATEGORY_DANGEROUS_CONTENT` |
| `safetySettings[].threshold`          | string   | No       | —           | `BLOCK_NONE`, `BLOCK_LOW_AND_ABOVE`, `BLOCK_MEDIUM_AND_ABOVE`, `BLOCK_ONLY_HIGH`             |
| `tools`                               | array    | No       | —           | Function declarations for function calling                                                     |

---

### Response Body

```json
{
  "candidates": [
    {
      "content": {
        "parts": [
          { "text": "Java is a high-level, object-oriented programming language..." }
        ],
        "role": "model"
      },
      "finishReason": "STOP",
      "index": 0,
      "safetyRatings": [
        {
          "category": "HARM_CATEGORY_HARASSMENT",
          "probability": "NEGLIGIBLE"
        }
      ]
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 10,
    "candidatesTokenCount": 85,
    "totalTokenCount": 95
  },
  "modelVersion": "gemini-2.0-flash"
}
```

### Response Fields Explained

| Field                                    | Description                                                          |
| ---------------------------------------- | -------------------------------------------------------------------- |
| `candidates`                             | Array of generated responses                                         |
| `candidates[].content.parts[].text`      | The actual generated text                                            |
| `candidates[].content.role`              | Always `model`                                                       |
| `candidates[].finishReason`              | `STOP` = done, `MAX_TOKENS` = hit limit, `SAFETY` = blocked          |
| `candidates[].safetyRatings`             | Safety scores for each category                                      |
| `usageMetadata.promptTokenCount`         | Input tokens used                                                    |
| `usageMetadata.candidatesTokenCount`     | Output tokens used                                                   |
| `usageMetadata.totalTokenCount`          | Total tokens consumed                                                |
| `modelVersion`                           | Actual model version used                                            |

---

### Postman Setup (Gemini)

```
Method: POST
URL:    https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=AIzaSy...

Headers:
  Content-Type: application/json

Body (raw JSON):
{
  "contents": [
    {
      "role": "user",
      "parts": [{ "text": "Explain REST API in 2 lines" }]
    }
  ],
  "generationConfig": {
    "temperature": 0.5,
    "maxOutputTokens": 100
  }
}
```

### Common Models

| Model                  | Context Window | Notes                                    |
| ---------------------- | -------------- | ---------------------------------------- |
| `gemini-2.5-pro`       | 1M tokens      | Most capable, for complex tasks          |
| `gemini-2.5-flash`     | 1M tokens      | Fast + thinking, good balance            |
| `gemini-2.0-flash`     | 1M tokens      | Fast, everyday tasks                     |
| `gemini-2.0-flash-lite`| 1M tokens      | Cheapest, high throughput                |

### Key Differences from OpenAI

- API key goes in **query param** (`?key=`), not in the header
- Role is `model` instead of `assistant`
- Messages use `contents[].parts[].text` instead of `messages[].content`
- System prompt is separate field `systemInstruction`, not a role in messages
- Has `topK` parameter (OpenAI doesn't)
- Safety settings are built-in and configurable

---
---

# Section 3: Anthropic Claude Chat API

## Base URL

```
https://api.anthropic.com/v1/messages
```

## Authentication

- Header: `x-api-key: YOUR_API_KEY`
- Header: `anthropic-version: 2023-06-01` (required, specifies the API version)
- Get your key from: https://console.anthropic.com/settings/keys

---

## Request (POST)

### Headers

| Header             | Value              |
| ------------------ | ------------------ |
| Content-Type       | application/json   |
| x-api-key          | YOUR_API_KEY       |
| anthropic-version  | 2023-06-01         |

### Request Body (JSON)

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 500,
  "system": "You are a helpful assistant.",
  "messages": [
    { "role": "user", "content": "What is Java?" }
  ],
  "temperature": 0.7,
  "top_p": 0.9,
  "top_k": 40,
  "stop_sequences": ["\n\n"],
  "stream": false
}
```

### Parameters Explained

| Parameter            | Type         | Required | Default     | Description                                                                                      |
| -------------------- | ------------ | -------- | ----------- | ------------------------------------------------------------------------------------------------ |
| `model`              | string       | Yes      | —           | Model ID. Examples: `claude-sonnet-4-20250514`, `claude-haiku-4-20250514`, `claude-opus-4-20250514` |
| `max_tokens`         | integer      | **Yes**  | —           | **Required in Claude!** Max tokens to generate. Must always be specified                         |
| `messages`           | array        | Yes      | —           | Conversation messages. Same `role`/`content` format as OpenAI                                    |
| `messages[].role`    | string       | Yes      | —           | `user` or `assistant` only (no system role in messages — it's a separate field)                   |
| `messages[].content` | string/array | Yes      | —           | Text string or array of content blocks `[{ "type": "text", "text": "..." }]`                     |
| `system`             | string/array | No       | —           | System prompt. Goes as a **top-level field**, NOT inside messages array                           |
| `temperature`        | number       | No       | 1.0         | Randomness. `0` = deterministic, `1` = max. Range: `0` to `1` (NOT 0-2 like OpenAI)             |
| `top_p`              | number       | No       | 0.999       | Nucleus sampling. Use either temperature OR top_p, not both                                       |
| `top_k`              | integer      | No       | —           | Only sample from top K tokens. Like Gemini's topK                                                |
| `stop_sequences`     | array        | No       | —           | Array of strings. Model stops when it generates any of these                                      |
| `stream`             | boolean      | No       | false       | `true` = SSE streaming response                                                                  |
| `tools`              | array        | No       | —           | Function/tool definitions for tool use (function calling)                                         |
| `tool_choice`        | object       | No       | auto        | `{"type": "auto"}`, `{"type": "any"}`, `{"type": "tool", "name": "func_name"}`                   |
| `metadata`           | object       | No       | —           | `{"user_id": "user123"}` — for abuse tracking                                                    |

---

### Response Body

```json
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "model": "claude-sonnet-4-20250514",
  "content": [
    {
      "type": "text",
      "text": "Java is a high-level, object-oriented programming language..."
    }
  ],
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 20,
    "output_tokens": 85,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 0
  }
}
```

### Response Fields Explained

| Field                              | Description                                                                     |
| ---------------------------------- | ------------------------------------------------------------------------------- |
| `id`                               | Unique message ID (starts with `msg_`)                                          |
| `type`                             | Always `message`                                                                |
| `role`                             | Always `assistant`                                                              |
| `model`                            | The model that was used                                                         |
| `content`                          | Array of content blocks (NOT a plain string like OpenAI)                        |
| `content[].type`                   | `text` for normal text, `tool_use` for tool calls                               |
| `content[].text`                   | The actual generated text                                                       |
| `stop_reason`                      | `end_turn` = done, `max_tokens` = hit limit, `stop_sequence` = hit stop string, `tool_use` = tool called |
| `stop_sequence`                    | Which stop sequence was hit (null if none)                                      |
| `usage.input_tokens`              | Input tokens consumed                                                            |
| `usage.output_tokens`             | Output tokens consumed                                                           |
| `usage.cache_creation_input_tokens`| Tokens written to prompt cache (if caching enabled)                             |
| `usage.cache_read_input_tokens`   | Tokens read from prompt cache (cheaper)                                          |

---

### Postman Setup (Claude)

```
Method: POST
URL:    https://api.anthropic.com/v1/messages

Headers:
  Content-Type:      application/json
  x-api-key:         sk-ant-xxxxxxxxxxxxxxxxxxxxxxxx
  anthropic-version:  2023-06-01

Body (raw JSON):
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 100,
  "messages": [
    { "role": "user", "content": "Explain REST API in 2 lines" }
  ],
  "temperature": 0.5
}
```

### Common Models

| Model                            | Context Window | Notes                                    |
| -------------------------------- | -------------- | ---------------------------------------- |
| `claude-opus-4-20250514`         | 200K tokens    | Most powerful, complex reasoning         |
| `claude-sonnet-4-20250514`       | 200K tokens    | Best balance of speed + intelligence     |
| `claude-haiku-4-20250514`        | 200K tokens    | Fastest + cheapest                       |

### Key Differences from OpenAI

- Auth header is `x-api-key` (NOT `Authorization: Bearer`)
- Requires `anthropic-version` header
- `max_tokens` is **required** (OpenAI makes it optional)
- System prompt is a **top-level `system` field**, not a message role
- Response `content` is an **array of blocks**, not a plain string
- Temperature range is `0` to `1` (OpenAI allows `0` to `2`)
- `stop_reason` values differ: `end_turn` instead of `stop`
- No `n` parameter — always returns 1 response

---
---

# Quick Comparison Table

| Feature              | OpenAI                           | Google Gemini                                  | Anthropic Claude                          |
| -------------------- | -------------------------------- | ---------------------------------------------- | ----------------------------------------- |
| **Endpoint**         | `/v1/chat/completions`           | `/v1beta/models/{model}:generateContent`       | `/v1/messages`                            |
| **Auth**             | `Authorization: Bearer KEY`      | `?key=KEY` (query param)                       | `x-api-key: KEY` + `anthropic-version`    |
| **System Prompt**    | `role: "system"` in messages     | `systemInstruction` (separate field)           | `system` (top-level field)                |
| **AI Role Name**     | `assistant`                      | `model`                                        | `assistant`                               |
| **Message Format**   | `messages[].content` (string)    | `contents[].parts[].text`                      | `messages[].content` (string or array)    |
| **Response Text**    | `choices[0].message.content`     | `candidates[0].content.parts[0].text`          | `content[0].text`                         |
| **max_tokens**       | Optional                         | Optional (`maxOutputTokens`)                   | **Required**                              |
| **Temperature Range**| 0 – 2                            | 0 – 2                                          | 0 – 1                                    |
| **Has topK?**        | No                               | Yes                                            | Yes                                       |
| **Multiple Responses** | `n` parameter                  | `candidateCount`                               | Not supported                             |
| **JSON Mode**        | `response_format: json_object`   | `responseMimeType: application/json`           | Use system prompt to ask for JSON         |
| **Stop Reason Values** | `stop`, `length`              | `STOP`, `MAX_TOKENS`, `SAFETY`                 | `end_turn`, `max_tokens`, `stop_sequence` |

---

# Multi-Turn Conversation Example (All 3)

### OpenAI — Multi-turn

```json
{
  "model": "gpt-4o",
  "messages": [
    { "role": "system", "content": "You are a Java tutor." },
    { "role": "user", "content": "What is an interface?" },
    { "role": "assistant", "content": "An interface is a contract that defines methods a class must implement." },
    { "role": "user", "content": "Give me an example." }
  ]
}
```

### Gemini — Multi-turn

```json
{
  "systemInstruction": {
    "parts": [{ "text": "You are a Java tutor." }]
  },
  "contents": [
    { "role": "user", "parts": [{ "text": "What is an interface?" }] },
    { "role": "model", "parts": [{ "text": "An interface is a contract that defines methods a class must implement." }] },
    { "role": "user", "parts": [{ "text": "Give me an example." }] }
  ]
}
```

### Claude — Multi-turn

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 500,
  "system": "You are a Java tutor.",
  "messages": [
    { "role": "user", "content": "What is an interface?" },
    { "role": "assistant", "content": "An interface is a contract that defines methods a class must implement." },
    { "role": "user", "content": "Give me an example." }
  ]
}
```

---

# Error Handling (Common HTTP Status Codes)

| Status Code | Meaning                | What To Do                                         |
| ----------- | ---------------------- | -------------------------------------------------- |
| `200`       | Success                | Parse the response body                            |
| `400`       | Bad Request            | Check your JSON body, missing required fields      |
| `401`       | Unauthorized           | Wrong or missing API key                           |
| `403`       | Forbidden              | Key doesn't have permission for this model/action  |
| `404`       | Not Found              | Wrong URL or model name                            |
| `429`       | Rate Limited           | Too many requests. Wait and retry (check headers)  |
| `500`       | Server Error           | Provider-side issue. Retry after some time          |
| `529`       | Overloaded (Claude)    | Anthropic-specific. API is overloaded, retry later  |

---

# Tips

- **Never hardcode API keys** in code — use environment variables
- **Token counting**: 1 token ≈ 4 English characters ≈ 0.75 words
- **Cost**: You pay for input tokens + output tokens. Check each provider's pricing page
- **Streaming**: All 3 support SSE streaming — useful for real-time chat UIs
- **Context Window**: This is the total (input + output) token budget per request

---
---

# Section 4: Tool Calling / Function Calling (All 3 Providers)

> **What is Tool Calling?**
> You define "tools" (functions) that the model can choose to call. The model does NOT execute them — it just returns the function name + arguments. YOUR code executes the function, then you send the result back to the model so it can form a final answer.

## Flow (Same for all 3 providers)

```
1. You send: user message + tool definitions
2. Model responds: "I want to call function X with args Y"  (tool_call)
3. You execute: function X(Y) → get result
4. You send: the result back to the model
5. Model responds: final natural language answer using the result
```

---

## 4.1 OpenAI — Tool Calling

### Step 1: Send request with tools defined

```
POST https://api.openai.com/v1/chat/completions
```

```json
{
  "model": "gpt-4o",
  "messages": [
    { "role": "user", "content": "What is the weather in Delhi today?" }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get the current weather for a given city",
        "parameters": {
          "type": "object",
          "properties": {
            "city": {
              "type": "string",
              "description": "The city name, e.g. Delhi"
            },
            "unit": {
              "type": "string",
              "enum": ["celsius", "fahrenheit"],
              "description": "Temperature unit"
            }
          },
          "required": ["city"]
        }
      }
    }
  ],
  "tool_choice": "auto"
}
```

#### `tools` Array — Parameters Explained

| Field                          | Type   | Required | Description                                                          |
| ------------------------------ | ------ | -------- | -------------------------------------------------------------------- |
| `tools[].type`                 | string | Yes      | Always `"function"`                                                  |
| `tools[].function.name`        | string | Yes      | Function name (model will reference this). Use snake_case            |
| `tools[].function.description` | string | Yes      | What this function does. The better the description, the smarter the model calls it |
| `tools[].function.parameters`  | object | Yes      | JSON Schema defining the function's input parameters                 |
| `parameters.type`              | string | Yes      | Always `"object"`                                                    |
| `parameters.properties`        | object | Yes      | Each key = param name, value = `{type, description, enum}`           |
| `parameters.required`          | array  | No       | Which parameters are mandatory                                       |

#### `tool_choice` — Controls whether model uses tools

| Value                                     | Behavior                                              |
| ----------------------------------------- | ----------------------------------------------------- |
| `"auto"`                                  | Model decides whether to call a tool or reply directly |
| `"none"`                                  | Model will NOT call any tool                           |
| `"required"`                              | Model MUST call at least one tool                      |
| `{"type":"function","function":{"name":"get_weather"}}` | Force call this specific function     |

### Step 2: Model responds with a tool call (NOT the final answer)

```json
{
  "id": "chatcmpl-abc123",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"city\": \"Delhi\", \"unit\": \"celsius\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ]
}
```

**Key points:**
- `content` is `null` (model didn't generate text, it wants to call a function)
- `tool_calls` array has the function name + arguments as a JSON string
- `finish_reason` is `"tool_calls"` (not `"stop"`)
- `arguments` is a **stringified JSON** — you must `JSON.parse()` it
- `id` (`call_abc123`) is important — you need it in Step 3

### Step 3: You execute the function, then send the result back

```json
{
  "model": "gpt-4o",
  "messages": [
    { "role": "user", "content": "What is the weather in Delhi today?" },
    {
      "role": "assistant",
      "content": null,
      "tool_calls": [
        {
          "id": "call_abc123",
          "type": "function",
          "function": {
            "name": "get_weather",
            "arguments": "{\"city\": \"Delhi\", \"unit\": \"celsius\"}"
          }
        }
      ]
    },
    {
      "role": "tool",
      "tool_call_id": "call_abc123",
      "content": "{\"temperature\": 38, \"condition\": \"Sunny\", \"humidity\": 45}"
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get the current weather for a given city",
        "parameters": {
          "type": "object",
          "properties": {
            "city": { "type": "string" },
            "unit": { "type": "string", "enum": ["celsius", "fahrenheit"] }
          },
          "required": ["city"]
        }
      }
    }
  ]
}
```

**Key points about the tool result message:**
- `role` is `"tool"` (special role, only for tool results)
- `tool_call_id` MUST match the `id` from the tool_call in step 2
- `content` is the stringified result from your function execution
- You must include the full conversation history (user → assistant → tool)

### Step 4: Model gives the final answer

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "The current weather in Delhi is 38°C with sunny skies and 45% humidity."
      },
      "finish_reason": "stop"
    }
  ]
}
```

---

## 4.2 Google Gemini — Tool Calling (Function Calling)

### Step 1: Send request with tools defined

```
POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=YOUR_KEY
```

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [{ "text": "What is the weather in Delhi today?" }]
    }
  ],
  "tools": [
    {
      "functionDeclarations": [
        {
          "name": "get_weather",
          "description": "Get the current weather for a given city",
          "parameters": {
            "type": "OBJECT",
            "properties": {
              "city": {
                "type": "STRING",
                "description": "The city name, e.g. Delhi"
              },
              "unit": {
                "type": "STRING",
                "enum": ["celsius", "fahrenheit"],
                "description": "Temperature unit"
              }
            },
            "required": ["city"]
          }
        }
      ]
    }
  ],
  "toolConfig": {
    "functionCallingConfig": {
      "mode": "AUTO"
    }
  }
}
```

#### Gemini Tool Parameters Explained

| Field                                    | Type   | Required | Description                                                          |
| ---------------------------------------- | ------ | -------- | -------------------------------------------------------------------- |
| `tools[].functionDeclarations`           | array  | Yes      | Array of function definitions (NOT wrapped in `type: function` like OpenAI) |
| `functionDeclarations[].name`            | string | Yes      | Function name                                                        |
| `functionDeclarations[].description`     | string | Yes      | What the function does                                               |
| `functionDeclarations[].parameters`      | object | Yes      | JSON Schema — but types are UPPERCASE (`STRING`, `OBJECT`, `NUMBER`) |
| `toolConfig`                             | object | No       | Controls function calling behavior                                    |
| `toolConfig.functionCallingConfig.mode`  | string | No       | `AUTO`, `NONE`, or `ANY` (ANY = must call a function)                |

**Gemini differences from OpenAI:**
- Tool definitions go inside `functionDeclarations` array (no `type: function` wrapper)
- JSON Schema types are **UPPERCASE**: `STRING`, `NUMBER`, `INTEGER`, `BOOLEAN`, `OBJECT`, `ARRAY`
- Mode `ANY` = OpenAI's `required`

### Step 2: Model responds with a function call

```json
{
  "candidates": [
    {
      "content": {
        "parts": [
          {
            "functionCall": {
              "name": "get_weather",
              "args": {
                "city": "Delhi",
                "unit": "celsius"
              }
            }
          }
        ],
        "role": "model"
      },
      "finishReason": "STOP"
    }
  ]
}
```

**Key points:**
- Response is in `parts[].functionCall` (NOT `tool_calls` like OpenAI)
- `args` is a **real JSON object** (NOT a stringified JSON like OpenAI — cleaner!)
- No separate `call_id` — function is identified by `name`

### Step 3: Send the function result back

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [{ "text": "What is the weather in Delhi today?" }]
    },
    {
      "role": "model",
      "parts": [
        {
          "functionCall": {
            "name": "get_weather",
            "args": { "city": "Delhi", "unit": "celsius" }
          }
        }
      ]
    },
    {
      "role": "user",
      "parts": [
        {
          "functionResponse": {
            "name": "get_weather",
            "response": {
              "temperature": 38,
              "condition": "Sunny",
              "humidity": 45
            }
          }
        }
      ]
    }
  ],
  "tools": [
    {
      "functionDeclarations": [
        {
          "name": "get_weather",
          "description": "Get the current weather for a given city",
          "parameters": {
            "type": "OBJECT",
            "properties": {
              "city": { "type": "STRING" },
              "unit": { "type": "STRING", "enum": ["celsius", "fahrenheit"] }
            },
            "required": ["city"]
          }
        }
      ]
    }
  ]
}
```

**Key points about Gemini tool result:**
- Tool result goes as `role: "user"` with `functionResponse` part (NOT a separate role)
- `functionResponse.name` must match the function that was called
- `response` is a real JSON object (not stringified)

### Step 4: Model gives the final answer

```json
{
  "candidates": [
    {
      "content": {
        "parts": [
          { "text": "The current weather in Delhi is 38°C with sunny skies and 45% humidity." }
        ],
        "role": "model"
      },
      "finishReason": "STOP"
    }
  ]
}
```

---

## 4.3 Anthropic Claude — Tool Use

### Step 1: Send request with tools defined

```
POST https://api.anthropic.com/v1/messages
```

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 1024,
  "messages": [
    { "role": "user", "content": "What is the weather in Delhi today?" }
  ],
  "tools": [
    {
      "name": "get_weather",
      "description": "Get the current weather for a given city",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": {
            "type": "string",
            "description": "The city name, e.g. Delhi"
          },
          "unit": {
            "type": "string",
            "enum": ["celsius", "fahrenheit"],
            "description": "Temperature unit"
          }
        },
        "required": ["city"]
      }
    }
  ],
  "tool_choice": { "type": "auto" }
}
```

#### Claude Tool Parameters Explained

| Field                    | Type   | Required | Description                                                          |
| ------------------------ | ------ | -------- | -------------------------------------------------------------------- |
| `tools[].name`           | string | Yes      | Function name (no `type: function` wrapper unlike OpenAI)            |
| `tools[].description`    | string | Yes      | What the function does                                               |
| `tools[].input_schema`   | object | Yes      | JSON Schema (called `input_schema`, NOT `parameters` like OpenAI)    |
| `tool_choice`            | object | No       | Controls tool use behavior                                           |

#### `tool_choice` — Controls whether model uses tools

| Value                                    | Behavior                                              |
| ---------------------------------------- | ----------------------------------------------------- |
| `{"type": "auto"}`                      | Model decides whether to use a tool                    |
| `{"type": "any"}`                       | Model MUST use at least one tool                       |
| `{"type": "tool", "name": "get_weather"}` | Force call this specific tool                       |

**Claude differences from OpenAI:**
- Schema field is `input_schema` (NOT `parameters`)
- No `type: function` wrapper — tools are flat objects
- `tool_choice` uses `{type: "any"}` instead of `"required"`

### Step 2: Model responds with a tool use block

```json
{
  "id": "msg_01Aq1234",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Let me check the weather in Delhi for you."
    },
    {
      "type": "tool_use",
      "id": "toolu_01A1234",
      "name": "get_weather",
      "input": {
        "city": "Delhi",
        "unit": "celsius"
      }
    }
  ],
  "stop_reason": "tool_use",
  "usage": {
    "input_tokens": 50,
    "output_tokens": 30
  }
}
```

**Key points:**
- `content` array can have BOTH `text` AND `tool_use` blocks (Claude often explains what it's doing)
- `tool_use` block has `id`, `name`, and `input` (a real JSON object, not stringified)
- `stop_reason` is `"tool_use"` (not `"tool_calls"` like OpenAI)
- `id` starts with `toolu_` — you need this in Step 3

### Step 3: Send the tool result back

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 1024,
  "messages": [
    { "role": "user", "content": "What is the weather in Delhi today?" },
    {
      "role": "assistant",
      "content": [
        {
          "type": "text",
          "text": "Let me check the weather in Delhi for you."
        },
        {
          "type": "tool_use",
          "id": "toolu_01A1234",
          "name": "get_weather",
          "input": { "city": "Delhi", "unit": "celsius" }
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_01A1234",
          "content": "{\"temperature\": 38, \"condition\": \"Sunny\", \"humidity\": 45}"
        }
      ]
    }
  ],
  "tools": [
    {
      "name": "get_weather",
      "description": "Get the current weather for a given city",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": { "type": "string" },
          "unit": { "type": "string", "enum": ["celsius", "fahrenheit"] }
        },
        "required": ["city"]
      }
    }
  ]
}
```

**Key points about Claude tool result:**
- Tool result goes as `role: "user"` with `content` array containing `type: "tool_result"`
- `tool_use_id` MUST match the `id` from the `tool_use` block (`toolu_01A1234`)
- `content` can be a string or array of content blocks
- You can also add `"is_error": true` if the function execution failed

### Step 4: Model gives the final answer

```json
{
  "id": "msg_01Bx5678",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "The current weather in Delhi is 38°C with sunny skies and 45% humidity."
    }
  ],
  "stop_reason": "end_turn"
}
```

---

## Tool Calling — Quick Comparison

| Feature                    | OpenAI                                | Google Gemini                           | Anthropic Claude                          |
| -------------------------- | ------------------------------------- | --------------------------------------- | ----------------------------------------- |
| **Tool definition wrapper**| `{type:"function", function:{...}}`   | `{functionDeclarations:[{...}]}`        | `{name, description, input_schema}`       |
| **Schema field name**      | `parameters`                          | `parameters` (UPPERCASE types)          | `input_schema`                            |
| **Model's tool call**      | `tool_calls[].function`               | `parts[].functionCall`                  | `content[].type:"tool_use"`              |
| **Args format**            | Stringified JSON                      | Real JSON object                        | Real JSON object                          |
| **Call ID**                | `call_abc123`                         | None (uses function name)               | `toolu_01A1234`                           |
| **Tool result role**       | `role: "tool"`                        | `role: "user"` + `functionResponse`    | `role: "user"` + `type: "tool_result"`   |
| **Force tool use**         | `"required"` or specific function     | `mode: "ANY"`                           | `{type: "any"}` or specific tool         |
| **finish_reason value**    | `"tool_calls"`                        | `"STOP"`                                | `"tool_use"`                              |

---
---

# Section 5: Vision / Image Input (All 3 Providers)

> Send an image (URL or base64) along with your text prompt. The model analyzes the image and responds.

## 5.1 OpenAI — Vision

```json
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "What is in this image?" },
        {
          "type": "image_url",
          "image_url": {
            "url": "https://example.com/photo.jpg",
            "detail": "auto"
          }
        }
      ]
    }
  ],
  "max_tokens": 300
}
```

**Using base64 instead of URL:**
```json
{
  "type": "image_url",
  "image_url": {
    "url": "data:image/jpeg;base64,/9j/4AAQSkZJRg..."
  }
}
```

| Field            | Description                                                              |
| ---------------- | ------------------------------------------------------------------------ |
| `type`           | `"image_url"` for image content                                         |
| `image_url.url`  | HTTP URL or `data:image/jpeg;base64,...` encoded image                   |
| `image_url.detail` | `"low"` (faster, cheaper), `"high"` (detailed), `"auto"` (model decides) |

## 5.2 Google Gemini — Vision

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [
        { "text": "What is in this image?" },
        {
          "inlineData": {
            "mimeType": "image/jpeg",
            "data": "/9j/4AAQSkZJRg..."  
          }
        }
      ]
    }
  ]
}
```

| Field                     | Description                                               |
| ------------------------- | --------------------------------------------------------- |
| `inlineData.mimeType`     | `image/jpeg`, `image/png`, `image/webp`, `image/gif`     |
| `inlineData.data`         | Base64-encoded image data (no `data:` prefix needed)      |

**Note:** Gemini also supports `fileData` with `fileUri` for files uploaded via the File API.

## 5.3 Anthropic Claude — Vision

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 300,
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "image",
          "source": {
            "type": "base64",
            "media_type": "image/jpeg",
            "data": "/9j/4AAQSkZJRg..."
          }
        },
        { "type": "text", "text": "What is in this image?" }
      ]
    }
  ]
}
```

**Using URL instead of base64:**
```json
{
  "type": "image",
  "source": {
    "type": "url",
    "url": "https://example.com/photo.jpg"
  }
}
```

| Field                  | Description                                                   |
| ---------------------- | ------------------------------------------------------------- |
| `type`                 | `"image"` (NOT `"image_url"` like OpenAI)                    |
| `source.type`          | `"base64"` or `"url"`                                        |
| `source.media_type`    | `image/jpeg`, `image/png`, `image/gif`, `image/webp`         |
| `source.data`          | Base64 string (when type is base64)                           |
| `source.url`           | HTTP URL (when type is url)                                   |

---
---

# Section 6: Streaming (All 3 Providers)

> Streaming returns the response token-by-token as Server-Sent Events (SSE). Useful for real-time chat UIs.

## 6.1 OpenAI — Streaming

Add `"stream": true` to the request body. Response comes as SSE:

```
data: {"id":"chatcmpl-abc","choices":[{"delta":{"role":"assistant"},"index":0}]}
data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":"The"},"index":0}]}
data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":" weather"},"index":0}]}
data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":" is"},"index":0}]}
data: [DONE]
```

- Each chunk has `delta` instead of `message`
- `delta.content` has the new token(s)
- Stream ends with `data: [DONE]`

## 6.2 Google Gemini — Streaming

Use `streamGenerateContent` instead of `generateContent` in the URL:

```
POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:streamGenerateContent?alt=sse&key=YOUR_KEY
```

Response comes as SSE:
```
data: {"candidates":[{"content":{"parts":[{"text":"The"}],"role":"model"}}]}
data: {"candidates":[{"content":{"parts":[{"text":" weather is"}],"role":"model"}}]}
data: {"candidates":[{"content":{"parts":[{"text":" sunny."}],"role":"model"},"finishReason":"STOP"}]}
```

- Change endpoint from `:generateContent` to `:streamGenerateContent`
- Add `?alt=sse` query parameter
- Each chunk is a partial candidate

## 6.3 Anthropic Claude — Streaming

Add `"stream": true` to the request body. Response comes as SSE:

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_01","type":"message","role":"assistant","model":"claude-sonnet-4-20250514"}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"The"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" weather"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":15}}

event: message_stop
data: {"type":"message_stop"}
```

**Claude streaming events in order:**

| Event                  | When it fires                         | Key data                    |
| ---------------------- | ------------------------------------- | --------------------------- |
| `message_start`        | Start of response                     | Message metadata            |
| `content_block_start`  | Start of a content block              | Block type (text/tool_use)  |
| `content_block_delta`  | Each new token                        | `delta.text` = new text     |
| `content_block_stop`   | End of a content block                | —                           |
| `message_delta`        | End of message                        | `stop_reason`, final usage  |
| `message_stop`         | Stream complete                       | —                           |

---
---

# Section 7: Structured Output / JSON Mode

## 7.1 OpenAI — JSON Mode

```json
{
  "model": "gpt-4o",
  "messages": [
    { "role": "system", "content": "Return a JSON object with name and age." },
    { "role": "user", "content": "Tell me about Albert Einstein" }
  ],
  "response_format": { "type": "json_object" }
}
```

**Strict Structured Output (with schema):**
```json
{
  "model": "gpt-4o",
  "messages": [
    { "role": "user", "content": "Tell me about Albert Einstein" }
  ],
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "person_info",
      "strict": true,
      "schema": {
        "type": "object",
        "properties": {
          "name": { "type": "string" },
          "age_at_death": { "type": "integer" },
          "field": { "type": "string" }
        },
        "required": ["name", "age_at_death", "field"],
        "additionalProperties": false
      }
    }
  }
}
```

## 7.2 Google Gemini — JSON Mode

```json
{
  "contents": [
    { "role": "user", "parts": [{ "text": "Tell me about Albert Einstein" }] }
  ],
  "generationConfig": {
    "responseMimeType": "application/json",
    "responseSchema": {
      "type": "OBJECT",
      "properties": {
        "name": { "type": "STRING" },
        "age_at_death": { "type": "INTEGER" },
        "field": { "type": "STRING" }
      },
      "required": ["name", "age_at_death", "field"]
    }
  }
}
```

## 7.3 Anthropic Claude — JSON Mode

Claude doesn't have a dedicated JSON mode. Use the system prompt or prefill:

```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 300,
  "system": "Always respond with valid JSON only. No other text.",
  "messages": [
    { "role": "user", "content": "Tell me about Albert Einstein. Return JSON with name, age_at_death, field." },
    { "role": "assistant", "content": "{" }
  ]
}
```

**Tip:** Prefilling assistant message with `{` forces the model to continue generating JSON.

Alternatively, use tool calling with a single tool to enforce structured output — the `input` will always be valid JSON matching your schema.

---
---

# Section 8: Multiple / Parallel Tool Calls

## OpenAI — Parallel Tool Calls

OpenAI can call multiple tools in a single response:

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [
        {
          "id": "call_001",
          "type": "function",
          "function": { "name": "get_weather", "arguments": "{\"city\":\"Delhi\"}" }
        },
        {
          "id": "call_002",
          "type": "function",
          "function": { "name": "get_weather", "arguments": "{\"city\":\"Mumbai\"}" }
        }
      ]
    },
    "finish_reason": "tool_calls"
  }]
}
```

Send back BOTH results, each with matching `tool_call_id`:

```json
{
  "messages": [
    { "role": "user", "content": "Compare weather in Delhi and Mumbai" },
    {
      "role": "assistant", "content": null,
      "tool_calls": [
        { "id": "call_001", "type": "function", "function": { "name": "get_weather", "arguments": "{\"city\":\"Delhi\"}" } },
        { "id": "call_002", "type": "function", "function": { "name": "get_weather", "arguments": "{\"city\":\"Mumbai\"}" } }
      ]
    },
    { "role": "tool", "tool_call_id": "call_001", "content": "{\"temp\": 38}" },
    { "role": "tool", "tool_call_id": "call_002", "content": "{\"temp\": 32}" }
  ]
}
```

To disable parallel calls: `"parallel_tool_calls": false`

## Claude — Parallel Tool Use

Claude can also return multiple tool_use blocks in one response:

```json
{
  "content": [
    { "type": "text", "text": "Let me check both cities." },
    { "type": "tool_use", "id": "toolu_01", "name": "get_weather", "input": {"city": "Delhi"} },
    { "type": "tool_use", "id": "toolu_02", "name": "get_weather", "input": {"city": "Mumbai"} }
  ],
  "stop_reason": "tool_use"
}
```

Send back all results in one user message:

```json
{
  "role": "user",
  "content": [
    { "type": "tool_result", "tool_use_id": "toolu_01", "content": "{\"temp\": 38}" },
    { "type": "tool_result", "tool_use_id": "toolu_02", "content": "{\"temp\": 32}" }
  ]
}
```

## Gemini — Parallel Function Calls

Gemini returns multiple `functionCall` parts:

```json
{
  "candidates": [{
    "content": {
      "parts": [
        { "functionCall": { "name": "get_weather", "args": {"city": "Delhi"} } },
        { "functionCall": { "name": "get_weather", "args": {"city": "Mumbai"} } }
      ],
      "role": "model"
    }
  }]
}
```

Send back all results:

```json
{
  "role": "user",
  "parts": [
    { "functionResponse": { "name": "get_weather", "response": {"temp": 38} } },
    { "functionResponse": { "name": "get_weather", "response": {"temp": 32} } }
  ]
}
