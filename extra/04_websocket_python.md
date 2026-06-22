# WebSocket — Server-Side (Python / FastAPI) Complete Learning Notes

---

## Table of Contents

1. [What Happens on the Server Side?](#1-what-happens-on-the-server-side)
2. [WebSocket in Python — The Options](#2-websocket-in-python--the-options)
3. [FastAPI WebSocket — Getting Started](#3-fastapi-websocket--getting-started)
4. [The WebSocket Object — Methods & Properties](#4-the-websocket-object--methods--properties)
5. [Accepting and Rejecting Connections](#5-accepting-and-rejecting-connections)
6. [Receiving Messages](#6-receiving-messages)
7. [Sending Messages](#7-sending-messages)
8. [The Connection Loop Pattern](#8-the-connection-loop-pattern)
9. [JSON Messaging Protocol](#9-json-messaging-protocol)
10. [Handling Disconnections Gracefully](#10-handling-disconnections-gracefully)
11. [Authentication & Authorization](#11-authentication--authorization)
12. [Connection Manager (Multiple Clients)](#12-connection-manager-multiple-clients)
13. [Rooms / Groups / Channels](#13-rooms--groups--channels)
14. [Broadcasting Messages](#14-broadcasting-messages)
15. [Sending Binary Data (Audio, Files)](#15-sending-binary-data-audio-files)
16. [Background Tasks During WebSocket](#16-background-tasks-during-websocket)
17. [WebSocket with Database Sessions](#17-websocket-with-database-sessions)
18. [Real-World Example: Chat Server](#18-real-world-example-chat-server)
19. [Real-World Example: AI Interview (Audio Streaming)](#19-real-world-example-ai-interview-audio-streaming)
20. [Testing WebSockets](#20-testing-websockets)
21. [Scaling WebSockets (Multiple Workers)](#21-scaling-websockets-multiple-workers)
22. [Common Pitfalls & Solutions](#22-common-pitfalls--solutions)
23. [Starlette WebSocket (Under the Hood)](#23-starlette-websocket-under-the-hood)
24. [Quick Reference & Mental Model](#24-quick-reference--mental-model)

---

## 1. What Happens on the Server Side?

When a client (browser) connects via WebSocket, the server needs to:

1. **Accept** the connection (upgrade from HTTP)
2. **Listen** for incoming messages in a loop
3. **Process** each message (parse JSON, run logic)
4. **Send** responses back to the client
5. **Handle disconnection** (clean up resources)

```
Browser                          Server
  │                                │
  │── HTTP GET /ws (upgrade) ────►│
  │                                │── accept()
  │◄─── 101 Switching Protocols ──│
  │                                │
  │════ WebSocket Connected ═══════│
  │                                │
  │── send("Hello") ──────────────►│── receive_text()
  │                                │── process message
  │◄────────────── send("Hi!") ───│── send_text()
  │                                │
  │── send("Bye") ────────────────►│── receive_text()
  │◄────── close(1000) ──────────│── close()
  │                                │
```

---

## 2. WebSocket in Python — The Options

| Library | Use Case | Async? |
|---------|----------|--------|
| **FastAPI** (Starlette) | Full web framework + WebSocket | ✅ |
| **websockets** | Standalone WebSocket server | ✅ |
| **Socket.IO (python-socketio)** | Event-based, auto-reconnect, rooms | ✅ |
| **Django Channels** | Django + WebSocket | ✅ |
| **Tornado** | Older async framework | ✅ |

**FastAPI is the most popular choice** for new Python projects because:
- WebSocket support is built into Starlette (which FastAPI wraps)
- You get REST + WebSocket in the same app
- Full async/await support
- Type hints and auto-documentation
- Same dependency injection system for both HTTP and WS

---

## 3. FastAPI WebSocket — Getting Started

### Installation

```bash
pip install fastapi uvicorn
```

### The Simplest WebSocket Server

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # Step 1: Accept the connection
    await websocket.accept()

    # Step 2: Communication loop
    while True:
        # Wait for a message from the client
        data = await websocket.receive_text()
        
        # Send a response back
        await websocket.send_text(f"You said: {data}")

# Run: uvicorn main:app --host 0.0.0.0 --port 8000
```

**That's it!** A working WebSocket server in 10 lines.

### Testing with JavaScript

```javascript
const ws = new WebSocket("ws://localhost:8000/ws");
ws.onopen = () => ws.send("Hello!");
ws.onmessage = (e) => console.log(e.data);  // "You said: Hello!"
```

### Path Parameters

```python
@app.websocket("/ws/chat/{room_name}")
async def chat_room(websocket: WebSocket, room_name: str):
    await websocket.accept()
    print(f"Client joined room: {room_name}")
    
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"[{room_name}] {data}")

# Client connects to: ws://localhost:8000/ws/chat/general
```

### Query Parameters

```python
@app.websocket("/ws")
async def websocket_with_params(websocket: WebSocket):
    # Access query parameters from the URL
    token = websocket.query_params.get("token")
    username = websocket.query_params.get("username", "Anonymous")
    
    if not token:
        await websocket.close(code=4001, reason="Token required")
        return
    
    await websocket.accept()
    await websocket.send_text(f"Welcome, {username}!")

# Client connects to: ws://localhost:8000/ws?token=abc123&username=Ritesh
```

---

## 4. The WebSocket Object — Methods & Properties

### Key Methods

```python
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    
    # ═══ Connection Methods ═══
    await websocket.accept()                            # Accept the connection
    await websocket.close(code=1000, reason="Done")     # Close the connection
    
    # ═══ Receiving Methods ═══
    text = await websocket.receive_text()               # Receive text message
    data = await websocket.receive_bytes()               # Receive binary message
    msg  = await websocket.receive_json()                # Receive & parse JSON
    raw  = await websocket.receive()                     # Raw ASGI message dict
    
    # ═══ Sending Methods ═══
    await websocket.send_text("hello")                   # Send text
    await websocket.send_bytes(b"\x00\x01\x02")         # Send binary
    await websocket.send_json({"key": "value"})          # Send JSON (auto-serializes)
```

### Key Properties

```python
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    # URL info
    print(websocket.url)              # ws://localhost:8000/ws?token=abc
    print(websocket.url.path)         # /ws
    print(websocket.url.scheme)       # ws

    # Query parameters
    print(websocket.query_params)     # QueryParams({'token': 'abc'})
    print(websocket.query_params["token"])   # 'abc'

    # Path parameters (from URL pattern)
    print(websocket.path_params)      # {'room_name': 'general'}

    # Headers (from the initial HTTP upgrade request)
    print(websocket.headers)          # Headers({'host': 'localhost:8000', ...})
    print(websocket.headers.get("user-agent"))

    # Cookies
    print(websocket.cookies)          # {'session_id': 'abc123'}

    # Client info
    print(websocket.client.host)      # '127.0.0.1'
    print(websocket.client.port)      # 54321

    # Application state
    print(websocket.state)            # State object for storing per-connection data
    websocket.state.username = "Ritesh"   # ← You can store anything here!
```

---

## 5. Accepting and Rejecting Connections

### Simple Accept

```python
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    # Connection is now open
```

### Accept with Subprotocol

```python
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    # Client may request a subprotocol (e.g., "graphql-ws")
    requested = websocket.headers.get("sec-websocket-protocol")
    
    await websocket.accept(subprotocol="graphql-ws")
```

### Rejecting a Connection

```python
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    token = websocket.query_params.get("token")
    
    if not token or not is_valid_token(token):
        # Close immediately with a custom code
        await websocket.close(code=4003, reason="Invalid token")
        return   # ← Important! Stop execution after closing
    
    await websocket.accept()
    # ... handle valid connection
```

### Rejecting with HTTP Response (Before Upgrade)

```python
from fastapi import WebSocket, WebSocketException, status

@app.websocket("/ws")
async def handler(websocket: WebSocket):
    token = websocket.query_params.get("token")
    
    if not token:
        # This sends an HTTP 403 BEFORE upgrading to WebSocket
        raise WebSocketException(code=status.WS_1008_POLICY_VIOLATION)
    
    await websocket.accept()
```

---

## 6. Receiving Messages

### Receive Text

```python
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    
    # This BLOCKS until a message arrives
    text = await websocket.receive_text()
    print(f"Received: {text}")
```

### Receive JSON

```python
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    
    # Automatically parses JSON string → Python dict
    data = await websocket.receive_json()
    print(data["type"])      # e.g., "chat_message"
    print(data["content"])   # e.g., "Hello!"
```

### Receive Binary

```python
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    
    # Receive raw bytes (e.g., audio, images)
    audio_bytes = await websocket.receive_bytes()
    print(f"Received {len(audio_bytes)} bytes of audio")
```

### Receive Raw (Mixed Text + Binary)

```python
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    
    while True:
        # Raw receive — returns a dict with either 'text' or 'bytes'
        message = await websocket.receive()
        
        if "text" in message:
            data = json.loads(message["text"])
            await handle_text_message(data)
        elif "bytes" in message:
            await handle_binary_message(message["bytes"])
        elif message.get("type") == "websocket.disconnect":
            break
```

---

## 7. Sending Messages

### Send Text

```python
await websocket.send_text("Hello, client!")
```

### Send JSON

```python
# Automatically converts dict → JSON string
await websocket.send_json({
    "type": "question",
    "text": "What is Python's GIL?",
    "question_number": 5
})

# Same as:
import json
await websocket.send_text(json.dumps({
    "type": "question",
    "text": "What is Python's GIL?",
    "question_number": 5
}))
```

### Send Binary

```python
# Send raw bytes (e.g., audio file)
audio_data = open("question.mp3", "rb").read()
await websocket.send_bytes(audio_data)
```

### Send JSON + Binary Together

A common pattern: send metadata as JSON, then send binary data separately:

```python
# First: send metadata
await websocket.send_json({
    "type": "audio_question",
    "text": "Explain Python decorators.",
    "question_number": 3,
    "audio_format": "mp3",
    "audio_size": len(audio_bytes)
})

# Then: send audio as binary
await websocket.send_bytes(audio_bytes)
```

Or encode binary as base64 inside JSON (simpler but larger):

```python
import base64

audio_bytes = synthesize_speech("Explain Python decorators.")
audio_base64 = base64.b64encode(audio_bytes).decode("utf-8")

await websocket.send_json({
    "type": "question",
    "text": "Explain Python decorators.",
    "audio": audio_base64,         # Base64-encoded audio inside JSON
    "question_number": 3
})
```

---

## 8. The Connection Loop Pattern

Every WebSocket server needs a **message loop** — an infinite loop that continuously listens for messages.

### Basic Loop

```python
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            data = await websocket.receive_json()
            
            # Process the message
            response = process_message(data)
            
            # Send response
            await websocket.send_json(response)
            
    except WebSocketDisconnect:
        print("Client disconnected")
```

### Loop with Message Type Routing

```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            message = await websocket.receive_json()
            msg_type = message.get("type")
            
            if msg_type == "chat_message":
                await handle_chat(websocket, message)
            elif msg_type == "typing_start":
                await handle_typing(websocket, message)
            elif msg_type == "join_room":
                await handle_join(websocket, message)
            elif msg_type == "ping":
                await websocket.send_json({"type": "pong"})
            else:
                await websocket.send_json({
                    "type": "error",
                    "message": f"Unknown message type: {msg_type}"
                })
                
    except WebSocketDisconnect as e:
        print(f"Client disconnected: code={e.code}, reason={e.reason}")
    except Exception as e:
        print(f"Error: {e}")
        await websocket.close(code=1011, reason="Internal error")
```

### Handler Map Pattern (Cleaner)

```python
class WebSocketHandler:
    def __init__(self, websocket: WebSocket):
        self.ws = websocket
        self.handlers = {
            "chat_message": self.handle_chat,
            "typing_start": self.handle_typing,
            "join_room": self.handle_join,
            "ping": self.handle_ping,
        }
    
    async def run(self):
        await self.ws.accept()
        
        try:
            while True:
                message = await self.ws.receive_json()
                handler = self.handlers.get(message.get("type"))
                
                if handler:
                    await handler(message)
                else:
                    await self.ws.send_json({
                        "type": "error",
                        "message": f"Unknown type: {message.get('type')}"
                    })
        except WebSocketDisconnect:
            await self.cleanup()
    
    async def handle_chat(self, msg):
        await self.ws.send_json({"type": "chat_message", "echo": msg["content"]})
    
    async def handle_typing(self, msg):
        pass  # Broadcast to others
    
    async def handle_join(self, msg):
        pass  # Add to room
    
    async def handle_ping(self, msg):
        await self.ws.send_json({"type": "pong"})
    
    async def cleanup(self):
        print("Client disconnected, cleaning up...")

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    handler = WebSocketHandler(websocket)
    await handler.run()
```

---

## 9. JSON Messaging Protocol

### Designing a Message Protocol

A well-designed protocol makes everything cleaner:

```python
# ═══ Client → Server messages ═══
# { "type": "...", ...fields }

INCOMING_MESSAGES = {
    "audio_response": {
        "audio": "base64 encoded audio bytes",
        "duration": 5.3
    },
    "pause": {},
    "resume": {},
    "anti_cheat_event": {
        "event_type": "tab_switch",
        "details": "..."
    },
    "webcam_frame": {
        "frame": "base64 JPEG"
    }
}

# ═══ Server → Client messages ═══
OUTGOING_MESSAGES = {
    "greeting": {"text": "...", "audio": "base64"},
    "question": {"text": "...", "audio": "base64", "question_number": 1},
    "follow_up": {"text": "...", "audio": "base64"},
    "processing": {"text": "Analyzing your response..."},
    "interview_complete": {"text": "...", "audio": "base64"},
    "error": {"text": "Something went wrong"},
}
```

### Using Pydantic for Message Validation

```python
from pydantic import BaseModel
from typing import Optional

class IncomingMessage(BaseModel):
    type: str

class AudioResponse(BaseModel):
    type: str = "audio_response"
    audio: str                # base64 encoded
    duration: Optional[float] = None

class AntiCheatEvent(BaseModel):
    type: str = "anti_cheat_event"
    event_type: str
    details: Optional[str] = None

# In your handler:
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    
    while True:
        raw = await websocket.receive_json()
        msg_type = raw.get("type")
        
        if msg_type == "audio_response":
            msg = AudioResponse(**raw)      # Validates the data!
            await process_audio(msg.audio)
            
        elif msg_type == "anti_cheat_event":
            msg = AntiCheatEvent(**raw)
            await log_anti_cheat(msg.event_type, msg.details)
```

### Sending Helper

```python
async def send_message(websocket: WebSocket, msg_type: str, **kwargs):
    """Standard way to send messages with consistent format."""
    await websocket.send_json({
        "type": msg_type,
        **kwargs
    })

# Usage:
await send_message(ws, "question",
    text="What is a decorator?",
    audio=audio_base64,
    question_number=5
)

await send_message(ws, "error", text="Session expired")

await send_message(ws, "processing", text="Analyzing your response...")
```

---

## 10. Handling Disconnections Gracefully

### WebSocketDisconnect Exception

When a client disconnects (closes tab, network drops), the `await websocket.receive_*()` call raises `WebSocketDisconnect`.

```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
            
    except WebSocketDisconnect as e:
        # e.code = close code (e.g., 1000, 1001, 1006)
        # e.reason = close reason string
        print(f"Client disconnected: code={e.code}")
        
        # Clean up resources
        await cleanup_session(websocket)
```

### Detecting Disconnect When Sending

```python
async def send_safe(websocket: WebSocket, data: dict) -> bool:
    """Send a message, return False if client disconnected."""
    try:
        await websocket.send_json(data)
        return True
    except Exception:
        return False

# Usage:
if not await send_safe(ws, {"type": "question", "text": "..."}):
    print("Client gone, aborting")
    return
```

### Cleanup Pattern

```python
@app.websocket("/ws/interview/{token}")
async def interview_handler(websocket: WebSocket, token: str):
    session = None
    
    try:
        # Setup
        session = await validate_and_get_session(token)
        await websocket.accept()
        
        # Main loop
        while True:
            data = await websocket.receive_json()
            await process_message(websocket, session, data)
            
    except WebSocketDisconnect:
        print("Client disconnected")
    except Exception as e:
        print(f"Error: {e}")
        try:
            await websocket.send_json({"type": "error", "text": str(e)})
            await websocket.close(code=1011)
        except:
            pass  # Client already gone
    finally:
        # ALWAYS runs — even after exceptions
        if session:
            await save_session_state(session)
            await update_status(session, "interrupted")
        print("Handler cleaned up")
```

---

## 11. Authentication & Authorization

### Token in Query Parameter

```python
from src.utility.token_utils import verify_token

@app.websocket("/ws/interview/{token}")
async def interview_ws(websocket: WebSocket, token: str):
    # Validate token BEFORE accepting
    try:
        payload = verify_token(token)
        session_id = payload["session_id"]
    except Exception:
        await websocket.close(code=4001, reason="Invalid or expired token")
        return
    
    # Token valid — accept connection
    await websocket.accept()
    
    # Store user info on the connection
    websocket.state.session_id = session_id
    
    # ... handle interview
```

### Cookie-Based Auth

```python
@app.websocket("/ws/dashboard")
async def dashboard_ws(websocket: WebSocket):
    # Cookies are sent automatically during WebSocket handshake
    session_cookie = websocket.cookies.get("coordinator_session")
    
    if not session_cookie or not verify_coordinator_session(session_cookie):
        await websocket.close(code=4003, reason="Not authenticated")
        return
    
    await websocket.accept()
    # ... handle dashboard updates
```

### Token in First Message

```python
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    
    # First message MUST be authentication
    try:
        auth_msg = await asyncio.wait_for(
            websocket.receive_json(),
            timeout=10.0  # 10 second timeout for auth
        )
    except asyncio.TimeoutError:
        await websocket.close(code=4000, reason="Auth timeout")
        return
    
    if auth_msg.get("type") != "authenticate" or not auth_msg.get("token"):
        await websocket.close(code=4001, reason="Must authenticate first")
        return
    
    if not verify_token(auth_msg["token"]):
        await websocket.close(code=4003, reason="Invalid token")
        return
    
    # Authenticated — proceed to main loop
    await websocket.send_json({"type": "authenticated"})
    
    while True:
        data = await websocket.receive_json()
        # ... handle messages
```

---

## 12. Connection Manager (Multiple Clients)

When you have multiple clients connected, you need a way to manage them.

### Basic Connection Manager

```python
from fastapi import WebSocket
from typing import List

class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []
    
    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)
        print(f"Client connected. Total: {len(self.active_connections)}")
    
    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)
        print(f"Client disconnected. Total: {len(self.active_connections)}")
    
    async def send_personal(self, websocket: WebSocket, message: dict):
        await websocket.send_json(message)
    
    async def broadcast(self, message: dict):
        """Send a message to ALL connected clients."""
        for connection in self.active_connections:
            try:
                await connection.send_json(message)
            except:
                pass  # Client might have disconnected

# Create a single global instance
manager = ConnectionManager()

@app.websocket("/ws/chat")
async def chat_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    
    try:
        while True:
            data = await websocket.receive_json()
            
            # Broadcast to everyone
            await manager.broadcast({
                "type": "chat_message",
                "content": data["content"],
                "sender": data.get("sender", "Anonymous")
            })
            
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast({
            "type": "system",
            "content": "A user left the chat"
        })
```

### Connection Manager with User Tracking

```python
from fastapi import WebSocket
from typing import Dict

class ConnectionManager:
    def __init__(self):
        # Map: user_id → WebSocket
        self.connections: Dict[str, WebSocket] = {}
    
    async def connect(self, user_id: str, websocket: WebSocket):
        await websocket.accept()
        
        # Disconnect previous session for this user (single session)
        if user_id in self.connections:
            old_ws = self.connections[user_id]
            try:
                await old_ws.close(code=4000, reason="Connected from another device")
            except:
                pass
        
        self.connections[user_id] = websocket
    
    def disconnect(self, user_id: str):
        self.connections.pop(user_id, None)
    
    async def send_to_user(self, user_id: str, message: dict):
        ws = self.connections.get(user_id)
        if ws:
            try:
                await ws.send_json(message)
            except:
                self.disconnect(user_id)
    
    async def broadcast(self, message: dict, exclude: str = None):
        for user_id, ws in list(self.connections.items()):
            if user_id == exclude:
                continue
            try:
                await ws.send_json(message)
            except:
                self.disconnect(user_id)
    
    def is_online(self, user_id: str) -> bool:
        return user_id in self.connections
    
    def online_users(self) -> list:
        return list(self.connections.keys())
```

---

## 13. Rooms / Groups / Channels

For chat applications, gaming, etc., you often need to group connections.

```python
from collections import defaultdict
from fastapi import WebSocket
from typing import Dict, Set

class RoomManager:
    def __init__(self):
        # room_name → set of WebSockets
        self.rooms: Dict[str, Set[WebSocket]] = defaultdict(set)
        # WebSocket → set of rooms
        self.user_rooms: Dict[WebSocket, Set[str]] = defaultdict(set)
    
    async def join_room(self, room: str, websocket: WebSocket):
        self.rooms[room].add(websocket)
        self.user_rooms[websocket].add(room)
        
        await self.broadcast_to_room(room, {
            "type": "system",
            "content": f"A user joined {room}"
        }, exclude=websocket)
    
    async def leave_room(self, room: str, websocket: WebSocket):
        self.rooms[room].discard(websocket)
        self.user_rooms[websocket].discard(room)
        
        if not self.rooms[room]:  # Clean up empty rooms
            del self.rooms[room]
        
        await self.broadcast_to_room(room, {
            "type": "system",
            "content": f"A user left {room}"
        })
    
    async def disconnect(self, websocket: WebSocket):
        """Remove from all rooms on disconnect."""
        rooms = list(self.user_rooms.get(websocket, set()))
        for room in rooms:
            await self.leave_room(room, websocket)
        self.user_rooms.pop(websocket, None)
    
    async def broadcast_to_room(self, room: str, message: dict,
                                 exclude: WebSocket = None):
        for ws in list(self.rooms.get(room, set())):
            if ws == exclude:
                continue
            try:
                await ws.send_json(message)
            except:
                self.rooms[room].discard(ws)


room_manager = RoomManager()

@app.websocket("/ws/chat")
async def chat_ws(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            msg = await websocket.receive_json()
            
            if msg["type"] == "join_room":
                await room_manager.join_room(msg["room"], websocket)
                
            elif msg["type"] == "leave_room":
                await room_manager.leave_room(msg["room"], websocket)
                
            elif msg["type"] == "chat_message":
                await room_manager.broadcast_to_room(msg["room"], {
                    "type": "chat_message",
                    "content": msg["content"],
                    "sender": msg.get("sender"),
                    "room": msg["room"]
                }, exclude=websocket)
                
    except WebSocketDisconnect:
        await room_manager.disconnect(websocket)
```

---

## 14. Broadcasting Messages

### Broadcast from HTTP Route

Sometimes you need to push WebSocket messages from a regular HTTP endpoint:

```python
manager = ConnectionManager()

@app.websocket("/ws")
async def ws_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            await websocket.receive_text()   # Keep connection alive
    except WebSocketDisconnect:
        manager.disconnect(websocket)

# HTTP endpoint that pushes to all WebSocket clients
@app.post("/api/notify")
async def send_notification(message: str):
    await manager.broadcast({
        "type": "notification",
        "content": message
    })
    return {"status": "sent", "clients": len(manager.active_connections)}
```

### Broadcast to Specific User from HTTP

```python
@app.post("/api/notify/{user_id}")
async def notify_user(user_id: str, message: str):
    if manager.is_online(user_id):
        await manager.send_to_user(user_id, {
            "type": "notification",
            "content": message
        })
        return {"status": "sent"}
    else:
        return {"status": "user offline"}
```

---

## 15. Sending Binary Data (Audio, Files)

### Receiving Audio from Client

```python
import base64

@app.websocket("/ws/interview/{token}")
async def interview_ws(websocket: WebSocket, token: str):
    await websocket.accept()
    
    while True:
        msg = await websocket.receive_json()
        
        if msg["type"] == "audio_response":
            # Decode base64 audio
            audio_base64 = msg["audio"]
            audio_bytes = base64.b64decode(audio_base64)
            
            # Save to file
            with open("response.webm", "wb") as f:
                f.write(audio_bytes)
            
            # Or process directly (e.g., transcribe)
            transcript = await transcribe_audio(audio_bytes)
            
            await websocket.send_json({
                "type": "transcript",
                "text": transcript
            })
```

### Sending Audio to Client

```python
import base64

async def send_question_with_audio(websocket: WebSocket, text: str, question_number: int):
    # Generate TTS audio
    audio_bytes = await text_to_speech(text)
    audio_base64 = base64.b64encode(audio_bytes).decode("utf-8")
    
    await websocket.send_json({
        "type": "question",
        "text": text,
        "audio": audio_base64,
        "question_number": question_number
    })
```

### Sending Raw Binary

```python
# For large files, send binary directly (no JSON, no base64 overhead)
audio_bytes = await text_to_speech("Hello!")

# First send metadata as JSON
await websocket.send_json({
    "type": "audio_incoming",
    "format": "mp3",
    "size": len(audio_bytes)
})

# Then send binary data
await websocket.send_bytes(audio_bytes)
```

---

## 16. Background Tasks During WebSocket

### Running Async Tasks Concurrently

```python
import asyncio

@app.websocket("/ws/interview/{token}")
async def interview_ws(websocket: WebSocket, token: str):
    await websocket.accept()
    
    # Run message handler and timer concurrently
    try:
        await asyncio.gather(
            message_loop(websocket),
            session_timer(websocket, timeout=1800),   # 30 min timeout
        )
    except WebSocketDisconnect:
        print("Client disconnected")

async def message_loop(websocket: WebSocket):
    while True:
        data = await websocket.receive_json()
        await process_message(websocket, data)

async def session_timer(websocket: WebSocket, timeout: int):
    await asyncio.sleep(timeout)
    await websocket.send_json({
        "type": "session_expired",
        "text": "Your session has timed out."
    })
    await websocket.close(code=4008, reason="Session timeout")
```

### Processing Audio While Listening for New Messages

```python
import asyncio

@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    processing_queue = asyncio.Queue()
    
    async def receiver():
        """Continuously receive messages from client."""
        while True:
            msg = await websocket.receive_json()
            await processing_queue.put(msg)
    
    async def processor():
        """Process messages from the queue."""
        while True:
            msg = await processing_queue.get()
            
            if msg["type"] == "audio_response":
                # This takes time but doesn't block receiving
                await websocket.send_json({"type": "processing"})
                transcript = await transcribe_audio(msg["audio"])  # 2-3 seconds
                score = await score_answer(transcript)              # 1-2 seconds
                await websocket.send_json({
                    "type": "scored",
                    "transcript": transcript,
                    "score": score
                })
    
    try:
        await asyncio.gather(receiver(), processor())
    except WebSocketDisconnect:
        pass
```

### Fire-and-Forget Background Work

```python
import asyncio

@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    
    while True:
        msg = await websocket.receive_json()
        
        if msg["type"] == "audio_response":
            # Don't await — run in background
            asyncio.create_task(save_audio_to_drive(msg["audio"]))
            
            # Continue processing immediately
            transcript = await transcribe(msg["audio"])
            await websocket.send_json({"type": "transcript", "text": transcript})

async def save_audio_to_drive(audio_base64: str):
    """Runs in background — client doesn't wait for this."""
    audio_bytes = base64.b64decode(audio_base64)
    await upload_to_google_drive(audio_bytes)
    print("Audio saved to Drive")
```

---

## 17. WebSocket with Database Sessions

### The Challenge

HTTP routes create a new database session per request. But WebSocket connections are long-lived — you can't hold a database session open for 30 minutes.

### Pattern: Create Session Per Operation

```python
from src.database import SessionLocal

@app.websocket("/ws/interview/{token}")
async def interview_ws(websocket: WebSocket, token: str):
    await websocket.accept()
    
    try:
        while True:
            msg = await websocket.receive_json()
            
            if msg["type"] == "audio_response":
                # Create a DB session just for this operation
                db = SessionLocal()
                try:
                    response = QuestionResponse(
                        interview_id=session_id,
                        transcript=transcript,
                    )
                    db.add(response)
                    db.commit()
                finally:
                    db.close()   # Always close!
                    
    except WebSocketDisconnect:
        pass
```

### Pattern: Context Manager Helper

```python
from contextlib import contextmanager

@contextmanager
def get_db():
    db = SessionLocal()
    try:
        yield db
        db.commit()
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()

@app.websocket("/ws/interview/{token}")
async def interview_ws(websocket: WebSocket, token: str):
    await websocket.accept()
    
    while True:
        msg = await websocket.receive_json()
        
        if msg["type"] == "audio_response":
            with get_db() as db:
                save_response(db, session_id, transcript)
                # commit happens automatically
```

### Pattern: Async Session (if using async SQLAlchemy)

```python
from sqlalchemy.ext.asyncio import AsyncSession
from src.database import async_session

@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    
    while True:
        msg = await websocket.receive_json()
        
        async with async_session() as db:
            async with db.begin():
                # Do database operations
                result = await db.execute(select(User).where(User.id == 1))
                user = result.scalar_one()
            # Auto-commit on exit
```

---

## 18. Real-World Example: Chat Server

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, List
from datetime import datetime
import json

app = FastAPI()

class ChatServer:
    def __init__(self):
        self.connections: Dict[str, WebSocket] = {}  # username → ws
        self.message_history: List[dict] = []         # Last 50 messages
    
    async def connect(self, username: str, websocket: WebSocket):
        await websocket.accept()
        self.connections[username] = websocket
        
        # Send message history to new user
        await websocket.send_json({
            "type": "history",
            "messages": self.message_history[-50:]
        })
        
        # Announce to everyone
        await self.broadcast({
            "type": "system",
            "content": f"{username} joined the chat",
            "timestamp": datetime.utcnow().isoformat(),
            "online_users": list(self.connections.keys())
        })
    
    async def disconnect(self, username: str):
        self.connections.pop(username, None)
        await self.broadcast({
            "type": "system",
            "content": f"{username} left the chat",
            "timestamp": datetime.utcnow().isoformat(),
            "online_users": list(self.connections.keys())
        })
    
    async def handle_message(self, username: str, content: str):
        message = {
            "type": "chat_message",
            "sender": username,
            "content": content,
            "timestamp": datetime.utcnow().isoformat()
        }
        self.message_history.append(message)
        
        # Keep only last 100 messages
        if len(self.message_history) > 100:
            self.message_history = self.message_history[-100:]
        
        await self.broadcast(message)
    
    async def broadcast(self, message: dict):
        disconnected = []
        for username, ws in self.connections.items():
            try:
                await ws.send_json(message)
            except:
                disconnected.append(username)
        
        for username in disconnected:
            self.connections.pop(username, None)


chat = ChatServer()

@app.websocket("/ws/chat/{username}")
async def chat_endpoint(websocket: WebSocket, username: str):
    if username in chat.connections:
        await websocket.close(code=4000, reason="Username already taken")
        return
    
    await chat.connect(username, websocket)
    
    try:
        while True:
            msg = await websocket.receive_json()
            
            if msg["type"] == "chat_message":
                await chat.handle_message(username, msg["content"])
            
            elif msg["type"] == "typing":
                await chat.broadcast({
                    "type": "typing",
                    "username": username
                })
                
    except WebSocketDisconnect:
        await chat.disconnect(username)
```

---

## 19. Real-World Example: AI Interview (Audio Streaming)

```python
import base64
import asyncio
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws/interview/{token}")
async def interview_websocket(websocket: WebSocket, token: str):
    """
    Full AI interview flow over WebSocket.
    
    Flow:
    1. Validate token → accept connection
    2. Send greeting (text + audio)
    3. Loop: send question → receive audio → transcribe → evaluate → next
    4. Send closing message
    """
    
    # ─── Validate & Setup ───
    session = await validate_token(token)
    if not session:
        await websocket.close(code=4001, reason="Invalid token")
        return
    
    await websocket.accept()
    
    try:
        # ─── Phase 1: Greeting ───
        greeting_text = await generate_greeting(session.candidate_name)
        greeting_audio = await text_to_speech(greeting_text)
        
        await websocket.send_json({
            "type": "greeting",
            "text": greeting_text,
            "audio": base64.b64encode(greeting_audio).decode(),
        })
        
        # ─── Phase 2: Questions ───
        questions = await get_questions(session.interview_id)
        
        for i, question in enumerate(questions):
            # Send question
            question_audio = await text_to_speech(question.text)
            await websocket.send_json({
                "type": "question",
                "text": question.text,
                "audio": base64.b64encode(question_audio).decode(),
                "question_number": i + 1,
                "total_questions": len(questions),
            })
            
            # Wait for candidate's audio response
            response = await websocket.receive_json()
            
            if response["type"] == "audio_response":
                # Transcribe audio
                audio_bytes = base64.b64decode(response["audio"])
                transcript = await speech_to_text(audio_bytes)
                
                # Save response to DB
                await save_response(session.interview_id, question.id, transcript)
                
                # Score in background (don't block the flow)
                asyncio.create_task(
                    score_response(question, transcript)
                )
                
                # Evaluate: does the AI want to ask a follow-up?
                evaluation = await evaluate_answer(question, transcript)
                
                if evaluation.needs_follow_up and question.followups_asked < 2:
                    # Send follow-up question
                    follow_up_audio = await text_to_speech(evaluation.follow_up_question)
                    await websocket.send_json({
                        "type": "follow_up",
                        "text": evaluation.follow_up_question,
                        "audio": base64.b64encode(follow_up_audio).decode(),
                    })
                    
                    # Wait for follow-up response
                    fu_response = await websocket.receive_json()
                    if fu_response["type"] == "audio_response":
                        fu_audio = base64.b64decode(fu_response["audio"])
                        fu_transcript = await speech_to_text(fu_audio)
                        await save_response(session.interview_id, question.id, fu_transcript)
            
            elif response["type"] == "pause":
                await websocket.send_json({"type": "paused", "remaining_seconds": 300})
                # Wait for resume
                resume_msg = await asyncio.wait_for(
                    websocket.receive_json(), timeout=300
                )
        
        # ─── Phase 3: Closing ───
        closing_text = await generate_closing(session.candidate_name)
        closing_audio = await text_to_speech(closing_text)
        
        await websocket.send_json({
            "type": "interview_complete",
            "text": closing_text,
            "audio": base64.b64encode(closing_audio).decode(),
        })
        
        # Update session status
        await update_session_status(session.interview_id, "COMPLETED")
        
        # Trigger full scoring pipeline
        asyncio.create_task(generate_session_summary(session.interview_id))
        
    except WebSocketDisconnect:
        await update_session_status(session.interview_id, "INTERRUPTED")
        print(f"Interview {session.interview_id} interrupted — client disconnected")
    
    except asyncio.TimeoutError:
        await update_session_status(session.interview_id, "INTERRUPTED")
        await websocket.send_json({"type": "session_expired"})
        await websocket.close(code=4008, reason="Session timeout")
    
    except Exception as e:
        print(f"Interview error: {e}")
        try:
            await websocket.send_json({"type": "error", "text": "An error occurred"})
        except:
            pass
```

---

## 20. Testing WebSockets

### Using FastAPI TestClient

```python
from fastapi.testclient import TestClient
from main import app

def test_websocket_basic():
    client = TestClient(app)
    
    with client.websocket_connect("/ws") as ws:
        ws.send_text("Hello")
        data = ws.receive_text()
        assert data == "Echo: Hello"

def test_websocket_json():
    client = TestClient(app)
    
    with client.websocket_connect("/ws") as ws:
        ws.send_json({"type": "chat_message", "content": "Hello!"})
        response = ws.receive_json()
        assert response["type"] == "chat_message"
        assert "content" in response

def test_websocket_auth_failure():
    client = TestClient(app)
    
    with pytest.raises(Exception):
        with client.websocket_connect("/ws/interview/invalid-token") as ws:
            pass  # Should be rejected
```

### Using pytest-asyncio with httpx

```python
import pytest
import httpx
from httpx_ws import aconnect_ws

@pytest.mark.asyncio
async def test_websocket():
    async with httpx.AsyncClient(app=app, base_url="http://test") as client:
        async with aconnect_ws("/ws", client) as ws:
            await ws.send_json({"type": "ping"})
            response = await ws.receive_json()
            assert response["type"] == "pong"
```

### Using websockets Library

```python
import asyncio
import websockets
import json

async def test_connection():
    uri = "ws://localhost:8000/ws/chat/testuser"
    
    async with websockets.connect(uri) as ws:
        # Send a message
        await ws.send(json.dumps({
            "type": "chat_message",
            "content": "Hello from test!"
        }))
        
        # Receive response
        response = json.loads(await ws.recv())
        print(f"Received: {response}")
        
        assert response["type"] == "chat_message"

asyncio.run(test_connection())
```

### Quick Manual Testing with websocat (CLI)

```bash
# Install websocat (Rust-based CLI WebSocket client)
# Then:
websocat ws://localhost:8000/ws

# Type messages and see responses in real-time
{"type": "ping"}
# → {"type": "pong"}
```

---

## 21. Scaling WebSockets (Multiple Workers)

### The Problem

When you run multiple server processes (workers), each process has its own memory. A WebSocket connected to Worker 1 can't be reached from Worker 2.

```
Client A ──► Worker 1 (has Client A's connection)
Client B ──► Worker 2 (has Client B's connection)

Worker 1 wants to send to Client B... but Client B is on Worker 2!
```

### Solution: Pub/Sub Backend (Redis)

```python
import aioredis

redis = aioredis.from_url("redis://localhost")

# When a message needs to be broadcast:
await redis.publish("chat:general", json.dumps(message))

# Each worker subscribes:
async def redis_listener():
    pubsub = redis.pubsub()
    await pubsub.subscribe("chat:general")
    
    async for msg in pubsub.listen():
        if msg["type"] == "message":
            data = json.loads(msg["data"])
            # Broadcast to local connections
            await manager.broadcast(data)
```

### For This Project (Single Worker)

If you're running a single Uvicorn worker (no `--workers` flag), you don't need any of this. The in-memory `ConnectionManager` works fine.

```bash
# Single worker — ConnectionManager works perfectly
uvicorn main:app --host 0.0.0.0 --port 8000

# Multiple workers — need Redis pub/sub
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

---

## 22. Common Pitfalls & Solutions

### Pitfall 1: Forgetting `await`

```python
# ❌ This does NOTHING — just creates a coroutine object
websocket.send_json({"type": "hello"})

# ✅ Must await
await websocket.send_json({"type": "hello"})
```

### Pitfall 2: Accepting After Sending

```python
# ❌ Can't send before accepting!
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.send_text("Hello!")   # Error!
    await websocket.accept()

# ✅ Accept first, then send
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    await websocket.send_text("Hello!")
```

### Pitfall 3: Not Handling WebSocketDisconnect

```python
# ❌ Crashes when client disconnects
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()   # Raises WebSocketDisconnect!

# ✅ Handle gracefully
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
    except WebSocketDisconnect:
        print("Client left")
```

### Pitfall 4: Holding Database Sessions Open

```python
# ❌ DB session open for the entire WebSocket lifetime (could be hours!)
@app.websocket("/ws")
async def handler(websocket: WebSocket, db: Session = Depends(get_db)):
    await websocket.accept()
    while True:
        # db session is open the whole time — connection pool exhaustion!
        data = await websocket.receive_text()

# ✅ Create sessions per operation
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        db = SessionLocal()
        try:
            # Use db briefly
            db.commit()
        finally:
            db.close()
```

### Pitfall 5: Blocking the Event Loop

```python
# ❌ Blocking call freezes ALL WebSocket connections!
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        result = heavy_computation(data)   # Blocks for 5 seconds!
        await websocket.send_text(result)

# ✅ Run CPU-intensive work in a thread pool
import asyncio

@app.websocket("/ws")
async def handler(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        result = await asyncio.get_event_loop().run_in_executor(
            None,                         # Default thread pool
            heavy_computation, data       # Function and args
        )
        await websocket.send_text(result)
```

### Pitfall 6: Not Closing WebSocket in Error Cases

```python
# ❌ Returns without closing — connection hangs!
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    token = websocket.query_params.get("token")
    if not token:
        return   # Connection hangs in CONNECTING state!

# ✅ Close with appropriate code
@app.websocket("/ws")
async def handler(websocket: WebSocket):
    token = websocket.query_params.get("token")
    if not token:
        await websocket.close(code=4001, reason="Token required")
        return
```

---

## 23. Starlette WebSocket (Under the Hood)

FastAPI's WebSocket support comes from **Starlette**. Understanding this helps when you read source code or need advanced features.

### The ASGI Protocol

WebSocket in Python works through **ASGI** (Asynchronous Server Gateway Interface):

```python
# Under the hood, a WebSocket endpoint is an ASGI app:
async def websocket_app(scope, receive, send):
    # scope = {"type": "websocket", "path": "/ws", "headers": [...]}
    
    # Wait for connection
    event = await receive()
    # event = {"type": "websocket.connect"}
    
    # Accept
    await send({"type": "websocket.accept"})
    
    # Receive message
    event = await receive()
    # event = {"type": "websocket.receive", "text": "Hello"}
    
    # Send message
    await send({"type": "websocket.send", "text": "Hi!"})
    
    # Close
    await send({"type": "websocket.close", "code": 1000})
```

**FastAPI/Starlette wraps this into the clean `WebSocket` object you use.** The `WebSocket` class is just a convenience wrapper around these raw ASGI events.

### Starlette WebSocket Class (Simplified)

```python
# What Starlette's WebSocket class roughly does:
class WebSocket:
    def __init__(self, scope, receive, send):
        self.scope = scope
        self._receive = receive
        self._send = send
    
    async def accept(self):
        await self._send({"type": "websocket.accept"})
    
    async def receive_text(self) -> str:
        message = await self._receive()
        return message["text"]
    
    async def send_text(self, data: str):
        await self._send({"type": "websocket.send", "text": data})
    
    async def close(self, code: int = 1000):
        await self._send({"type": "websocket.close", "code": code})
    
    @property
    def url(self):
        return URL(scope=self.scope)
    
    @property
    def query_params(self):
        return QueryParams(self.scope.get("query_string", b""))
```

---

## 24. Quick Reference & Mental Model

### Mental Model

```
FastAPI Server
┌──────────────────────────────────────────────────┐
│                                                  │
│  @app.websocket("/ws/{token}")                   │
│  async def handler(websocket, token):            │
│      │                                           │
│      ├── await websocket.accept()                │
│      │        ↕ connection established           │
│      │                                           │
│      ├── while True:                             │
│      │   │                                       │
│      │   ├── msg = await websocket.receive_json()│  ← Waits here
│      │   │       (blocks until client sends)     │
│      │   │                                       │
│      │   ├── result = await process(msg)         │  ← Your logic
│      │   │                                       │
│      │   └── await websocket.send_json(result)   │  ← Send back
│      │                                           │
│      └── except WebSocketDisconnect:             │
│              cleanup()                           │
│                                                  │
└──────────────────────────────────────────────────┘
```

### Quick Reference

| Action | Code |
|--------|------|
| **Define endpoint** | `@app.websocket("/ws/{param}")` |
| **Accept connection** | `await websocket.accept()` |
| **Receive text** | `text = await websocket.receive_text()` |
| **Receive JSON** | `data = await websocket.receive_json()` |
| **Receive binary** | `bytes = await websocket.receive_bytes()` |
| **Send text** | `await websocket.send_text("hello")` |
| **Send JSON** | `await websocket.send_json({"key": "val"})` |
| **Send binary** | `await websocket.send_bytes(b"\x00")` |
| **Close** | `await websocket.close(code=1000)` |
| **Get query param** | `websocket.query_params.get("token")` |
| **Get path param** | Defined in function signature |
| **Get cookie** | `websocket.cookies.get("session")` |
| **Get header** | `websocket.headers.get("user-agent")` |
| **Get client IP** | `websocket.client.host` |
| **Handle disconnect** | `except WebSocketDisconnect:` |

### When to Use WebSocket on Server

| Scenario | HTTP or WebSocket? |
|----------|--------------------|
| Real-time chat | **WebSocket** |
| Live audio interview | **WebSocket** |
| Push notifications | **WebSocket** |
| Live dashboard updates | **WebSocket** |
| REST API (CRUD) | HTTP |
| File upload | HTTP |
| Authentication (login) | HTTP |
| One-time data fetch | HTTP |
| Form submission | HTTP |

### Architecture Decision Guide

```
"Does the server need to push data to the client unprompted?"
    │
    ├── YES → "Is it real-time (< 1 second latency)?"
    │           │
    │           ├── YES → WebSocket
    │           └── NO  → Server-Sent Events (SSE) or polling
    │
    └── NO  → HTTP (REST API)
```

---

*FastAPI makes WebSockets feel natural — it's just an async function with `await websocket.accept()`, a loop, and `await websocket.receive/send()`. The key patterns to master: the connection loop, JSON message routing, graceful disconnect handling, and the ConnectionManager for multi-client scenarios.*
