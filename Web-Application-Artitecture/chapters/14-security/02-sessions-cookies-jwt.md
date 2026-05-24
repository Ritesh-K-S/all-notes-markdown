# Sessions, Cookies & Tokens (JWT)

> **What you'll learn**: How web applications "remember" that you're logged in across multiple requests — using sessions, cookies, and tokens — and when to use each approach.

---

## Real-Life Analogy

Imagine you're at an **amusement park**.

**Scenario 1: Wristband (Session + Cookie)**
- At the entrance, you pay and get a **wristband** with a number on it (e.g., #4821).
- The park keeps a **ledger** at the office: "#4821 → Alice, VIP pass, entered at 2pm."
- Every time you go on a ride, the staff **scans your wristband** and checks the ledger.
- The wristband itself doesn't say WHO you are — it's just a reference number. The park's ledger has all the details.

**Scenario 2: Passport Stamp (Token/JWT)**
- Instead of a wristband, you get a **special card** that says: "Alice, VIP, entered at 2pm" with a **holographic seal** that can't be forged.
- Every ride attendant **reads the card directly** — no need to call the main office.
- The card is self-contained: all the information is right there.
- But if you lose it, anyone who finds it can pretend to be you (until it expires).

```
SESSION-BASED (Wristband)              TOKEN-BASED (Passport Card)
┌──────────┐                           ┌──────────┐
│  Client  │ has: session_id=4821      │  Client  │ has: full token with data
└─────┬────┘                           └─────┬────┘
      │                                      │
      │ "I'm #4821"                          │ "Here's my full card"
      ▼                                      ▼
┌──────────┐                           ┌──────────┐
│  Server  │ looks up #4821 in         │  Server  │ verifies the seal (signature)
│          │ session store             │          │ reads data directly from card
└──────────┘                           └──────────┘
      │                                
      ▼                                No lookup needed!
┌──────────┐                           
│ Session  │ Redis/Memory/DB           
│ Store    │ {4821: {user: alice,...}}  
└──────────┘                           
```

---

## Core Concept Explained Step-by-Step

### Step 1: The Problem — HTTP is Stateless

HTTP doesn't remember you between requests. Every request is independent:

```
Request 1: GET /profile (Who are you? I don't know you!)
Request 2: GET /orders  (Who are you? Still don't know you!)
```

We need a way to say: "Hey, I already logged in! Remember me!"

### Step 2: Cookies — The Browser's Memory

A **cookie** is a small piece of data the server sends to the browser. The browser stores it and sends it back with EVERY subsequent request to that server.

```
1. Login Request:
   Client ──▶ POST /login {username, password} ──▶ Server

2. Server Response (sets a cookie):
   Server ──▶ Set-Cookie: session_id=abc123; HttpOnly; Secure ──▶ Client

3. Every subsequent request (browser sends cookie automatically):
   Client ──▶ GET /profile + Cookie: session_id=abc123 ──▶ Server
```

**Cookie attributes that matter:**

| Attribute | Purpose | Example |
|-----------|---------|---------|
| `HttpOnly` | JavaScript can't access it (prevents XSS theft) | `HttpOnly` |
| `Secure` | Only sent over HTTPS | `Secure` |
| `SameSite` | Prevents CSRF attacks | `SameSite=Strict` |
| `Domain` | Which domains receive it | `Domain=.example.com` |
| `Path` | Which paths receive it | `Path=/api` |
| `Expires/Max-Age` | When it dies | `Max-Age=3600` |

### Step 3: Sessions — Server-Side State

A **session** is server-side storage tied to a session ID. The session ID is stored in a cookie.

```
┌─────────────────────────────────────────────────────────────┐
│                    SESSION FLOW                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. User logs in successfully                                │
│     └── Server creates session: {id: "xyz", user: "alice"}  │
│     └── Server stores session in Redis/memory/DB             │
│     └── Server sends cookie: session_id=xyz                  │
│                                                              │
│  2. User makes next request                                  │
│     └── Browser auto-sends cookie: session_id=xyz            │
│     └── Server looks up "xyz" in session store               │
│     └── Server finds: {user: "alice", role: "admin"}         │
│     └── Server knows who the user is!                        │
│                                                              │
│  3. User logs out                                            │
│     └── Server deletes session "xyz" from store              │
│     └── Cookie becomes meaningless                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Step 4: Tokens (JWT) — Client-Side State

A **JSON Web Token (JWT)** contains the user's information INSIDE the token itself. No server-side storage needed.

**JWT Structure:**

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiYWxpY2UiLCJyb2xlIjoiYWRtaW4ifQ.4xR2...

      HEADER              PAYLOAD                    SIGNATURE
   ┌──────────┐     ┌─────────────────┐        ┌──────────────┐
   │ {        │     │ {               │        │              │
   │  "alg":  │  .  │  "user":"alice",│   .    │  HMAC-SHA256(│
   │  "HS256" │     │  "role":"admin",│        │    header +  │
   │ }        │     │  "exp": 172800  │        │    payload,  │
   │          │     │ }               │        │    SECRET    │
   └──────────┘     └─────────────────┘        │  )           │
                                                └──────────────┘
   Base64url         Base64url                   Base64url
   encoded           encoded                     encoded
```

**How JWT works:**

```
┌────────┐                         ┌────────────┐
│ Client │                         │   Server   │
└───┬────┘                         └─────┬──────┘
    │                                    │
    │ 1. POST /login                     │
    │    {username, password}             │
    │───────────────────────────────────▶│
    │                                    │ Verify credentials
    │                                    │ Create JWT with user data
    │                                    │ Sign with SECRET key
    │ 2. Response: {token: "eyJ..."}     │
    │◀───────────────────────────────────│
    │                                    │
    │ (Client stores token in            │
    │  localStorage or cookie)           │
    │                                    │
    │ 3. GET /profile                    │
    │    Authorization: Bearer eyJ...    │
    │───────────────────────────────────▶│
    │                                    │ Verify signature
    │                                    │ Decode payload
    │                                    │ No DB lookup needed!
    │ 4. Response: {user data}           │
    │◀───────────────────────────────────│
```

### Step 5: Sessions vs Tokens — The Key Differences

```
┌─────────────────────────────────────────────────────────────────┐
│              SESSIONS vs TOKENS COMPARISON                        │
├──────────────────┬──────────────────────┬───────────────────────┤
│ Aspect           │ Sessions             │ Tokens (JWT)          │
├──────────────────┼──────────────────────┼───────────────────────┤
│ State stored     │ Server-side          │ Client-side           │
│ Scalability      │ Needs shared store   │ Stateless, scales easy│
│ Revocation       │ Easy (delete session)│ Hard (wait for expiry)│
│ Size             │ Small cookie (~32b)  │ Larger (~1-2 KB)      │
│ Database lookup  │ Every request        │ None (self-contained) │
│ Offline capable  │ No                   │ Yes                   │
│ Cross-domain     │ Difficult            │ Easy (in header)      │
│ Mobile friendly  │ Harder               │ Easier                │
└──────────────────┴──────────────────────┴───────────────────────┘
```

---

## How It Works Internally

### Session Storage Options

```
┌──────────────────────────────────────────────────────────────┐
│                   SESSION STORAGE OPTIONS                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. In-Memory (process memory)                                │
│     ├── Fastest                                               │
│     ├── Lost on restart                                       │
│     └── Can't share between servers ❌                        │
│                                                               │
│  2. Redis / Memcached                                         │
│     ├── Fast (in-memory but persistent)                       │
│     ├── Shared between all servers ✓                          │
│     ├── Auto-expiry with TTL                                  │
│     └── Most common in production ⭐                          │
│                                                               │
│  3. Database (PostgreSQL, MongoDB)                            │
│     ├── Persistent                                            │
│     ├── Slower than Redis                                     │
│     └── Good for audit trails                                 │
│                                                               │
│  4. Encrypted Cookie (cookie-based sessions)                  │
│     ├── No server storage needed                              │
│     ├── Size limited (~4KB)                                   │
│     └── Used by Rails, Django                                 │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### JWT Internals — How Signing Works

```
┌───────────────────────────────────────────────────────────────┐
│                    JWT SIGNING PROCESS                          │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  Creating (Server-side):                                       │
│  ┌──────────────────────────────────────────────────┐         │
│  │ header = base64url({"alg":"HS256","typ":"JWT"})  │         │
│  │ payload = base64url({"sub":"alice","exp":1234})  │         │
│  │ signature = HMAC-SHA256(header + "." + payload,  │         │
│  │                         SECRET_KEY)               │         │
│  │ token = header + "." + payload + "." + signature │         │
│  └──────────────────────────────────────────────────┘         │
│                                                                │
│  Verifying (Server-side):                                      │
│  ┌──────────────────────────────────────────────────┐         │
│  │ 1. Split token into header, payload, signature   │         │
│  │ 2. Recompute: expected = HMAC(header.payload,    │         │
│  │                                SECRET_KEY)        │         │
│  │ 3. Compare: expected == received_signature?       │         │
│  │ 4. Check: Is "exp" in the future?                │         │
│  │ 5. If all pass → VALID ✓                         │         │
│  └──────────────────────────────────────────────────┘         │
│                                                                │
│  ⚠️ Payload is NOT encrypted — only signed!                    │
│     Anyone can decode and READ it.                             │
│     But nobody can MODIFY it without the secret key.           │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

### Refresh Token Pattern

Access tokens expire quickly (15 min). Refresh tokens last longer (7 days) and are used to get new access tokens:

```
┌────────┐                              ┌────────────┐
│ Client │                              │   Server   │
└───┬────┘                              └─────┬──────┘
    │                                         │
    │ 1. Login → Get access_token (15min)     │
    │           + refresh_token (7 days)       │
    │◀────────────────────────────────────────│
    │                                         │
    │ 2. API calls with access_token          │
    │───────────────────────────────────────▶│ ✓ Valid
    │                                         │
    │ ... 15 minutes later ...                │
    │                                         │
    │ 3. API call with expired access_token   │
    │───────────────────────────────────────▶│ ✗ 401 Expired
    │                                         │
    │ 4. POST /refresh {refresh_token}        │
    │───────────────────────────────────────▶│
    │                                         │ Verify refresh token
    │ 5. New access_token (15min)             │ Issue new access token
    │◀────────────────────────────────────────│
    │                                         │
    │ 6. Retry API call with new token        │
    │───────────────────────────────────────▶│ ✓ Valid
```

---

## Code Examples

### Python — Session-Based Auth with Flask

```python
# session_auth.py — Using Flask sessions (stored in signed cookie)
from flask import Flask, session, request, jsonify
import secrets

app = Flask(__name__)
app.secret_key = secrets.token_hex(32)  # Secret for signing session cookie

# Simulated user database
users = {"alice": "hashed_password_here"}

@app.route("/login", methods=["POST"])
def login():
    data = request.json
    username = data.get("username")
    password = data.get("password")
    
    # Authentication: verify credentials
    if username in users and users[username] == password:
        # Create session — stored server-side (or in signed cookie)
        session["user"] = username
        session["role"] = "admin"
        return jsonify({"message": "Logged in!"}), 200
    
    return jsonify({"error": "Invalid credentials"}), 401

@app.route("/profile")
def profile():
    # Check session exists (user is authenticated)
    if "user" not in session:
        return jsonify({"error": "Not authenticated"}), 401
    
    return jsonify({"user": session["user"], "role": session["role"]})

@app.route("/logout", methods=["POST"])
def logout():
    session.clear()  # Destroy session — instant revocation!
    return jsonify({"message": "Logged out"})
```

### Python — JWT-Based Auth

```python
# jwt_auth.py — Stateless token-based authentication
import jwt
import datetime
from flask import Flask, request, jsonify

app = Flask(__name__)
SECRET_KEY = "your-256-bit-secret-keep-this-safe"

def create_token(user_id: str, role: str) -> str:
    """Create a JWT with user info and expiry"""
    payload = {
        "sub": user_id,               # Subject (who)
        "role": role,                   # Custom claim
        "iat": datetime.datetime.utcnow(),  # Issued at
        "exp": datetime.datetime.utcnow() + datetime.timedelta(minutes=15)  # Expires
    }
    return jwt.encode(payload, SECRET_KEY, algorithm="HS256")

def verify_token(token: str) -> dict | None:
    """Verify and decode a JWT. Returns payload or None."""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload
    except jwt.ExpiredSignatureError:
        return None  # Token expired
    except jwt.InvalidTokenError:
        return None  # Invalid token

@app.route("/login", methods=["POST"])
def login():
    data = request.json
    # ... verify credentials ...
    
    # Issue token (no server-side storage!)
    token = create_token(user_id="alice", role="admin")
    return jsonify({"access_token": token, "token_type": "Bearer"})

@app.route("/protected")
def protected():
    # Extract token from Authorization header
    auth_header = request.headers.get("Authorization", "")
    if not auth_header.startswith("Bearer "):
        return jsonify({"error": "Missing token"}), 401
    
    token = auth_header.split(" ")[1]
    payload = verify_token(token)
    
    if not payload:
        return jsonify({"error": "Invalid or expired token"}), 401
    
    return jsonify({"message": f"Hello {payload['sub']}!", "role": payload["role"]})
```

### Java — JWT with JJWT Library

```java
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import java.security.Key;
import java.util.Date;

public class JwtExample {
    
    // Secret key (in production, load from environment/vault)
    private static final Key SECRET = Keys.secretKeyFor(SignatureAlgorithm.HS256);
    private static final long EXPIRY_MS = 15 * 60 * 1000; // 15 minutes

    // CREATE a JWT
    public static String createToken(String userId, String role) {
        return Jwts.builder()
                .setSubject(userId)              // Who is this token for
                .claim("role", role)             // Custom claims
                .setIssuedAt(new Date())         // When was it created
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRY_MS))
                .signWith(SECRET)                // Sign with secret key
                .compact();                       // Build the string
    }

    // VERIFY and DECODE a JWT
    public static Claims verifyToken(String token) {
        try {
            return Jwts.parserBuilder()
                    .setSigningKey(SECRET)
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
        } catch (ExpiredJwtException e) {
            System.out.println("Token expired!");
            return null;
        } catch (JwtException e) {
            System.out.println("Invalid token!");
            return null;
        }
    }

    public static void main(String[] args) {
        // Login → create token
        String token = createToken("alice", "admin");
        System.out.println("Token: " + token);

        // Later → verify token
        Claims claims = verifyToken(token);
        if (claims != null) {
            System.out.println("User: " + claims.getSubject());   // alice
            System.out.println("Role: " + claims.get("role"));    // admin
        }
    }
}
```

---

## Infrastructure Examples

### Redis for Session Storage

```bash
# Store session in Redis with 30-minute TTL
SET session:abc123 '{"user":"alice","role":"admin","login_time":"2024-01-15T10:00:00"}' EX 1800

# Retrieve session
GET session:abc123

# Delete session (logout)
DEL session:abc123

# Check how long until expiry
TTL session:abc123
```

### Session Store Architecture (Multi-Server)

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│ Server 1 │────▶│              │◀────│ Server 2 │
└──────────┘     │    Redis     │     └──────────┘
                 │  (Sessions)  │
┌──────────┐     │              │     ┌──────────┐
│ Server 3 │────▶│ session:abc  │◀────│ Server 4 │
└──────────┘     │ session:def  │     └──────────┘
                 │ session:ghi  │
                 └──────────────┘
                 
All servers share the SAME session store.
Any server can validate any session.
```

---

## Real-World Example

### How GitHub Uses Tokens

GitHub uses a **layered approach**:
- **Session cookies** for the web UI (traditional session-based)
- **Personal Access Tokens (PAT)** for API access (like JWTs but opaque)
- **OAuth tokens** for third-party app integrations
- **Short-lived tokens** for GitHub Actions (expire after the workflow)

### How Amazon Handles Sessions at Scale

- **Billions** of active sessions at any time
- Sessions stored in **DynamoDB** (key-value, auto-scaling)
- Session data is **encrypted** at rest
- **Different TTLs** for different actions:
  - Browsing: session lasts 30 days
  - Purchasing: re-authentication required
  - Account settings: fresh login needed

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Dangerous | Fix |
|---------|-------------------|-----|
| Storing JWT in localStorage | Vulnerable to XSS attacks | Use HttpOnly cookies |
| Not setting token expiry | Stolen token works forever | Set short expiry (15 min) |
| Using JWT without HTTPS | Token can be intercepted | Always use HTTPS |
| Storing sensitive data in JWT payload | JWT payload is Base64 — NOT encrypted | Only store IDs and roles |
| Not implementing refresh tokens | Users must re-login every 15 min | Use refresh token rotation |
| Session fixation | Attacker sets the session ID | Regenerate session ID after login |
| Not invalidating sessions on password change | Old sessions still work | Clear all sessions on password reset |

---

## When to Use / When NOT to Use

### Use Sessions When:
- You need **instant revocation** (e.g., "log out all devices")
- Your app is a **traditional web app** with server-rendered pages
- You have a **small number of servers** or a shared session store
- You need to store **large amounts of session data**

### Use JWTs When:
- You have a **microservices architecture** (services need to verify independently)
- You need **cross-domain authentication** (multiple subdomains/services)
- You're building a **mobile app** or **SPA**
- You want to **avoid database lookups** on every request
- You need **stateless horizontal scaling**

### Use Both (Hybrid):
- JWT for **service-to-service** communication
- Sessions for **user-facing** web application
- Refresh tokens stored in **HttpOnly cookies**, access tokens in memory

---

## Key Takeaways

- **Cookies** are the browser's way of sending data with every request — they're the transport mechanism
- **Sessions** store user state on the server, referenced by a session ID in a cookie
- **JWTs** are self-contained tokens — all user data is inside the token itself
- JWT payloads are **signed but NOT encrypted** — don't put secrets in them
- Use **HttpOnly + Secure + SameSite cookies** for maximum security
- For scaling: sessions need **shared storage** (Redis); JWTs are **stateless**
- Always implement **token expiry** and **refresh token rotation**

---

## What's Next?

Next, we'll explore **OAuth 2.0 & OpenID Connect** — the protocol that powers "Login with Google" and "Login with GitHub." How does a third-party service safely give your app access to a user's data without sharing their password? That's Chapter 14.3.
