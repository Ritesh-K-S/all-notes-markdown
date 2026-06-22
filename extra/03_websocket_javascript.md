# WebSocket — Client-Side (JavaScript) Complete Learning Notes

---

## Table of Contents

1. [What Are WebSockets?](#1-what-are-websockets)
2. [WebSocket vs HTTP — The Core Difference](#2-websocket-vs-http--the-core-difference)
3. [The WebSocket Lifecycle](#3-the-websocket-lifecycle)
4. [Your First WebSocket Connection](#4-your-first-websocket-connection)
5. [Sending Messages](#5-sending-messages)
6. [Receiving Messages](#6-receiving-messages)
7. [JSON Messaging Pattern](#7-json-messaging-pattern)
8. [Connection States](#8-connection-states)
9. [Error Handling](#9-error-handling)
10. [Automatic Reconnection](#10-automatic-reconnection)
11. [Sending Binary Data (Audio, Images, Files)](#11-sending-binary-data-audio-images-files)
12. [Heartbeat / Keep-Alive](#12-heartbeat--keep-alive)
13. [Real-World Example: Chat Application](#13-real-world-example-chat-application)
14. [Real-World Example: Live Audio Streaming](#14-real-world-example-live-audio-streaming)
15. [Real-World Example: Real-Time Notifications](#15-real-world-example-real-time-notifications)
16. [Working with MediaRecorder (Audio/Video)](#16-working-with-mediarecorder-audiovideo)
17. [Security Considerations](#17-security-considerations)
18. [Debugging WebSockets](#18-debugging-websockets)
19. [Common Pitfalls & Solutions](#19-common-pitfalls--solutions)
20. [Mental Model & Quick Reference](#20-mental-model--quick-reference)

---

## 1. What Are WebSockets?

### The Problem with HTTP

HTTP is a **request-response** protocol. The client asks, the server answers, and the connection is done.

```
Client: "Hey server, any new messages?"     →  Server: "Nope"
Client: "How about now?"                    →  Server: "Nope"
Client: "Now?"                              →  Server: "Nope"
Client: "Now??"                             →  Server: "Yes! Here's one"
Client: "Any more?"                         →  Server: "Nope"
... (polling every 1 second = 3600 requests/hour = waste)
```

This is called **polling**, and it's inefficient:
- Most requests return "nothing new"
- Each request has HTTP headers overhead (~800 bytes per request)
- Delay between events (up to 1 second)
- Server wastes resources handling empty requests

### The WebSocket Solution

WebSockets create a **persistent, bidirectional** connection. Once connected, both sides can send messages anytime, instantly.

```
Client: "Let's open a WebSocket connection"  →  Server: "Sure, connected!"
                                                 
         ←── "New message from Alice" ──────  Server pushes instantly
         ←── "Bob is typing..." ────────────  Server pushes instantly
Client:  ──── "Hello everyone!" ────────────→  Client sends anytime
         ←── "Message delivered ✓" ─────────  Server pushes instantly
```

**No polling. No wasted requests. Instant delivery in both directions.**

---

## 2. WebSocket vs HTTP — The Core Difference

```
HTTP (Request-Response):
┌────────┐         ┌────────┐
│ Client │──req──► │ Server │
│        │◄──res── │        │
│        │         │        │
│        │──req──► │        │   Each arrow = new connection
│        │◄──res── │        │
└────────┘         └────────┘

WebSocket (Persistent Bidirectional):
┌────────┐         ┌────────┐
│ Client │◄═══════►│ Server │   One persistent connection
│        │  ↕ ↕ ↕  │        │   Both sides send anytime
└────────┘         └────────┘
```

| Feature | HTTP | WebSocket |
|---------|------|-----------|
| Connection | New per request | Single persistent |
| Direction | Client → Server only | Both directions |
| Overhead per message | ~800 bytes (headers) | ~6 bytes (frame header) |
| Latency | High (new TCP handshake) | Low (already connected) |
| Server can push? | No (client must poll) | Yes |
| Use case | REST APIs, page loads | Chat, gaming, live data, audio |

### How the Connection Starts (The Upgrade Handshake)

WebSocket connections start as HTTP, then **upgrade**:

```
1. Client sends HTTP request with upgrade header:
   GET /ws/chat HTTP/1.1
   Host: example.com
   Upgrade: websocket              ← "I want to upgrade to WebSocket"
   Connection: Upgrade
   Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

2. Server responds with 101:
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

3. Now it's a WebSocket connection — no more HTTP!
   ◄═══════════ bidirectional messages ═══════════►
```

**The URL scheme changes from `http://` to `ws://` (or `https://` to `wss://` for secure).**

---

## 3. The WebSocket Lifecycle

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   new WebSocket(url)                                     │
│        │                                                 │
│        ▼                                                 │
│   ┌─────────┐                                           │
│   │CONNECTING│ ── readyState = 0                        │
│   └────┬────┘                                           │
│        │  Handshake succeeds                             │
│        ▼                                                 │
│   ┌─────────┐                                           │
│   │  OPEN   │ ── readyState = 1                         │
│   │         │    onopen fires                            │
│   │         │    Can send/receive messages               │
│   └────┬────┘                                           │
│        │  close() called or error                        │
│        ▼                                                 │
│   ┌─────────┐                                           │
│   │ CLOSING │ ── readyState = 2                         │
│   └────┬────┘                                           │
│        │  Close handshake complete                        │
│        ▼                                                 │
│   ┌─────────┐                                           │
│   │ CLOSED  │ ── readyState = 3                         │
│   │         │    onclose fires                           │
│   └─────────┘                                           │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 4. Your First WebSocket Connection

### The Simplest Possible Example

```javascript
// Create a connection
const ws = new WebSocket("ws://localhost:8000/ws");

// Connection opened
ws.onopen = function(event) {
    console.log("✅ Connected to server!");
    ws.send("Hello Server!");
};

// Received a message from server
ws.onmessage = function(event) {
    console.log("📩 Server says:", event.data);
};

// Connection closed
ws.onclose = function(event) {
    console.log("🔌 Disconnected:", event.code, event.reason);
};

// Error occurred
ws.onerror = function(event) {
    console.error("❌ WebSocket error:", event);
};
```

### Using addEventListener (Alternative Syntax)

```javascript
const ws = new WebSocket("ws://localhost:8000/ws");

ws.addEventListener("open", (event) => {
    console.log("Connected!");
});

ws.addEventListener("message", (event) => {
    console.log("Received:", event.data);
});

ws.addEventListener("close", (event) => {
    console.log("Closed:", event.code);
});

ws.addEventListener("error", (event) => {
    console.error("Error:", event);
});
```

Both syntaxes work. `addEventListener` allows multiple handlers per event; `onmessage` allows only one.

### Secure WebSocket (WSS)

```javascript
// For HTTPS sites, use wss:// (WebSocket Secure)
const ws = new WebSocket("wss://mysite.com/ws");

// wss:// is to ws:// what https:// is to http://
// Always use wss:// in production!
```

---

## 5. Sending Messages

### Sending Text

```javascript
// Simple string
ws.send("Hello!");

// IMPORTANT: Check if connection is open before sending
if (ws.readyState === WebSocket.OPEN) {
    ws.send("Hello!");
} else {
    console.log("Not connected yet!");
}
```

### Sending JSON (Most Common Pattern)

```javascript
// You MUST stringify JSON — WebSocket only sends strings or binary
const message = {
    type: "chat_message",
    content: "Hello everyone!",
    timestamp: Date.now()
};

ws.send(JSON.stringify(message));
```

### Sending Binary Data

```javascript
// ArrayBuffer
const buffer = new ArrayBuffer(8);
ws.send(buffer);

// Blob
const blob = new Blob(["binary data"], { type: "application/octet-stream" });
ws.send(blob);

// TypedArray
const bytes = new Uint8Array([72, 101, 108, 108, 111]);
ws.send(bytes.buffer);
```

### Checking Buffer Status

```javascript
// bufferedAmount tells you how many bytes are queued but not yet sent
if (ws.bufferedAmount === 0) {
    ws.send(largeData);  // Safe to send — buffer is empty
} else {
    console.log(`${ws.bufferedAmount} bytes still in buffer, waiting...`);
}
```

---

## 6. Receiving Messages

### Text Messages

```javascript
ws.onmessage = function(event) {
    // event.data is always a string (for text messages)
    console.log(event.data);             // Raw string
    console.log(typeof event.data);       // "string"
};
```

### JSON Messages

```javascript
ws.onmessage = function(event) {
    const data = JSON.parse(event.data);
    console.log(data.type);              // e.g., "chat_message"
    console.log(data.content);           // e.g., "Hello!"
};
```

### Binary Messages

```javascript
// Tell WebSocket to give you binary data as ArrayBuffer (not Blob)
ws.binaryType = "arraybuffer";   // or "blob" (default)

ws.onmessage = function(event) {
    if (typeof event.data === "string") {
        // Text message
        const json = JSON.parse(event.data);
        handleTextMessage(json);
    } else {
        // Binary message (ArrayBuffer or Blob)
        handleBinaryMessage(event.data);
    }
};
```

### Mixed Text + Binary Protocol

A common pattern: server sends JSON for control messages and raw binary for audio/images.

```javascript
ws.binaryType = "arraybuffer";

ws.onmessage = function(event) {
    if (typeof event.data === "string") {
        // Control message (JSON)
        const msg = JSON.parse(event.data);

        switch (msg.type) {
            case "question":
                displayQuestion(msg.text);
                break;
            case "processing":
                showSpinner();
                break;
            case "error":
                showError(msg.text);
                break;
        }
    } else {
        // Audio data (binary)
        playAudio(event.data);
    }
};
```

---

## 7. JSON Messaging Pattern

The most common WebSocket pattern is to send JSON objects with a `type` field that acts as a message router.

### Message Protocol Design

```javascript
// Define your message types
// Client → Server:
{ type: "chat_message",   content: "Hello!", room: "general" }
{ type: "typing_start",   room: "general" }
{ type: "typing_stop",    room: "general" }
{ type: "join_room",      room: "general" }

// Server → Client:
{ type: "chat_message",   content: "Hello!", sender: "Alice", timestamp: "..." }
{ type: "user_joined",    username: "Bob" }
{ type: "user_left",      username: "Charlie" }
{ type: "error",          message: "Room not found" }
```

### Message Handler (Switch Pattern)

```javascript
ws.onmessage = function(event) {
    const message = JSON.parse(event.data);

    switch (message.type) {
        case "chat_message":
            appendMessage(message.sender, message.content);
            break;

        case "user_joined":
            showNotification(`${message.username} joined`);
            updateUserList(message.users);
            break;

        case "user_left":
            showNotification(`${message.username} left`);
            updateUserList(message.users);
            break;

        case "typing":
            showTypingIndicator(message.username);
            break;

        case "error":
            showError(message.message);
            break;

        default:
            console.warn("Unknown message type:", message.type);
    }
};
```

### Message Handler (Map Pattern — Cleaner)

```javascript
// Define handlers as a map
const handlers = {
    chat_message: (data) => appendMessage(data.sender, data.content),
    user_joined:  (data) => showNotification(`${data.username} joined`),
    user_left:    (data) => showNotification(`${data.username} left`),
    typing:       (data) => showTypingIndicator(data.username),
    error:        (data) => showError(data.message),
};

ws.onmessage = function(event) {
    const message = JSON.parse(event.data);
    const handler = handlers[message.type];

    if (handler) {
        handler(message);
    } else {
        console.warn("Unknown message type:", message.type);
    }
};
```

### Helper Function to Send Messages

```javascript
function sendMessage(type, data = {}) {
    if (ws.readyState !== WebSocket.OPEN) {
        console.error("WebSocket is not connected");
        return false;
    }

    ws.send(JSON.stringify({ type, ...data }));
    return true;
}

// Usage — clean and readable:
sendMessage("chat_message", { content: "Hello!", room: "general" });
sendMessage("typing_start", { room: "general" });
sendMessage("join_room",    { room: "general" });
```

---

## 8. Connection States

```javascript
const ws = new WebSocket("ws://localhost:8000/ws");

// Check state at any time:
switch (ws.readyState) {
    case WebSocket.CONNECTING:  // 0
        console.log("Still connecting...");
        break;
    case WebSocket.OPEN:        // 1
        console.log("Connected and ready!");
        break;
    case WebSocket.CLOSING:     // 2
        console.log("Connection closing...");
        break;
    case WebSocket.CLOSED:      // 3
        console.log("Connection closed");
        break;
}

// Constants reference:
// WebSocket.CONNECTING = 0
// WebSocket.OPEN       = 1
// WebSocket.CLOSING    = 2
// WebSocket.CLOSED     = 3
```

---

## 9. Error Handling

### The `onerror` Event

```javascript
ws.onerror = function(event) {
    // NOTE: The error event gives you almost NO useful information!
    // For security reasons, browsers don't expose error details.
    console.error("WebSocket error occurred");
    // event.message is usually undefined
    // You can't know if it was a network error, auth failure, etc.
};
```

### The `onclose` Event — Where the Real Info Is

```javascript
ws.onclose = function(event) {
    console.log("Code:", event.code);         // Numeric close code
    console.log("Reason:", event.reason);     // Human-readable reason
    console.log("Clean:", event.wasClean);    // Was it a graceful close?
};
```

### Close Codes Reference

| Code | Name | Meaning |
|------|------|---------|
| `1000` | Normal Closure | Everything's fine, closing normally |
| `1001` | Going Away | Server shutting down, or page navigating away |
| `1002` | Protocol Error | Something violated the WebSocket protocol |
| `1003` | Unsupported Data | Received data type not supported |
| `1005` | No Status Received | No status code in close frame (abnormal) |
| `1006` | Abnormal Closure | Connection dropped (no close frame) — network issue |
| `1008` | Policy Violation | Message violated a policy |
| `1009` | Message Too Big | Message exceeds size limit |
| `1011` | Internal Error | Server hit an unexpected error |
| `1012` | Service Restart | Server is restarting |
| `1013` | Try Again Later | Server is overloaded |
| `4000-4999` | Application Codes | Custom codes for your app |

### Closing the Connection

```javascript
// Graceful close
ws.close();                          // Code: 1000 (normal)
ws.close(1000, "Done chatting");     // With reason

// Custom close code (4000-4999 range)
ws.close(4001, "Session expired");
ws.close(4002, "Authentication failed");
```

---

## 10. Automatic Reconnection

The browser's `WebSocket` API does NOT auto-reconnect. You must implement it yourself.

### Simple Reconnection

```javascript
let ws;
let reconnectAttempts = 0;
const MAX_RECONNECT_ATTEMPTS = 10;

function connect() {
    ws = new WebSocket("ws://localhost:8000/ws");

    ws.onopen = function() {
        console.log("Connected!");
        reconnectAttempts = 0;   // Reset counter on successful connection
    };

    ws.onmessage = function(event) {
        handleMessage(JSON.parse(event.data));
    };

    ws.onclose = function(event) {
        if (event.code === 1000) {
            console.log("Closed normally, not reconnecting");
            return;   // Don't reconnect if closed intentionally
        }

        if (reconnectAttempts < MAX_RECONNECT_ATTEMPTS) {
            reconnectAttempts++;
            const delay = Math.min(1000 * Math.pow(2, reconnectAttempts), 30000);
            console.log(`Reconnecting in ${delay}ms (attempt ${reconnectAttempts})...`);
            setTimeout(connect, delay);
        } else {
            console.error("Max reconnection attempts reached");
            showError("Connection lost. Please refresh the page.");
        }
    };

    ws.onerror = function() {
        // Error will trigger onclose, which handles reconnection
    };
}

// Start
connect();
```

### Exponential Backoff Explained

```
Attempt 1: wait 2 seconds    (2^1 × 1000ms)
Attempt 2: wait 4 seconds    (2^2 × 1000ms)
Attempt 3: wait 8 seconds    (2^3 × 1000ms)
Attempt 4: wait 16 seconds   (2^4 × 1000ms)
Attempt 5: wait 30 seconds   (capped at 30s)
Attempt 6: wait 30 seconds   (capped at 30s)
```

This prevents hammering the server when it's down.

### Reconnection with Queue (Don't Lose Messages)

```javascript
let ws;
let messageQueue = [];   // Buffer messages while disconnected

function connect() {
    ws = new WebSocket("ws://localhost:8000/ws");

    ws.onopen = function() {
        console.log("Connected! Flushing message queue...");

        // Send any messages that were queued while disconnected
        while (messageQueue.length > 0) {
            const msg = messageQueue.shift();
            ws.send(msg);
        }
    };

    ws.onclose = function(event) {
        if (event.code !== 1000) {
            setTimeout(connect, 3000);
        }
    };
}

function sendMessage(type, data = {}) {
    const msg = JSON.stringify({ type, ...data });

    if (ws && ws.readyState === WebSocket.OPEN) {
        ws.send(msg);
    } else {
        // Queue the message — it'll be sent when reconnected
        messageQueue.push(msg);
        console.log("Queued message (offline):", type);
    }
}

connect();
```

---

## 11. Sending Binary Data (Audio, Images, Files)

### Sending a File

```javascript
const fileInput = document.getElementById("fileInput");

fileInput.addEventListener("change", function() {
    const file = this.files[0];
    const reader = new FileReader();

    reader.onload = function(e) {
        // Send the file as binary
        ws.send(e.target.result);   // ArrayBuffer
    };

    reader.readAsArrayBuffer(file);
});
```

### Sending Base64-Encoded Data (Common Pattern)

When you need to mix text metadata with binary data:

```javascript
// Convert binary to base64 and wrap in JSON
function sendAudioAsBase64(audioBlob) {
    const reader = new FileReader();

    reader.onloadend = function() {
        const base64Data = reader.result.split(",")[1];   // Remove data URL prefix

        ws.send(JSON.stringify({
            type: "audio_response",
            audio: base64Data,          // Base64-encoded audio
            duration: 5.3,              // Metadata alongside the audio
            format: "webm"
        }));
    };

    reader.readAsDataURL(audioBlob);
}
```

### Receiving and Playing Audio

```javascript
ws.binaryType = "arraybuffer";

ws.onmessage = async function(event) {
    if (event.data instanceof ArrayBuffer) {
        // Binary data — treat as audio
        const audioBlob = new Blob([event.data], { type: "audio/mpeg" });
        const audioUrl = URL.createObjectURL(audioBlob);
        const audio = new Audio(audioUrl);
        await audio.play();
    } else {
        // Text data — treat as JSON
        const msg = JSON.parse(event.data);
        handleControlMessage(msg);
    }
};
```

### Receiving Base64 Audio in JSON

```javascript
ws.onmessage = function(event) {
    const msg = JSON.parse(event.data);

    if (msg.type === "question" && msg.audio) {
        // Decode base64 audio and play it
        const audioBytes = atob(msg.audio);     // Decode base64 → binary string
        const arrayBuffer = new Uint8Array(audioBytes.length);
        for (let i = 0; i < audioBytes.length; i++) {
            arrayBuffer[i] = audioBytes.charCodeAt(i);
        }

        const blob = new Blob([arrayBuffer], { type: "audio/mpeg" });
        const url = URL.createObjectURL(blob);
        const audio = new Audio(url);
        audio.play();

        // Also display the text
        displayQuestion(msg.text, msg.question_number);
    }
};
```

---

## 12. Heartbeat / Keep-Alive

Some proxies, load balancers, or firewalls close idle WebSocket connections after 30-60 seconds. A heartbeat prevents this.

### Client-Side Heartbeat

```javascript
let ws;
let heartbeatInterval;
let missedHeartbeats = 0;
const HEARTBEAT_INTERVAL = 30000;   // 30 seconds
const MAX_MISSED = 3;

function connect() {
    ws = new WebSocket("ws://localhost:8000/ws");

    ws.onopen = function() {
        console.log("Connected!");
        startHeartbeat();
    };

    ws.onmessage = function(event) {
        const msg = JSON.parse(event.data);

        if (msg.type === "pong") {
            missedHeartbeats = 0;   // Server responded — connection is alive
            return;
        }

        // Handle other messages...
    };

    ws.onclose = function() {
        stopHeartbeat();
        // Reconnect logic...
    };
}

function startHeartbeat() {
    missedHeartbeats = 0;
    heartbeatInterval = setInterval(() => {
        if (missedHeartbeats >= MAX_MISSED) {
            console.log("Server not responding — closing connection");
            ws.close();
            return;
        }

        if (ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify({ type: "ping" }));
            missedHeartbeats++;
        }
    }, HEARTBEAT_INTERVAL);
}

function stopHeartbeat() {
    clearInterval(heartbeatInterval);
}
```

### Server-Side Heartbeat (Ping/Pong Frames)

WebSocket protocol has built-in Ping/Pong frames. Most servers send Ping frames automatically, and the browser responds with Pong automatically — you don't need to handle this in JavaScript. But the application-level heartbeat above is still useful for detecting logical disconnections.

---

## 13. Real-World Example: Chat Application

### HTML

```html
<div id="chat-container">
    <div id="messages"></div>
    <form id="message-form">
        <input type="text" id="message-input" placeholder="Type a message..." />
        <button type="submit">Send</button>
    </form>
    <div id="status">Connecting...</div>
</div>
```

### JavaScript

```javascript
const messagesDiv = document.getElementById("messages");
const messageForm = document.getElementById("message-form");
const messageInput = document.getElementById("message-input");
const statusDiv = document.getElementById("status");

let ws;
let username = prompt("Enter your name:");

function connect() {
    // Pass username as query parameter
    ws = new WebSocket(`ws://localhost:8000/ws/chat?username=${encodeURIComponent(username)}`);

    ws.onopen = function() {
        statusDiv.textContent = "Connected ✅";
        statusDiv.style.color = "green";
    };

    ws.onmessage = function(event) {
        const msg = JSON.parse(event.data);

        switch (msg.type) {
            case "chat_message":
                addMessage(msg.sender, msg.content, msg.sender === username);
                break;
            case "system":
                addSystemMessage(msg.content);
                break;
        }
    };

    ws.onclose = function(event) {
        statusDiv.textContent = "Disconnected ❌ — Reconnecting...";
        statusDiv.style.color = "red";
        setTimeout(connect, 3000);
    };
}

messageForm.addEventListener("submit", function(e) {
    e.preventDefault();
    const content = messageInput.value.trim();
    if (!content) return;

    ws.send(JSON.stringify({
        type: "chat_message",
        content: content
    }));

    messageInput.value = "";
});

function addMessage(sender, content, isMe) {
    const div = document.createElement("div");
    div.className = `message ${isMe ? "sent" : "received"}`;
    div.innerHTML = `<strong>${sender}:</strong> ${escapeHtml(content)}`;
    messagesDiv.appendChild(div);
    messagesDiv.scrollTop = messagesDiv.scrollHeight;   // Auto-scroll
}

function addSystemMessage(content) {
    const div = document.createElement("div");
    div.className = "message system";
    div.textContent = content;
    messagesDiv.appendChild(div);
}

// Prevent XSS!
function escapeHtml(text) {
    const div = document.createElement("div");
    div.textContent = text;
    return div.innerHTML;
}

connect();
```

---

## 14. Real-World Example: Live Audio Streaming

### Recording Audio and Sending via WebSocket

```javascript
let ws;
let mediaRecorder;
let audioChunks = [];

async function startInterview() {
    // Connect WebSocket
    ws = new WebSocket("ws://localhost:8000/ws/interview/token123");

    // Get microphone access
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });

    // Create MediaRecorder
    mediaRecorder = new MediaRecorder(stream, {
        mimeType: "audio/webm;codecs=opus"   // WebM format with Opus codec
    });

    // Collect audio chunks as they're recorded
    mediaRecorder.ondataavailable = function(event) {
        if (event.data.size > 0) {
            audioChunks.push(event.data);
        }
    };

    // When recording stops, send the audio
    mediaRecorder.onstop = function() {
        const audioBlob = new Blob(audioChunks, { type: "audio/webm" });
        audioChunks = [];   // Reset for next recording

        sendAudio(audioBlob);
    };

    ws.onopen = function() {
        console.log("Interview WebSocket connected");
    };

    ws.onmessage = function(event) {
        const msg = JSON.parse(event.data);

        switch (msg.type) {
            case "question":
                displayQuestion(msg.text);
                if (msg.audio) playAudio(msg.audio);
                // Start recording after question is asked
                setTimeout(() => startRecording(), 500);
                break;

            case "interview_complete":
                showCompletion(msg.text);
                break;
        }
    };
}

function startRecording() {
    audioChunks = [];
    mediaRecorder.start();
    console.log("🎙️ Recording...");
}

function stopRecording() {
    if (mediaRecorder.state === "recording") {
        mediaRecorder.stop();   // Triggers onstop → sends audio
        console.log("⏹️ Stopped recording");
    }
}

function sendAudio(blob) {
    const reader = new FileReader();
    reader.onloadend = function() {
        const base64 = reader.result.split(",")[1];
        ws.send(JSON.stringify({
            type: "audio_response",
            audio: base64
        }));
    };
    reader.readAsDataURL(blob);
}

async function playAudio(base64Audio) {
    const binaryString = atob(base64Audio);
    const bytes = new Uint8Array(binaryString.length);
    for (let i = 0; i < binaryString.length; i++) {
        bytes[i] = binaryString.charCodeAt(i);
    }
    const blob = new Blob([bytes], { type: "audio/mpeg" });
    const url = URL.createObjectURL(blob);
    const audio = new Audio(url);
    return audio.play();
}
```

---

## 15. Real-World Example: Real-Time Notifications

### Notification System

```javascript
class NotificationSocket {
    constructor(userId) {
        this.userId = userId;
        this.ws = null;
        this.handlers = new Map();
        this.connect();
    }

    connect() {
        this.ws = new WebSocket(`wss://api.example.com/ws/notifications?user=${this.userId}`);

        this.ws.onopen = () => {
            console.log("Notification socket connected");
        };

        this.ws.onmessage = (event) => {
            const notification = JSON.parse(event.data);
            this.dispatch(notification);
        };

        this.ws.onclose = (event) => {
            if (event.code !== 1000) {
                setTimeout(() => this.connect(), 5000);
            }
        };
    }

    // Register handlers for different notification types
    on(type, callback) {
        if (!this.handlers.has(type)) {
            this.handlers.set(type, []);
        }
        this.handlers.get(type).push(callback);
    }

    dispatch(notification) {
        const callbacks = this.handlers.get(notification.type) || [];
        callbacks.forEach(cb => cb(notification));

        // Also dispatch to wildcard handlers
        const wildcardCallbacks = this.handlers.get("*") || [];
        wildcardCallbacks.forEach(cb => cb(notification));
    }

    close() {
        this.ws.close(1000, "User logged out");
    }
}

// Usage:
const notifier = new NotificationSocket("user-123");

notifier.on("new_message", (data) => {
    showToast(`New message from ${data.sender}`);
    updateBadge(data.unread_count);
});

notifier.on("order_status", (data) => {
    showToast(`Order ${data.order_id}: ${data.status}`);
});

notifier.on("*", (data) => {
    console.log("Any notification:", data);
});
```

---

## 16. Working with MediaRecorder (Audio/Video)

MediaRecorder is often used alongside WebSockets for real-time audio/video.

### Audio Recording with Silence Detection

```javascript
let audioContext;
let analyser;
let mediaRecorder;
let silenceTimer;
const SILENCE_THRESHOLD = 10;       // Volume level (0-255)
const SILENCE_DURATION = 3000;       // 3 seconds of silence = stop

async function setupAudioRecording() {
    const stream = await navigator.mediaDevices.getUserMedia({
        audio: {
            echoCancellation: true,
            noiseSuppression: true,
            sampleRate: 44100
        }
    });

    // Set up Web Audio API for silence detection
    audioContext = new AudioContext();
    const source = audioContext.createMediaStreamSource(stream);
    analyser = audioContext.createAnalyser();
    analyser.fftSize = 256;
    source.connect(analyser);

    // Set up MediaRecorder for recording
    mediaRecorder = new MediaRecorder(stream, {
        mimeType: "audio/webm;codecs=opus"
    });

    return stream;
}

function startRecordingWithSilenceDetection() {
    const chunks = [];

    mediaRecorder.ondataavailable = (e) => {
        if (e.data.size > 0) chunks.push(e.data);
    };

    mediaRecorder.onstop = () => {
        const blob = new Blob(chunks, { type: "audio/webm" });
        sendAudioToServer(blob);
    };

    mediaRecorder.start();
    detectSilence();
}

function detectSilence() {
    const dataArray = new Uint8Array(analyser.frequencyBinCount);

    function check() {
        if (mediaRecorder.state !== "recording") return;

        analyser.getByteFrequencyData(dataArray);
        const average = dataArray.reduce((a, b) => a + b) / dataArray.length;

        if (average < SILENCE_THRESHOLD) {
            // Silence detected
            if (!silenceTimer) {
                silenceTimer = setTimeout(() => {
                    console.log("3 seconds of silence — stopping recording");
                    mediaRecorder.stop();
                }, SILENCE_DURATION);
            }
        } else {
            // Sound detected — reset timer
            if (silenceTimer) {
                clearTimeout(silenceTimer);
                silenceTimer = null;
            }
        }

        requestAnimationFrame(check);
    }

    check();
}
```

### Audio Level Visualization

```javascript
function visualizeAudioLevel(analyser, canvasElement) {
    const canvas = canvasElement;
    const ctx = canvas.getContext("2d");
    const dataArray = new Uint8Array(analyser.frequencyBinCount);

    function draw() {
        requestAnimationFrame(draw);
        analyser.getByteFrequencyData(dataArray);

        const average = dataArray.reduce((a, b) => a + b) / dataArray.length;
        const level = average / 255;   // Normalize to 0-1

        ctx.clearRect(0, 0, canvas.width, canvas.height);

        // Draw level bar
        const barWidth = canvas.width * level;
        ctx.fillStyle = level > 0.7 ? "red" : level > 0.3 ? "orange" : "green";
        ctx.fillRect(0, 0, barWidth, canvas.height);
    }

    draw();
}
```

---

## 17. Security Considerations

### Always Use WSS (Encrypted)

```javascript
// ❌ Insecure — data is plaintext
const ws = new WebSocket("ws://example.com/ws");

// ✅ Secure — data is encrypted (TLS)
const ws = new WebSocket("wss://example.com/ws");
```

### Authentication via Token

WebSocket doesn't support custom headers in the browser. Common approaches:

```javascript
// Method 1: Token in URL (simple, common)
const token = "eyJhbGciOiJIUzI1Ni...";
const ws = new WebSocket(`wss://api.example.com/ws?token=${token}`);

// Method 2: Token in first message
const ws = new WebSocket("wss://api.example.com/ws");
ws.onopen = function() {
    ws.send(JSON.stringify({
        type: "authenticate",
        token: "eyJhbGciOiJIUzI1Ni..."
    }));
};

// Method 3: Cookie-based (automatic if same domain)
// The browser sends cookies automatically during the WebSocket handshake
// No extra code needed if your auth cookie is set
```

### Validate All Incoming Messages

```javascript
ws.onmessage = function(event) {
    let msg;
    try {
        msg = JSON.parse(event.data);
    } catch (e) {
        console.error("Invalid JSON received");
        return;   // Ignore malformed messages
    }

    if (!msg.type) {
        console.error("Message missing type field");
        return;
    }

    // Only handle known message types
    const handler = handlers[msg.type];
    if (handler) {
        handler(msg);
    }
};
```

### Escape User Content (Prevent XSS)

```javascript
// ❌ NEVER do this — XSS vulnerability!
messagesDiv.innerHTML += `<p>${msg.content}</p>`;

// ✅ Use textContent (auto-escapes HTML)
const p = document.createElement("p");
p.textContent = msg.content;
messagesDiv.appendChild(p);

// ✅ Or use a sanitizer
function escapeHtml(text) {
    const div = document.createElement("div");
    div.textContent = text;
    return div.innerHTML;
}
messagesDiv.innerHTML += `<p>${escapeHtml(msg.content)}</p>`;
```

---

## 18. Debugging WebSockets

### Browser DevTools — Network Tab

1. Open DevTools (`F12`)
2. Go to **Network** tab
3. Filter by **WS** (WebSocket)
4. Click on your WebSocket connection
5. Go to **Messages** tab — see ALL sent/received messages in real-time

```
▲ {"type":"chat_message","content":"Hello"}        ← Sent (green arrow up)
▼ {"type":"chat_message","sender":"Alice","content":"Hi!"}  ← Received (red arrow down)
▼ {"type":"user_joined","username":"Bob"}           ← Received
▲ {"type":"typing_start"}                           ← Sent
```

### Console Debugging

```javascript
// Wrap WebSocket for debugging
const originalSend = WebSocket.prototype.send;
WebSocket.prototype.send = function(data) {
    console.log("📤 SENT:", data);
    originalSend.call(this, data);
};

// Or debug a specific connection:
const realOnMessage = ws.onmessage;
ws.onmessage = function(event) {
    console.log("📩 RECEIVED:", event.data);
    realOnMessage.call(this, event);
};
```

### Testing Without a Server

```javascript
// Use a public WebSocket echo server for testing
const ws = new WebSocket("wss://echo.websocket.org");

ws.onopen = () => ws.send("Hello!");
ws.onmessage = (e) => console.log("Echo:", e.data);
// Server echoes back whatever you send
```

---

## 19. Common Pitfalls & Solutions

### Pitfall 1: Sending Before Connected

```javascript
// ❌ This fails — WebSocket isn't connected yet
const ws = new WebSocket("ws://localhost:8000/ws");
ws.send("Hello!");   // Error: INVALID_STATE_ERR

// ✅ Wait for onopen
const ws = new WebSocket("ws://localhost:8000/ws");
ws.onopen = () => ws.send("Hello!");
```

### Pitfall 2: Forgetting to JSON.stringify

```javascript
// ❌ Sends "[object Object]" as string
ws.send({ type: "message", content: "Hello" });

// ✅ Stringify first
ws.send(JSON.stringify({ type: "message", content: "Hello" }));
```

### Pitfall 3: Not Handling Page Unload

```javascript
// Clean up when user leaves the page
window.addEventListener("beforeunload", () => {
    if (ws && ws.readyState === WebSocket.OPEN) {
        ws.close(1000, "Page closing");
    }
});
```

### Pitfall 4: Memory Leaks with Event Listeners

```javascript
// ❌ Creates a new WebSocket without cleaning up the old one
function reconnect() {
    ws = new WebSocket("ws://localhost:8000/ws");
    ws.onmessage = handleMessage;
    // Old WebSocket and its listeners are still in memory!
}

// ✅ Clean up old connection first
function reconnect() {
    if (ws) {
        ws.onclose = null;   // Prevent reconnection loop
        ws.close();
    }
    ws = new WebSocket("ws://localhost:8000/ws");
    ws.onmessage = handleMessage;
}
```

### Pitfall 5: Mixing ws:// and https://

```javascript
// ❌ Browsers block ws:// on https:// pages (mixed content)
// If your page is https://mysite.com, you MUST use wss://

// ✅ Auto-detect protocol
const protocol = window.location.protocol === "https:" ? "wss:" : "ws:";
const ws = new WebSocket(`${protocol}//${window.location.host}/ws`);
```

---

## 20. Mental Model & Quick Reference

### The Mental Model

```
Browser                                    Server
┌─────────────────────────┐     ┌─────────────────────────┐
│                         │     │                         │
│  const ws = new         │     │                         │
│    WebSocket(url)       │─────│  accept() connection    │
│                         │     │                         │
│  ws.send(data) ─────────│────►│  receive(data)          │
│                         │     │                         │
│  ws.onmessage ◄─────────│────│  send(data)             │
│                         │     │                         │
│  ws.close() ────────────│────►│  connection closed      │
│                         │     │                         │
└─────────────────────────┘     └─────────────────────────┘
```

### Quick Reference

| Action | Code |
|--------|------|
| Connect | `const ws = new WebSocket("wss://host/path")` |
| On connect | `ws.onopen = () => { ... }` |
| On message | `ws.onmessage = (e) => { ... }` |
| On close | `ws.onclose = (e) => { ... }` |
| On error | `ws.onerror = (e) => { ... }` |
| Send text | `ws.send("hello")` |
| Send JSON | `ws.send(JSON.stringify({...}))` |
| Send binary | `ws.send(arrayBuffer)` |
| Close | `ws.close(1000, "reason")` |
| Check state | `ws.readyState === WebSocket.OPEN` |
| Binary mode | `ws.binaryType = "arraybuffer"` |

### When to Use WebSockets

| Use Case | WebSocket? | Why |
|----------|-----------|-----|
| Chat/messaging | ✅ | Real-time bidirectional |
| Live audio/video | ✅ | Streaming binary data |
| Gaming | ✅ | Low-latency state sync |
| Live dashboard | ✅ | Server pushes updates |
| Notifications | ✅ | Server pushes instantly |
| File upload | ❌ | Use HTTP (better progress, resume support) |
| REST API | ❌ | HTTP is simpler and cacheable |
| One-time data fetch | ❌ | HTTP is sufficient |

---

*WebSockets give you a persistent, bidirectional pipe between browser and server. The browser API is simple — just 4 events (`open`, `message`, `close`, `error`) and 2 methods (`send`, `close`). The complexity is in what you build on top: reconnection, message protocols, binary handling, and error recovery.*
