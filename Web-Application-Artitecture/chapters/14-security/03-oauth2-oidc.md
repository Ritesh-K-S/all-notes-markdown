# OAuth 2.0 & OpenID Connect — Login with Google/GitHub

> **What you'll learn**: How OAuth 2.0 allows third-party apps to access your data without your password, how OpenID Connect adds identity verification on top, and why "Login with Google" works the way it does.

---

## Real-Life Analogy

Imagine you're staying at a **hotel** and you want to let a **cleaning service** into your room.

**Without OAuth (the bad way):**
You give the cleaning service **your room key** — now they have full access to everything, forever, and you can't control what they do.

**With OAuth (the smart way):**
1. You go to the **front desk** (Authorization Server).
2. You say: "I want to give CleanCo access to clean my room, but ONLY between 9am-11am, and they CANNOT open the safe."
3. The front desk gives CleanCo a **limited-access keycard** that works only during those hours and only for the door (not the safe).
4. You can **revoke** this keycard anytime by calling the front desk.

```
WITHOUT OAuth:                          WITH OAuth:
┌─────────┐                            ┌─────────┐
│  User   │ gives password to          │  User   │ approves limited access
│ (Alice) │────────────────▶ App       │ (Alice) │──▶ Google ──▶ App gets
└─────────┘                            └─────────┘    (AuthZ)    limited token
                                        
"Here's my Google password,            "Google, let this app see my
 do whatever you want"  ❌              email and profile only" ✓
```

---

## Core Concept Explained Step-by-Step

### Step 1: The Problem OAuth Solves

Before OAuth, if an app wanted to access your Google contacts, you had to **give it your Google password**. This was terrible because:
- The app had **full access** to your entire Google account
- You couldn't **limit** what it could do
- You couldn't **revoke** access without changing your password
- If the app was hacked, your **Google password leaked**

OAuth 2.0 solves this: **"Let apps access specific data without your password."**

### Step 2: The Key Players (Roles)

```
┌─────────────────────────────────────────────────────────────────┐
│                    OAUTH 2.0 ROLES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐     The user who owns the data            │
│  │ Resource Owner   │     Example: You (Alice)                   │
│  └──────────────────┘                                            │
│                                                                  │
│  ┌──────────────────┐     The app that wants access              │
│  │ Client (App)     │     Example: A TODO app that wants         │
│  └──────────────────┘     your Google Calendar                   │
│                                                                  │
│  ┌──────────────────┐     Authenticates the user and             │
│  │ Authorization    │     issues tokens                          │
│  │ Server           │     Example: Google's auth server          │
│  └──────────────────┘     (accounts.google.com)                  │
│                                                                  │
│  ┌──────────────────┐     Hosts the protected data/API           │
│  │ Resource Server  │     Example: Google Calendar API           │
│  └──────────────────┘     (calendar.googleapis.com)              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: OAuth 2.0 — Authorization Code Flow (Most Common)

This is the flow used by "Login with Google" buttons:

```
┌──────┐          ┌───────┐          ┌─────────────┐          ┌──────────┐
│ User │          │  App  │          │ Google Auth │          │ Google   │
│(Alice)│          │(TODO) │          │   Server    │          │ API      │
└──┬───┘          └───┬───┘          └──────┬──────┘          └────┬─────┘
   │                  │                     │                      │
   │ 1. Click         │                     │                      │
   │ "Login with      │                     │                      │
   │  Google"         │                     │                      │
   │─────────────────▶│                     │                      │
   │                  │                     │                      │
   │                  │ 2. Redirect to      │                      │
   │                  │    Google with:      │                      │
   │                  │    - client_id       │                      │
   │                  │    - redirect_uri    │                      │
   │                  │    - scope           │                      │
   │                  │    - state           │                      │
   │◀─────────────────│                     │                      │
   │                  │                     │                      │
   │ 3. Google shows consent page:          │                      │
   │    "TODO App wants to access           │                      │
   │     your email and calendar"           │                      │
   │────────────────────────────────────────▶                      │
   │                  │                     │                      │
   │ 4. User clicks "Allow"                 │                      │
   │────────────────────────────────────────▶                      │
   │                  │                     │                      │
   │                  │ 5. Redirect back    │                      │
   │                  │    with AUTH CODE    │                      │
   │                  │◀────────────────────│                      │
   │                  │                     │                      │
   │                  │ 6. Exchange code    │                      │
   │                  │    for tokens       │                      │
   │                  │    (server-to-      │                      │
   │                  │     server)         │                      │
   │                  │────────────────────▶│                      │
   │                  │                     │                      │
   │                  │ 7. Access token +   │                      │
   │                  │    Refresh token    │                      │
   │                  │◀────────────────────│                      │
   │                  │                     │                      │
   │                  │ 8. Call Google API   │                      │
   │                  │    with access token │                      │
   │                  │─────────────────────────────────────────────▶
   │                  │                     │                      │
   │                  │ 9. Return user data  │                      │
   │                  │◀─────────────────────────────────────────────
   │                  │                     │                      │
   │ 10. Logged in!   │                     │                      │
   │◀─────────────────│                     │                      │
```

### Step 4: Scopes — Limiting What an App Can Do

**Scopes** define the specific permissions the app is requesting:

```
┌───────────────────────────────────────────────────────┐
│           GOOGLE CONSENT SCREEN                        │
├───────────────────────────────────────────────────────┤
│                                                        │
│  "TODO App" wants to access your Google account        │
│                                                        │
│  This will allow TODO App to:                          │
│                                                        │
│  ✓ See your email address (scope: email)               │
│  ✓ See your basic profile info (scope: profile)        │
│  ✓ View your calendar events (scope: calendar.read)    │
│                                                        │
│  ✗ It will NOT be able to:                             │
│    - Delete your emails                                │
│    - Change your password                              │
│    - Access your Drive files                           │
│                                                        │
│         [Allow]          [Deny]                        │
│                                                        │
└───────────────────────────────────────────────────────┘
```

Common Google scopes:
- `openid` — Basic identity (required for OIDC)
- `email` — User's email address
- `profile` — Name, picture, etc.
- `https://www.googleapis.com/auth/calendar.readonly` — Read calendar

### Step 5: OpenID Connect (OIDC) — Adding Identity

**OAuth 2.0** only handles **authorization** (access to resources).
**OpenID Connect** adds **authentication** (proving WHO the user is).

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   OAuth 2.0 alone:                                           │
│   "Here's a token to access Calendar API"                    │
│   (But WHO is the user? OAuth doesn't tell you!)             │
│                                                              │
│   OAuth 2.0 + OpenID Connect:                                │
│   "Here's a token to access Calendar API"                    │
│   AND "The user is alice@gmail.com, her name is Alice"       │
│   (ID Token tells you WHO they are)                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘

OAuth 2.0:  Access Token  → "What can you access?"
OIDC:       ID Token (JWT) → "Who are you?"
```

**OIDC adds the `id_token`** — a JWT containing user identity claims:

```json
{
  "iss": "https://accounts.google.com",
  "sub": "1234567890",
  "email": "alice@gmail.com",
  "name": "Alice Smith",
  "picture": "https://...",
  "aud": "your-app-client-id",
  "exp": 1700000000,
  "iat": 1699999000
}
```

---

## How It Works Internally

### The Authorization Code Exchange (Why Two Steps?)

Why does OAuth use a **code** that you then **exchange** for a token? Why not give the token directly?

```
INSECURE (Implicit Flow - Deprecated):
  Google ──redirect──▶ https://your-app.com#access_token=SECRET
  
  ⚠️ Problem: Token visible in URL, browser history, referer headers!

SECURE (Authorization Code Flow):
  Step 1: Google ──redirect──▶ https://your-app.com?code=TEMPORARY_CODE
           (Code is short-lived, one-time use, useless alone)
           
  Step 2: Your Server ──POST──▶ Google Token Endpoint
           {code: "...", client_secret: "...", redirect_uri: "..."}
           
           Google ──response──▶ {access_token, refresh_token, id_token}
           (Tokens NEVER appear in the browser!)
```

### PKCE — Protection for Public Clients (SPAs, Mobile Apps)

For apps that **can't keep secrets** (JavaScript in browser, mobile apps), PKCE adds protection:

```
┌──────────────────────────────────────────────────────────────┐
│                     PKCE FLOW                                  │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Client generates random "code_verifier" (secret)          │
│     code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p..."        │
│                                                               │
│  2. Client creates "code_challenge" (hash of verifier)        │
│     code_challenge = SHA256(code_verifier) = "E9Melhoa..."    │
│                                                               │
│  3. Client sends code_challenge in auth request               │
│     GET /authorize?code_challenge=E9Melhoa...                 │
│                                                               │
│  4. Server stores the challenge                               │
│                                                               │
│  5. After user approves, client exchanges code                │
│     POST /token {code: "...", code_verifier: "dBjft..."}      │
│                                                               │
│  6. Server verifies: SHA256(code_verifier) == stored_challenge│
│     If match → issue tokens                                   │
│                                                               │
│  Why it works: Even if attacker intercepts the code,          │
│  they don't have the code_verifier, so they can't             │
│  exchange it for tokens!                                      │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### OAuth 2.0 Grant Types Summary

```
┌───────────────────────────────────────────────────────────────┐
│                    GRANT TYPES                                  │
├───────────────────┬───────────────────────────────────────────┤
│ Grant Type        │ When to Use                                │
├───────────────────┼───────────────────────────────────────────┤
│ Authorization     │ Web apps with a backend server             │
│ Code (+PKCE)      │ (most common, most secure)                 │
├───────────────────┼───────────────────────────────────────────┤
│ Client            │ Server-to-server (no user involved)        │
│ Credentials       │ Machine-to-machine communication           │
├───────────────────┼───────────────────────────────────────────┤
│ Device Code       │ Smart TVs, CLI tools, IoT                  │
│                   │ (devices with limited input)               │
├───────────────────┼───────────────────────────────────────────┤
│ Refresh Token     │ Getting new access tokens without          │
│                   │ re-authentication                          │
├───────────────────┼───────────────────────────────────────────┤
│ Implicit          │ ❌ DEPRECATED — use Auth Code + PKCE       │
│ (don't use)       │                                            │
├───────────────────┼───────────────────────────────────────────┤
│ Password          │ ❌ DEPRECATED — only for legacy migration  │
│ (don't use)       │                                            │
└───────────────────┴───────────────────────────────────────────┘
```

---

## Code Examples

### Python — OAuth 2.0 Authorization Code Flow

```python
# oauth_example.py — Flask app with Google OAuth
from flask import Flask, redirect, request, session, jsonify
import requests
import secrets

app = Flask(__name__)
app.secret_key = secrets.token_hex(32)

# Register your app at https://console.cloud.google.com
CLIENT_ID = "your-client-id.apps.googleusercontent.com"
CLIENT_SECRET = "your-client-secret"
REDIRECT_URI = "http://localhost:5000/callback"
AUTH_URL = "https://accounts.google.com/o/oauth2/v2/auth"
TOKEN_URL = "https://oauth2.googleapis.com/token"
USERINFO_URL = "https://www.googleapis.com/oauth2/v3/userinfo"

@app.route("/login")
def login():
    """Step 1: Redirect user to Google's consent page"""
    state = secrets.token_urlsafe(16)  # CSRF protection
    session["oauth_state"] = state
    
    params = {
        "client_id": CLIENT_ID,
        "redirect_uri": REDIRECT_URI,
        "response_type": "code",          # We want an authorization code
        "scope": "openid email profile",   # What we want to access
        "state": state,                    # CSRF protection
        "access_type": "offline",          # Get refresh token too
    }
    auth_url = f"{AUTH_URL}?{'&'.join(f'{k}={v}' for k, v in params.items())}"
    return redirect(auth_url)

@app.route("/callback")
def callback():
    """Step 2: Google redirects back with a code"""
    # Verify state to prevent CSRF
    if request.args.get("state") != session.get("oauth_state"):
        return "CSRF detected!", 403
    
    code = request.args.get("code")
    if not code:
        return "Authorization denied", 400
    
    # Step 3: Exchange code for tokens (server-to-server, secure!)
    token_response = requests.post(TOKEN_URL, data={
        "code": code,
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "redirect_uri": REDIRECT_URI,
        "grant_type": "authorization_code",
    })
    tokens = token_response.json()
    access_token = tokens["access_token"]
    
    # Step 4: Use access token to get user info
    user_info = requests.get(USERINFO_URL, headers={
        "Authorization": f"Bearer {access_token}"
    }).json()
    
    # User is authenticated! Store in session
    session["user"] = {
        "email": user_info["email"],
        "name": user_info["name"],
        "picture": user_info["picture"],
    }
    return redirect("/profile")

@app.route("/profile")
def profile():
    """Protected route — only for authenticated users"""
    if "user" not in session:
        return redirect("/login")
    return jsonify(session["user"])
```

### Java — OAuth 2.0 with Spring Security

```java
// SecurityConfig.java — Spring Boot OAuth2 Login
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/public/**").permitAll()  // Public pages
                .anyRequest().authenticated()                    // Everything else needs login
            )
            .oauth2Login(oauth -> oauth
                .loginPage("/login")                            // Custom login page
                .defaultSuccessUrl("/dashboard", true)          // After successful login
            );
        return http.build();
    }
}
```

```yaml
# application.yml — Google OAuth configuration
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: your-client-id.apps.googleusercontent.com
            client-secret: your-client-secret
            scope: openid, email, profile
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          google:
            authorization-uri: https://accounts.google.com/o/oauth2/v2/auth
            token-uri: https://oauth2.googleapis.com/token
            user-info-uri: https://www.googleapis.com/oauth2/v3/userinfo
```

### Python — Client Credentials Flow (Machine-to-Machine)

```python
# client_credentials.py — Service-to-service auth (no user involved)
import requests

# Your service's credentials (registered with auth provider)
CLIENT_ID = "my-backend-service"
CLIENT_SECRET = "service-secret-key"
TOKEN_URL = "https://auth.example.com/oauth2/token"

def get_service_token():
    """Get an access token for service-to-service calls"""
    response = requests.post(TOKEN_URL, data={
        "grant_type": "client_credentials",
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "scope": "orders:read payments:write",  # What this service needs
    })
    return response.json()["access_token"]

def call_payment_service(order_id):
    """Call another microservice using the service token"""
    token = get_service_token()
    response = requests.post(
        "https://payments.internal/charge",
        headers={"Authorization": f"Bearer {token}"},
        json={"order_id": order_id, "amount": 99.99}
    )
    return response.json()
```

---

## Infrastructure Examples

### OAuth Provider Configuration (Keycloak Docker)

```yaml
# docker-compose.yml — Self-hosted OAuth/OIDC with Keycloak
version: '3.8'
services:
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloak
    ports:
      - "8080:8080"
    command: start-dev
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloak
    volumes:
      - keycloak_data:/var/lib/postgresql/data

volumes:
  keycloak_data:
```

### Nginx — OAuth2 Proxy Pattern

```nginx
# Protect any app behind OAuth without modifying the app
server {
    listen 443 ssl;
    server_name app.example.com;

    location /oauth2/ {
        proxy_pass http://oauth2-proxy:4180;
    }

    location / {
        # Every request goes through OAuth2 Proxy first
        auth_request /oauth2/auth;
        
        # If authenticated, forward to the actual app
        error_page 401 = /oauth2/sign_in;
        proxy_pass http://backend-app:8080;
        
        # Pass user info to backend
        auth_request_set $user $upstream_http_x_auth_request_user;
        proxy_set_header X-User $user;
    }
}
```

---

## Real-World Example

### How "Login with Google" Works (Step by Step)

When you click "Sign in with Google" on Spotify:

1. **Spotify** redirects you to `accounts.google.com` with its client_id
2. **Google** shows: "Spotify wants access to your email and profile"
3. **You** click "Allow"
4. **Google** redirects back to Spotify with a one-time code
5. **Spotify's server** exchanges the code for tokens (access + id token)
6. **Spotify** reads the id_token to get your email/name
7. **Spotify** creates an account (or logs you into existing one)
8. You're in! No password needed.

### How GitHub Uses OAuth for Third-Party Apps

Every GitHub App that accesses your repos uses OAuth:
- Scope `repo` = full access to your repositories
- Scope `read:user` = read your profile info
- Scope `gist` = create/read gists

You can see all authorized apps at: github.com/settings/applications

---

## Common Mistakes / Pitfalls

| Mistake | Risk | Fix |
|---------|------|-----|
| Not validating `state` parameter | CSRF attacks | Always generate and verify state |
| Using Implicit flow for SPAs | Token exposure in URL | Use Authorization Code + PKCE |
| Not validating redirect_uri | Open redirect attacks | Whitelist exact redirect URIs |
| Requesting too many scopes | Users won't trust your app | Request minimum necessary scopes |
| Not implementing token refresh | Users re-login frequently | Use refresh tokens |
| Storing client_secret in frontend JS | Secret is exposed | Keep secrets server-side only |
| Not validating id_token signature | Token forgery | Always verify JWT signature and claims |

---

## When to Use / When NOT to Use

### Use OAuth 2.0 When:
- You want "Login with Google/GitHub/Facebook"
- Your app needs to access user data on another platform
- You're building a platform that third-party apps integrate with
- Machine-to-machine (service-to-service) authentication

### Use OpenID Connect When:
- You need to know WHO the user is (identity), not just access their data
- You're building a single sign-on (SSO) system
- You want standardized user profile information

### Don't Use OAuth When:
- Simple username/password login is sufficient (no third-party involvement)
- Internal-only applications where you control all services (use mTLS instead)
- You just need API key authentication for a simple service

---

## Key Takeaways

- **OAuth 2.0** is for **authorization** — granting limited access without sharing passwords
- **OpenID Connect** adds **authentication** on top of OAuth — proving user identity
- The **Authorization Code flow** (+PKCE) is the recommended approach for all clients
- **Never use Implicit flow** — it's deprecated and insecure
- **Scopes** limit what an app can do with a token
- The **state parameter** prevents CSRF attacks — always use it
- **Access tokens** are short-lived (minutes); **refresh tokens** are long-lived (days/weeks)
- For machine-to-machine, use **Client Credentials** grant

---

## What's Next?

Next, we'll explore **SSL/TLS & Certificate Management** — how data is encrypted in transit so that nobody can eavesdrop on tokens, passwords, or personal data flowing between client and server. That's Chapter 14.4.
