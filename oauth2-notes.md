# OAuth 2.0 & OpenID Connect — Complete Notes (Generic / API-Based)

> **Focus**: Language-agnostic. Everything here can be tested with Postman or any HTTP client. No framework-specific code — only raw HTTP requests, URLs, and JSON.

---

## Table of Contents

1. [What Problem Does OAuth2 Solve?](#1-what-problem-does-oauth2-solve)
2. [Key Terminology](#2-key-terminology)
3. [OAuth2 vs Authentication vs Authorization](#3-oauth2-vs-authentication-vs-authorization)
4. [OAuth2 Roles (The 4 Players)](#4-oauth2-roles-the-4-players)
5. [Tokens — Access Token, Refresh Token, ID Token](#5-tokens--access-token-refresh-token-id-token)
6. [OAuth2 Grant Types (Flows)](#6-oauth2-grant-types-flows)
7. [Authorization Code Flow (Most Common)](#7-authorization-code-flow-most-common)
8. [Authorization Code Flow with PKCE](#8-authorization-code-flow-with-pkce)
9. [Client Credentials Flow](#9-client-credentials-flow)
10. [Resource Owner Password Flow (Deprecated)](#10-resource-owner-password-flow-deprecated)
11. [Implicit Flow (Deprecated)](#11-implicit-flow-deprecated)
12. [Device Authorization Flow](#12-device-authorization-flow)
13. [Refresh Token Flow](#13-refresh-token-flow)
14. [OpenID Connect (OIDC) — Identity Layer on OAuth2](#14-openid-connect-oidc--identity-layer-on-oauth2)
15. [Scopes](#15-scopes)
16. [Token Introspection & Revocation](#16-token-introspection--revocation)
17. [JWKS & Token Validation](#17-jwks--token-validation)
18. [Provider: Google](#18-provider-google)
19. [Provider: Microsoft (Azure AD / Entra ID)](#19-provider-microsoft-azure-ad--entra-id)
20. [Provider: GitHub](#20-provider-github)
21. [Provider: Keycloak (Self-Hosted)](#21-provider-keycloak-self-hosted)
22. [Provider: Okta](#22-provider-okta)
23. [Well-Known Endpoints (Discovery)](#23-well-known-endpoints-discovery)
24. [Security Best Practices](#24-security-best-practices)
25. [Common Errors & Troubleshooting](#25-common-errors--troubleshooting)
26. [Quick Reference — All Endpoints Cheat Sheet](#26-quick-reference--all-endpoints-cheat-sheet)
27. [Why Authorization Code Instead of Direct Token?](#27-why-authorization-code-instead-of-direct-token)
28. [Real-World OAuth Login — How Apps Handle Users](#28-real-world-oauth-login--how-apps-handle-users)
29. [ID Token vs Access Token — Deep Dive](#29-id-token-vs-access-token--deep-dive)
30. [SPA + Backend API Architecture (Decoupled Apps)](#30-spa--backend-api-architecture-decoupled-apps)
31. [Microsoft-Specific — oid vs sub, Custom API Scopes, OBO Flow](#31-microsoft-specific--oid-vs-sub-custom-api-scopes-obo-flow)

---

## 1. WHAT PROBLEM DOES OAUTH2 SOLVE?

### The Problem (Before OAuth2)

Imagine you want a third-party app (e.g., a calendar app) to read your Google emails. Without OAuth2:

```
OLD WAY (Dangerous):
  You give your Google username + password to the calendar app
  → The app can read emails, delete emails, change password, do ANYTHING
  → If the app is hacked, your Google account is compromised
  → You can't revoke access without changing your password
```

### The Solution (OAuth2)

```
OAUTH2 WAY (Safe):
  Calendar app redirects you to Google's login page
  → YOU log in directly with Google (app never sees your password)
  → Google asks: "Calendar app wants to read your emails. Allow?"
  → You click "Allow"
  → Google gives the calendar app a LIMITED TOKEN
  → Token can ONLY read emails (not delete, not change password)
  → You can revoke the token anytime without changing your password
```

### One-Line Definition

**OAuth 2.0** is an authorization framework that lets a third-party application access a user's resources on another server **without the user sharing their password**.

---

## 2. KEY TERMINOLOGY

| Term | Meaning | Example |
|------|---------|---------|
| **Resource Owner** | The user who owns the data | You (your Google account) |
| **Client** | The app that wants to access the data | Calendar app |
| **Authorization Server** | The server that authenticates the user and issues tokens | Google's OAuth server |
| **Resource Server** | The server that holds the protected data | Gmail API |
| **Access Token** | A credential that allows access to resources | `eyJhbGciOi...` |
| **Refresh Token** | A credential used to get a new access token | `dGhpcyBpcy...` |
| **ID Token** | A JWT containing user identity info (OIDC only) | `eyJhbGciOi...` (contains name, email) |
| **Scope** | What permissions the token has | `read:email`, `profile` |
| **Grant Type** | The method/flow used to obtain a token | `authorization_code`, `client_credentials` |
| **Redirect URI** | Where the user is sent after authorization | `https://myapp.com/callback` |
| **Client ID** | Public identifier for the app | `abc123.apps.googleusercontent.com` |
| **Client Secret** | Private secret for the app (like a password) | `GOCSPX-xxxxx` |
| **Authorization Code** | A temporary code exchanged for tokens | `4/0AX4XfWi...` |
| **PKCE** | Proof Key for Code Exchange — extra security for public clients | Code verifier + challenge |
| **Consent Screen** | The page where user approves permissions | "App X wants to access your email" |

---

## 3. OAUTH2 VS AUTHENTICATION VS AUTHORIZATION

| Concept | Question It Answers | Example |
|---------|-------------------|---------|
| **Authentication** | WHO are you? | "I am John" (login) |
| **Authorization** | WHAT can you do? | "John can read emails but not delete them" |
| **OAuth2** | How to grant LIMITED ACCESS to someone else | "Calendar app can read John's emails" |

> **Critical**: OAuth2 is an **authorization** framework, NOT an authentication framework. It was designed to grant access, not to verify identity. **OpenID Connect (OIDC)** was built on top of OAuth2 to add authentication (identity).

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│  OAuth 2.0  →  Authorization  →  "What can you do?"  │
│       +                                              │
│  OIDC       →  Authentication →  "Who are you?"      │
│       =                                              │
│  Complete identity + access solution                  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 4. OAUTH2 ROLES (THE 4 PLAYERS)

```
┌───────────────────────────────────────────────────────────────────┐
│                      THE 4 OAUTH2 ROLES                          │
│                                                                   │
│  ┌──────────────┐          ┌─────────────────────┐               │
│  │ Resource     │          │ Client              │               │
│  │ Owner (User) │          │ (Your App)          │               │
│  │              │          │                     │               │
│  │ "I own the   │          │ "I want to access   │               │
│  │  data"       │          │  the user's data"   │               │
│  └──────┬───────┘          └──────────┬──────────┘               │
│         │                             │                           │
│         │ 1. Logs in &                │ 2. Sends client_id       │
│         │    gives consent            │    & requests token       │
│         ▼                             ▼                           │
│  ┌─────────────────────────────────────────────┐                 │
│  │         Authorization Server                 │                 │
│  │         (Google, Microsoft, Keycloak)         │                 │
│  │                                               │                 │
│  │  "I verify the user, issue tokens,           │                 │
│  │   and manage permissions"                     │                 │
│  └──────────────────┬────────────────────────────┘                │
│                     │                                             │
│                     │ 3. Returns access_token                     │
│                     ▼                                             │
│  ┌─────────────────────────────────────────────┐                 │
│  │         Resource Server                       │                │
│  │         (Gmail API, Graph API)                │                │
│  │                                               │                │
│  │  "I hold the protected data.                 │                │
│  │   Show me a valid token to access it."       │                │
│  └───────────────────────────────────────────────┘                │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### In Many Cases, Authorization Server and Resource Server Are the Same

| Scenario | Auth Server | Resource Server |
|----------|------------|----------------|
| Google login for your app | Google (`accounts.google.com`) | Google APIs (`googleapis.com`) |
| Microsoft login | Azure AD (`login.microsoftonline.com`) | Microsoft Graph (`graph.microsoft.com`) |
| Keycloak for your microservices | Keycloak (`keycloak.mycompany.com`) | Your own APIs |
| Your own OAuth server | Your auth service | Your API service |

---

## 5. TOKENS — ACCESS TOKEN, REFRESH TOKEN, ID TOKEN

### Access Token

- **Purpose**: Grants access to protected resources.
- **Lifespan**: Short (15 mins to 1 hour typically).
- **Format**: Can be opaque string OR JWT.
- **Sent as**: `Authorization: Bearer <token>` header.

```
Access Token (JWT format):
{
  "sub": "1234567890",        // user ID
  "iss": "https://accounts.google.com",  // who issued it
  "aud": "my-app-client-id",  // who it's intended for
  "exp": 1700003600,          // expiry time
  "scope": "email profile",   // what permissions it grants
  "iat": 1700000000           // issued at
}
```

### Refresh Token

- **Purpose**: Get a new access token WITHOUT re-login.
- **Lifespan**: Long (days, weeks, or months).
- **Format**: Usually opaque string (not JWT).
- **Stored**: Securely on the client (never exposed to browser JS).
- **Usage**: Sent to the token endpoint to get a fresh access token.

```
Access token expired?
       │
       ▼
Send refresh token to /token endpoint
       │
       ▼
Get new access token (+ optionally new refresh token)
       │
       ▼
Continue accessing resources with new token

NO re-login needed!
```

### ID Token (OIDC Only)

- **Purpose**: Contains user identity information (authentication).
- **Format**: Always a JWT.
- **Contains**: User's name, email, profile picture, etc.
- **NOT used** to access APIs — only to know WHO the user is.

```
ID Token (JWT payload):
{
  "sub": "1234567890",
  "name": "John Doe",
  "email": "john@example.com",
  "picture": "https://example.com/photo.jpg",
  "iss": "https://accounts.google.com",
  "aud": "my-app-client-id",
  "exp": 1700003600,
  "iat": 1700000000,
  "nonce": "abc123"           // prevents replay attacks
}
```

### Token Comparison

| Aspect | Access Token | Refresh Token | ID Token |
|--------|-------------|---------------|----------|
| Purpose | Access resources | Get new access token | Identify user |
| Lifespan | Short (minutes/hours) | Long (days/weeks) | Short (minutes/hours) |
| Format | JWT or opaque | Usually opaque | Always JWT |
| Sent to | Resource Server | Authorization Server | Your app only |
| Contains | Scopes, permissions | Nothing readable | User info (name, email) |
| Standard | OAuth2 | OAuth2 | OIDC (extension) |

---

## 6. OAUTH2 GRANT TYPES (FLOWS)

| Grant Type | Use Case | User Involved? | Recommended? |
|-----------|----------|----------------|-------------|
| **Authorization Code** | Web apps with server backend | Yes | **Yes** |
| **Authorization Code + PKCE** | SPAs, mobile apps, public clients | Yes | **Yes (preferred)** |
| **Client Credentials** | Machine-to-machine (no user) | No | **Yes** |
| **Resource Owner Password** | Legacy — user gives password directly | Yes | **Deprecated** |
| **Implicit** | Old SPAs (token in URL) | Yes | **Deprecated** |
| **Device Authorization** | Smart TVs, CLI tools, IoT | Yes (on another device) | **Yes** |
| **Refresh Token** | Renewing expired access tokens | No (already authenticated) | **Yes** |

```
Which flow should I use?

Is a USER involved?
├── NO  → Client Credentials Flow
│         (machine-to-machine, cron jobs, microservice-to-microservice)
│
└── YES → Does the app have a BACKEND SERVER?
          ├── YES → Authorization Code Flow
          │         (traditional web apps)
          │
          └── NO  → Is it a BROWSER app (SPA) or MOBILE app?
                    ├── YES → Authorization Code + PKCE
                    │         (React, Angular, iOS, Android)
                    │
                    └── Is it a device with NO BROWSER?
                          ├── YES → Device Authorization Flow
                          │         (Smart TV, CLI, IoT)
                          │
                          └── Authorization Code + PKCE (default)
```

---

## 7. AUTHORIZATION CODE FLOW (MOST COMMON)

> Used by: Web apps with a backend server. Most secure flow for user-involved authentication.

### Why This Flow Exists

The access token should NEVER be exposed in the browser URL or front-end. This flow ensures the token is exchanged **server-side** using the client secret.

### Complete Flow

```
┌──────┐       ┌──────────┐       ┌──────────────────┐       ┌────────────┐
│ User │       │  Client  │       │  Authorization   │       │  Resource  │
│      │       │ (Your App)│      │  Server (Google)  │       │  Server    │
└──┬───┘       └────┬─────┘       └────────┬─────────┘       └─────┬──────┘
   │                │                       │                       │
   │ 1. Click       │                       │                       │
   │ "Login with    │                       │                       │
   │  Google"       │                       │                       │
   │───────────────>│                       │                       │
   │                │                       │                       │
   │                │ 2. Redirect user to   │                       │
   │                │    /authorize         │                       │
   │<───────────────│    (with client_id,   │                       │
   │  (302 redirect)│     redirect_uri,     │                       │
   │                │     scope, state)     │                       │
   │                │                       │                       │
   │ 3. User sees   │                       │                       │
   │    login page  │                       │                       │
   │───────────────────────────────────────>│                       │
   │                │                       │                       │
   │ 4. User logs   │                       │                       │
   │    in & gives  │                       │                       │
   │    consent     │                       │                       │
   │───────────────────────────────────────>│                       │
   │                │                       │                       │
   │ 5. Redirect    │                       │                       │
   │    back to app │                       │                       │
   │    with CODE   │                       │                       │
   │<──────────────────────────────────────│                       │
   │  (redirect to  │                       │                       │
   │   callback URL │                       │                       │
   │   ?code=xxx    │                       │                       │
   │   &state=yyy)  │                       │                       │
   │                │                       │                       │
   │───────────────>│                       │                       │
   │                │                       │                       │
   │                │ 6. Exchange code for  │                       │
   │                │    tokens (POST       │                       │
   │                │    /token) — SERVER   │                       │
   │                │    SIDE with secret   │                       │
   │                │──────────────────────>│                       │
   │                │                       │                       │
   │                │ 7. Returns:           │                       │
   │                │    access_token       │                       │
   │                │    refresh_token      │                       │
   │                │    id_token (if OIDC) │                       │
   │                │<──────────────────────│                       │
   │                │                       │                       │
   │                │ 8. Use access_token   │                       │
   │                │    to call API        │                       │
   │                │───────────────────────────────────────────────>│
   │                │                       │                       │
   │                │ 9. Returns protected  │                       │
   │                │    resource           │                       │
   │                │<──────────────────────────────────────────────│
   │                │                       │                       │
   │ 10. Show data  │                       │                       │
   │<───────────────│                       │                       │
```

### Step-by-Step with Actual HTTP Requests

#### Step 1 — Redirect User to Authorization Endpoint

Open this URL in the browser (redirect the user):

```
GET https://authorization-server.com/authorize
    ?response_type=code
    &client_id=YOUR_CLIENT_ID
    &redirect_uri=https://yourapp.com/callback
    &scope=openid email profile
    &state=random_csrf_protection_string
```

| Parameter | Purpose |
|-----------|---------|
| `response_type=code` | Tells the server you want an authorization code |
| `client_id` | Identifies your app |
| `redirect_uri` | Where to send the user after login (must match registered URI) |
| `scope` | What permissions you're requesting |
| `state` | Random string to prevent CSRF attacks — you verify this on callback |

#### Step 2 — User Logs In & Gives Consent

The authorization server shows its login page. The user enters credentials and approves permissions. This happens entirely on the authorization server's domain — your app never sees the password.

#### Step 3 — Authorization Server Redirects Back with Code

```
HTTP/1.1 302 Found
Location: https://yourapp.com/callback
    ?code=AUTHORIZATION_CODE_HERE
    &state=random_csrf_protection_string
```

> **Important**: Verify that `state` matches what you sent in Step 1. If not, reject — it's a CSRF attack.

#### Step 4 — Exchange Code for Tokens (Server-Side)

**Postman Request:**

```
POST https://authorization-server.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=AUTHORIZATION_CODE_HERE
&redirect_uri=https://yourapp.com/callback
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
```

**Response:**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g...",
  "id_token": "eyJhbGciOiJSUzI1NiJ9...",
  "scope": "openid email profile"
}
```

#### Step 5 — Use Access Token to Call API

```
GET https://resource-server.com/api/userinfo
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
```

### Why Is This Secure?

```
Security features:
  1. User's password → NEVER seen by your app
  2. Authorization code → short-lived (usually 10 mins), single-use
  3. Code → token exchange → happens server-side (not in browser)
  4. Client secret → sent server-side (never exposed to browser)
  5. State parameter → prevents CSRF attacks
  6. Redirect URI → must match registered URI (prevents code theft)
```

---

## 8. AUTHORIZATION CODE FLOW WITH PKCE

> Used by: SPAs (React, Angular), mobile apps, desktop apps — any client that **cannot safely store a client secret**.

### Why PKCE?

SPAs and mobile apps run on the user's device. They **cannot keep a secret** — any secret embedded in JavaScript or a mobile APK can be extracted. PKCE adds security without needing a client secret.

**PKCE** = **P**roof **K**ey for **C**ode **E**xchange (pronounced "pixy")

### How PKCE Works

```
Before the flow starts, the client generates:

  1. code_verifier  = random string (43-128 characters)
     Example: "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

  2. code_challenge = BASE64URL(SHA256(code_verifier))
     Example: "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"

Flow:
  → Client sends code_challenge during authorization request
  → Authorization server stores it
  → Client sends code_verifier during token exchange
  → Authorization server verifies: SHA256(code_verifier) == stored code_challenge
  → If match → issue tokens
  → If no match → reject (someone stole the code but doesn't have the verifier)
```

### Complete Flow

```
┌──────────┐                        ┌──────────────────┐
│  Client  │                        │  Authorization   │
│  (SPA)   │                        │  Server          │
└────┬─────┘                        └────────┬─────────┘
     │                                       │
     │ 1. Generate:                          │
     │    code_verifier (random)             │
     │    code_challenge = SHA256(verifier)   │
     │                                       │
     │ 2. GET /authorize                     │
     │    ?response_type=code                │
     │    &client_id=xxx                     │
     │    &code_challenge=E9Mel...           │
     │    &code_challenge_method=S256        │
     │    &redirect_uri=...                  │
     │    &scope=openid profile              │
     │──────────────────────────────────────>│
     │                                       │
     │ 3. User logs in & consents            │
     │                                       │
     │ 4. Redirect with code                 │
     │<──────────────────────────────────────│
     │    ?code=AUTH_CODE&state=xxx          │
     │                                       │
     │ 5. POST /token                        │
     │    grant_type=authorization_code       │
     │    &code=AUTH_CODE                     │
     │    &code_verifier=dBjftJeZ...         │  ← sends original verifier
     │    &client_id=xxx                     │     (NO client_secret needed!)
     │    &redirect_uri=...                  │
     │──────────────────────────────────────>│
     │                                       │
     │    Server checks:                     │
     │    SHA256("dBjftJeZ...") == "E9Mel..."│
     │    MATCH ✅ → issue tokens            │
     │                                       │
     │ 6. Returns access_token,              │
     │    refresh_token, id_token            │
     │<──────────────────────────────────────│
```

### Postman — Step by Step

**Step 1 — Generate PKCE Values**

```
code_verifier:  dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
code_challenge: E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM

(You can use https://tonyxu-io.github.io/pkce-generator/ to generate these)
```

**Step 2 — Authorization Request (Browser)**

```
GET https://authorization-server.com/authorize
    ?response_type=code
    &client_id=YOUR_CLIENT_ID
    &redirect_uri=https://yourapp.com/callback
    &scope=openid email profile
    &state=random_string
    &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
    &code_challenge_method=S256
```

**Step 3 — Token Exchange (Postman)**

```
POST https://authorization-server.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=AUTHORIZATION_CODE
&redirect_uri=https://yourapp.com/callback
&client_id=YOUR_CLIENT_ID
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

> Notice: **No `client_secret`** needed! The `code_verifier` proves you're the same client that started the flow.

### PKCE vs No PKCE

| Aspect | Without PKCE | With PKCE |
|--------|-------------|-----------|
| Code stolen from redirect URL | Attacker can exchange code (if they have client_secret) | Attacker cannot exchange code (doesn't have code_verifier) |
| Client secret needed | Yes | No |
| Suitable for SPAs | No (can't hide secret) | Yes |
| Suitable for mobile | No (can't hide secret) | Yes |

---

## 9. CLIENT CREDENTIALS FLOW

> Used by: Machine-to-machine communication. No user involved — the app authenticates as itself.

### When to Use

- Microservice A calling Microservice B
- Cron jobs accessing APIs
- Backend services accessing other services
- Any server-to-server communication

### Flow

```
┌──────────────┐                    ┌──────────────────┐
│  Client      │                    │  Authorization   │
│  (Service A) │                    │  Server          │
└──────┬───────┘                    └────────┬─────────┘
       │                                     │
       │ 1. POST /token                      │
       │    grant_type=client_credentials     │
       │    &client_id=SERVICE_A_ID           │
       │    &client_secret=SERVICE_A_SECRET   │
       │    &scope=api.read                   │
       │────────────────────────────────────>│
       │                                     │
       │ 2. Returns access_token             │
       │    (NO refresh_token)               │
       │    (NO id_token — no user)          │
       │<────────────────────────────────────│
       │                                     │
       │ 3. Use token to call Service B      │
       │─────────────────────────────────────────> Resource Server
```

### Postman Request

```
POST https://authorization-server.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&scope=api.read api.write
```

**Response:**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "api.read api.write"
}
```

> **Note**: No `refresh_token` is returned — the client can always request a new token using its credentials. No `id_token` either — there's no user involved.

---

## 10. RESOURCE OWNER PASSWORD FLOW (DEPRECATED)

> **Deprecated in OAuth 2.1.** Included here for understanding legacy systems.

### Why It's Deprecated

The user gives their password **directly to the client app**. This defeats the entire purpose of OAuth2 (not sharing passwords).

### Flow

```
POST https://authorization-server.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=password
&username=john@example.com
&password=secret123
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&scope=openid email
```

**Response:**

```json
{
  "access_token": "eyJhbG...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "dGhpcyB..."
}
```

> **Do NOT use this flow for new applications.** Use Authorization Code + PKCE instead.

---

## 11. IMPLICIT FLOW (DEPRECATED)

> **Deprecated in OAuth 2.1.** Was used for SPAs before PKCE existed.

### Why It's Deprecated

The access token is returned **directly in the browser URL fragment** (`#access_token=xxx`), making it vulnerable to:
- URL history exposure
- Referer header leakage
- Browser extensions reading the URL

### Flow (Do Not Implement — For Reference Only)

```
GET https://authorization-server.com/authorize
    ?response_type=token              ← asks for token directly (no code)
    &client_id=YOUR_CLIENT_ID
    &redirect_uri=https://yourapp.com/callback
    &scope=email

Redirect back:
https://yourapp.com/callback#access_token=eyJhbG...&token_type=Bearer&expires_in=3600

Token is in the URL fragment (after #) — DANGEROUS!
```

> **Use Authorization Code + PKCE instead.**

---

## 12. DEVICE AUTHORIZATION FLOW

> Used by: Smart TVs, CLI tools, IoT devices, printers — devices that have no browser or limited input.

### Flow

```
┌───────────────┐                    ┌──────────────────┐
│  Device       │                    │  Authorization   │
│  (Smart TV)   │                    │  Server          │
└──────┬────────┘                    └────────┬─────────┘
       │                                      │
       │ 1. POST /device/code                 │
       │    client_id=TV_APP_ID               │
       │────────────────────────────────────>│
       │                                      │
       │ 2. Returns:                          │
       │    device_code: "ABCDEF..."          │
       │    user_code: "WDJB-MJHT"            │
       │    verification_uri:                 │
       │      "https://provider.com/device"   │
       │<────────────────────────────────────│
       │                                      │
       │ 3. TV displays:                      │
       │    "Go to https://provider.com/device│
       │     and enter code: WDJB-MJHT"       │
       │                                      │

    USER opens browser on phone/laptop:
    → Goes to https://provider.com/device
    → Enters code: WDJB-MJHT
    → Logs in & gives consent

       │                                      │
       │ 4. TV polls POST /token              │
       │    grant_type=                       │
       │      urn:ietf:params:oauth:          │
       │      grant-type:device_code          │
       │    &device_code=ABCDEF...            │
       │    &client_id=TV_APP_ID              │
       │────────────────────────────────────>│
       │                                      │
       │    (If user hasn't approved yet:     │
       │     { "error": "authorization_pending" })
       │                                      │
       │    (If user approved:)               │
       │ 5. Returns access_token              │
       │<────────────────────────────────────│
```

### Postman — Step 1: Get Device Code

```
POST https://authorization-server.com/device/code
Content-Type: application/x-www-form-urlencoded

client_id=YOUR_CLIENT_ID
&scope=openid email
```

### Postman — Step 2: Poll for Token

```
POST https://authorization-server.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:device_code
&device_code=DEVICE_CODE_FROM_STEP_1
&client_id=YOUR_CLIENT_ID
```

---

## 13. REFRESH TOKEN FLOW

### When Access Token Expires

```
┌──────────────┐                    ┌──────────────────┐
│  Client      │                    │  Authorization   │
│  (Your App)  │                    │  Server          │
└──────┬───────┘                    └────────┬─────────┘
       │                                     │
       │ 1. API call with expired token      │
       │────────────────────────────────────────> Resource Server
       │                                          │
       │ 2. Returns 401 Unauthorized              │
       │<────────────────────────────────────────│
       │                                     │
       │ 3. POST /token                      │
       │    grant_type=refresh_token          │
       │    &refresh_token=REFRESH_TOKEN      │
       │    &client_id=YOUR_CLIENT_ID        │
       │    &client_secret=YOUR_SECRET        │
       │────────────────────────────────────>│
       │                                     │
       │ 4. Returns new tokens:              │
       │    access_token (new)               │
       │    refresh_token (new — rotated)    │
       │<────────────────────────────────────│
       │                                     │
       │ 5. Retry API call with new token    │
       │────────────────────────────────────────> Resource Server
       │                                          │
       │ 6. Returns data ✅                       │
       │<────────────────────────────────────────│
```

### Postman Request

```
POST https://authorization-server.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=YOUR_REFRESH_TOKEN
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
```

**Response:**

```json
{
  "access_token": "new_access_token...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "new_refresh_token...",
  "scope": "openid email profile"
}
```

> **Refresh Token Rotation**: Many providers return a NEW refresh token with each use and invalidate the old one. This prevents stolen refresh tokens from being reused.

---

## 14. OPENID CONNECT (OIDC) — IDENTITY LAYER ON OAUTH2

### What is OIDC?

```
OAuth 2.0  = "This app can access your photos"  (AUTHORIZATION)
OIDC       = "This user is John Doe"             (AUTHENTICATION)

OIDC = OAuth 2.0 + Identity Layer
```

### What OIDC Adds on Top of OAuth2

| Feature | OAuth2 | OIDC |
|---------|--------|------|
| Access Token | Yes | Yes |
| Refresh Token | Yes | Yes |
| **ID Token** | No | **Yes** |
| **UserInfo Endpoint** | No | **Yes** |
| **Standard Scopes** (openid, profile, email) | No | **Yes** |
| **Discovery Document** (`.well-known`) | No | **Yes** |
| Standard Claims (name, email, picture) | No | **Yes** |

### Key OIDC Scopes

| Scope | What It Returns in ID Token |
|-------|-----------------------------|
| `openid` | **Required** — tells the server you want OIDC (returns `sub`) |
| `profile` | `name`, `family_name`, `given_name`, `picture`, `locale` |
| `email` | `email`, `email_verified` |
| `address` | `address` (structured) |
| `phone` | `phone_number`, `phone_number_verified` |

### UserInfo Endpoint

After getting an access token, you can call the UserInfo endpoint to get the user's profile:

```
GET https://authorization-server.com/userinfo
Authorization: Bearer ACCESS_TOKEN
```

**Response:**

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "given_name": "John",
  "family_name": "Doe",
  "email": "john@example.com",
  "email_verified": true,
  "picture": "https://example.com/photo.jpg"
}
```

### How to Request OIDC (vs Plain OAuth2)

```
Just add "openid" to the scope:

OAuth2 only (no identity):
  scope=email photos

OIDC (with identity):
  scope=openid email profile     ← "openid" triggers OIDC
```

When `openid` is in the scope, the token response will include an `id_token` alongside the `access_token`.

---

## 15. SCOPES

### What Are Scopes?

Scopes define **what permissions** the token grants. Think of them as a checklist of allowed actions.

```
Requesting scope:
  scope=read:email write:calendar

Means:
  ✅ Can read emails
  ✅ Can write to calendar
  ❌ Cannot delete emails
  ❌ Cannot read contacts
```

### Common Scopes by Provider

| Provider | Common Scopes |
|----------|--------------|
| **Google** | `openid`, `email`, `profile`, `https://www.googleapis.com/auth/gmail.readonly` |
| **Microsoft** | `openid`, `email`, `profile`, `User.Read`, `Mail.Read`, `Calendars.Read` |
| **GitHub** | `user`, `repo`, `read:org`, `gist`, `notifications` |
| **Keycloak** | `openid`, `email`, `profile`, `roles` (custom) |

### Scope vs Role vs Authority

| Concept | Where | Meaning |
|---------|-------|---------|
| **Scope** | OAuth2 token | What the TOKEN is allowed to do |
| **Role** | Application | What the USER is allowed to do |
| **Authority** | Application | Fine-grained permission for the USER |

```
Example:
  Token scope: "read:products"     → Token can read products
  User role: "ADMIN"               → User is an admin
  User authority: "DELETE_PRODUCT" → User can delete products

  Even if the user is ADMIN, if the token only has "read:products" scope,
  the token CANNOT delete products.

  Scope limits the TOKEN, not the user.
```

---

## 16. TOKEN INTROSPECTION & REVOCATION

### Token Introspection — "Is This Token Still Valid?"

Used when tokens are **opaque** (not JWT) — the resource server can't decode them, so it asks the authorization server.

```
POST https://authorization-server.com/introspect
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

token=ACCESS_TOKEN_TO_CHECK
```

**Response (Active):**

```json
{
  "active": true,
  "sub": "john",
  "client_id": "my-app",
  "scope": "openid email",
  "exp": 1700003600,
  "iat": 1700000000
}
```

**Response (Expired/Revoked):**

```json
{
  "active": false
}
```

### Token Revocation — "Invalidate This Token"

Used when a user logs out — tells the authorization server to invalidate the token.

```
POST https://authorization-server.com/revoke
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

token=TOKEN_TO_REVOKE
&token_type_hint=refresh_token
```

**Response:** `200 OK` (no body)

> **Best practice**: Revoke the **refresh token** on logout. The access token will expire on its own (it's short-lived).

---

## 17. JWKS & TOKEN VALIDATION

### How Token Validation Works (Asymmetric Keys)

OAuth2 providers sign tokens with a **private key** and publish the **public key** at a JWKS (JSON Web Key Set) endpoint. Anyone can verify the token using the public key.

```
Authorization Server                     Resource Server (Your API)
       │                                        │
       │ Signs token with PRIVATE key            │
       │                                        │
       │ Publishes PUBLIC key at                │
       │ /.well-known/jwks.json                 │
       │                                        │
       │         Token sent to API              │
       │ ─────────────────────────────────────> │
       │                                        │
       │                     Fetches public key │
       │ <───────────────────────────────────── │
       │  (GET /.well-known/jwks.json)          │
       │                                        │
       │                     Verifies signature │
       │                     with PUBLIC key    │
       │                     Checks expiry      │
       │                     Checks audience    │
       │                     VALID ✅           │
```

### JWKS Endpoint Response

```
GET https://authorization-server.com/.well-known/jwks.json
```

```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "key-id-1",
      "use": "sig",
      "alg": "RS256",
      "n": "0vx7agoebGc...long_base64_modulus...",
      "e": "AQAB"
    }
  ]
}
```

### Self-Issued JWT vs Provider JWT — Validation Comparison

| Aspect | Self-Issued (Your App) | Provider (Google, Keycloak) |
|--------|----------------------|---------------------------|
| Signing | Symmetric (HMAC — same secret key) | Asymmetric (RSA/EC — private/public key pair) |
| Validation needs | Your secret key | Provider's public key (from JWKS) |
| Key distribution | Secret — never shared | Public — anyone can fetch from JWKS |
| If key is leaked | Attacker can forge tokens | Can only verify, not forge (public key is useless for signing) |

---

## 18. PROVIDER: GOOGLE

### Setup

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a project → APIs & Services → Credentials
3. Create **OAuth 2.0 Client ID**
4. Set authorized redirect URIs (e.g., `https://yourapp.com/callback`)
5. Note down: `client_id` and `client_secret`

### Endpoints

| Endpoint | URL |
|----------|-----|
| Discovery | `https://accounts.google.com/.well-known/openid-configuration` |
| Authorization | `https://accounts.google.com/o/oauth2/v2/auth` |
| Token | `https://oauth2.googleapis.com/token` |
| UserInfo | `https://openidconnect.googleapis.com/v1/userinfo` |
| JWKS | `https://www.googleapis.com/oauth2/v3/certs` |
| Revocation | `https://oauth2.googleapis.com/revoke` |

### Step 1 — Authorization Request (Browser)

```
https://accounts.google.com/o/oauth2/v2/auth
    ?response_type=code
    &client_id=YOUR_CLIENT_ID.apps.googleusercontent.com
    &redirect_uri=https://yourapp.com/callback
    &scope=openid email profile
    &state=random_state_string
    &access_type=offline             ← needed to get refresh_token
    &prompt=consent                  ← force consent screen (to get refresh_token)
```

### Step 2 — Exchange Code for Tokens (Postman)

```
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=4/0AX4XfWi...
&redirect_uri=https://yourapp.com/callback
&client_id=YOUR_CLIENT_ID.apps.googleusercontent.com
&client_secret=GOCSPX-xxxxx
```

**Response:**

```json
{
  "access_token": "ya29.a0AfH6SM...",
  "expires_in": 3599,
  "refresh_token": "1//0eXx...",
  "scope": "openid https://www.googleapis.com/auth/userinfo.email https://www.googleapis.com/auth/userinfo.profile",
  "token_type": "Bearer",
  "id_token": "eyJhbGciOiJSUzI1NiJ9..."
}
```

### Step 3 — Get User Info (Postman)

```
GET https://openidconnect.googleapis.com/v1/userinfo
Authorization: Bearer ya29.a0AfH6SM...
```

**Response:**

```json
{
  "sub": "1098765432",
  "name": "John Doe",
  "given_name": "John",
  "family_name": "Doe",
  "picture": "https://lh3.googleusercontent.com/...",
  "email": "john@gmail.com",
  "email_verified": true,
  "locale": "en"
}
```

### Step 4 — Refresh Token (Postman)

```
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=1//0eXx...
&client_id=YOUR_CLIENT_ID.apps.googleusercontent.com
&client_secret=GOCSPX-xxxxx
```

### Step 5 — Revoke Token (Postman)

```
POST https://oauth2.googleapis.com/revoke
Content-Type: application/x-www-form-urlencoded

token=ya29.a0AfH6SM...
```

### Google-Specific Notes

- `access_type=offline` is required to receive a `refresh_token`.
- `prompt=consent` forces the consent screen — without it, Google returns `refresh_token` only on the FIRST authorization.
- Google access tokens expire in **1 hour**.
- Google does NOT support token introspection — tokens are JWTs, validate locally using JWKS.

---

## 19. PROVIDER: MICROSOFT (AZURE AD / ENTRA ID)

### Setup

1. Go to [Azure Portal](https://portal.azure.com/) → Azure Active Directory → App registrations
2. Create **New registration**
3. Set redirect URI (e.g., `https://yourapp.com/callback`)
4. Under Certificates & Secrets → Create a **Client Secret**
5. Note down: `client_id` (Application ID), `client_secret`, `tenant_id`

### Tenant Types

| Tenant | Endpoint | Who Can Login |
|--------|----------|--------------|
| Single tenant | `/YOUR_TENANT_ID/` | Only your org |
| Multi-tenant | `/organizations/` | Any Azure AD org |
| Personal + Work | `/common/` | Any Microsoft account |
| Personal only | `/consumers/` | Personal Microsoft accounts |

### Endpoints (Replace `{tenant}` with your tenant ID or `common`)

| Endpoint | URL |
|----------|-----|
| Discovery | `https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration` |
| Authorization | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize` |
| Token | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token` |
| UserInfo | `https://graph.microsoft.com/oidc/userinfo` |
| JWKS | `https://login.microsoftonline.com/{tenant}/discovery/v2.0/keys` |
| Logout | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/logout` |

### Step 1 — Authorization Request (Browser)

```
https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize
    ?response_type=code
    &client_id=YOUR_CLIENT_ID
    &redirect_uri=https://yourapp.com/callback
    &scope=openid email profile User.Read
    &state=random_state_string
    &response_mode=query
```

### Step 2 — Exchange Code for Tokens (Postman)

```
POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=OAQABAAIAAAAm...
&redirect_uri=https://yourapp.com/callback
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&scope=openid email profile User.Read
```

**Response:**

```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLC...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "OAQABAAAAAAm...",
  "id_token": "eyJ0eXAiOiJKV1QiLC...",
  "scope": "openid profile email User.Read"
}
```

### Step 3 — Call Microsoft Graph API (Postman)

```
GET https://graph.microsoft.com/v1.0/me
Authorization: Bearer eyJ0eXAiOiJKV1QiLC...
```

**Response:**

```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users/$entity",
  "displayName": "John Doe",
  "givenName": "John",
  "surname": "Doe",
  "mail": "john@contoso.com",
  "userPrincipalName": "john@contoso.com",
  "id": "abcd-1234-efgh-5678"
}
```

### Step 4 — Client Credentials (Machine-to-Machine)

```
POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&scope=https://graph.microsoft.com/.default
```

> **Note**: For client credentials, use `/.default` suffix on the scope (e.g., `https://graph.microsoft.com/.default`).

### Microsoft-Specific Notes

- Microsoft uses **v2.0** endpoints — avoid v1.0 for new apps.
- Scopes use the format `resource/permission` (e.g., `User.Read`, `Mail.Send`).
- `/.default` scope = all permissions registered in the app.
- Tokens can be for Microsoft Graph API OR your own API (different `scope`).
- Azure AD supports **certificate-based** client authentication (more secure than client_secret).
- Access tokens for Graph API are **always JWT**.

---

## 20. PROVIDER: GITHUB

### Setup

1. Go to [GitHub Settings](https://github.com/settings/developers) → OAuth Apps → New OAuth App
2. Set Authorization callback URL (e.g., `https://yourapp.com/callback`)
3. Note down: `client_id` and `client_secret`

### Endpoints

| Endpoint | URL |
|----------|-----|
| Authorization | `https://github.com/login/oauth/authorize` |
| Token | `https://github.com/login/oauth/access_token` |
| User API | `https://api.github.com/user` |
| User Emails | `https://api.github.com/user/emails` |

> **Note**: GitHub does NOT support OIDC — it's OAuth2 only. No `id_token`, no `.well-known`, no JWKS.

### Step 1 — Authorization Request (Browser)

```
https://github.com/login/oauth/authorize
    ?client_id=YOUR_CLIENT_ID
    &redirect_uri=https://yourapp.com/callback
    &scope=user:email read:org
    &state=random_state_string
```

### Step 2 — Exchange Code for Token (Postman)

```
POST https://github.com/login/oauth/access_token
Accept: application/json
Content-Type: application/x-www-form-urlencoded

client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&code=AUTHORIZATION_CODE
&redirect_uri=https://yourapp.com/callback
```

> **Important**: Add `Accept: application/json` header, otherwise GitHub returns the response as form-encoded text.

**Response:**

```json
{
  "access_token": "gho_xxxxxxxxxxxx",
  "token_type": "bearer",
  "scope": "user:email,read:org"
}
```

> **Note**: GitHub does NOT return `refresh_token` for regular OAuth apps. Tokens don't expire unless revoked. For GitHub Apps, tokens expire in 8 hours and include refresh tokens.

### Step 3 — Get User Info (Postman)

```
GET https://api.github.com/user
Authorization: Bearer gho_xxxxxxxxxxxx
```

**Response:**

```json
{
  "login": "johndoe",
  "id": 1234567,
  "name": "John Doe",
  "email": "john@example.com",
  "avatar_url": "https://avatars.githubusercontent.com/u/1234567",
  "html_url": "https://github.com/johndoe",
  "type": "User",
  "bio": "Developer",
  "public_repos": 42,
  "followers": 100,
  "following": 50
}
```

### Get User Emails (If Email Is Private)

```
GET https://api.github.com/user/emails
Authorization: Bearer gho_xxxxxxxxxxxx
```

**Response:**

```json
[
  {
    "email": "john@example.com",
    "primary": true,
    "verified": true,
    "visibility": "public"
  },
  {
    "email": "john@work.com",
    "primary": false,
    "verified": true,
    "visibility": null
  }
]
```

### GitHub Scopes

| Scope | Access |
|-------|--------|
| `user` | Read/write access to profile info |
| `user:email` | Read user email addresses |
| `read:user` | Read-only access to profile info |
| `repo` | Full access to repositories |
| `read:org` | Read org membership |
| `gist` | Create gists |
| `notifications` | Read notifications |
| `admin:org` | Full org access |

### GitHub-Specific Notes

- GitHub is **NOT an OIDC provider** — OAuth2 only.
- No `id_token`, no JWKS, no `.well-known/openid-configuration`.
- Regular OAuth app tokens **don't expire** (unless revoked).
- GitHub App tokens expire in **8 hours** and support refresh tokens.
- Access tokens are **opaque strings** (not JWTs).
- Always add `Accept: application/json` when requesting tokens.

---

## 21. PROVIDER: KEYCLOAK (SELF-HOSTED)

### What is Keycloak?

- **Open-source** Identity and Access Management (IAM) solution by Red Hat.
- Supports OAuth2, OIDC, SAML 2.0.
- You host it yourself — full control.
- Supports user management, social login, MFA, LDAP/AD integration.
- Ideal for enterprise and microservice architectures.

### Setup

1. Run Keycloak: `docker run -p 8080:8080 -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak start-dev`
2. Go to `http://localhost:8080/admin` → Login with admin/admin
3. Create a **Realm** (e.g., `my-realm`)
4. Create a **Client** (e.g., `my-app`)
   - Client authentication: ON (for confidential clients) or OFF (for public clients)
   - Set Valid redirect URIs (e.g., `https://yourapp.com/callback`)
5. Create **Users** and assign **Roles**
6. Note down: `client_id`, `client_secret`, `realm name`

### Endpoints (Replace `{realm}` with your realm name)

| Endpoint | URL |
|----------|-----|
| Discovery | `http://localhost:8080/realms/{realm}/.well-known/openid-configuration` |
| Authorization | `http://localhost:8080/realms/{realm}/protocol/openid-connect/auth` |
| Token | `http://localhost:8080/realms/{realm}/protocol/openid-connect/token` |
| UserInfo | `http://localhost:8080/realms/{realm}/protocol/openid-connect/userinfo` |
| JWKS | `http://localhost:8080/realms/{realm}/protocol/openid-connect/certs` |
| Introspection | `http://localhost:8080/realms/{realm}/protocol/openid-connect/token/introspect` |
| Revocation | `http://localhost:8080/realms/{realm}/protocol/openid-connect/revoke` |
| Logout | `http://localhost:8080/realms/{realm}/protocol/openid-connect/logout` |
| Admin API | `http://localhost:8080/admin/realms/{realm}/` |

### Step 1 — Authorization Code Flow (Browser)

```
http://localhost:8080/realms/my-realm/protocol/openid-connect/auth
    ?response_type=code
    &client_id=my-app
    &redirect_uri=https://yourapp.com/callback
    &scope=openid email profile
    &state=random_state_string
```

### Step 2 — Exchange Code for Tokens (Postman)

```
POST http://localhost:8080/realms/my-realm/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=AUTHORIZATION_CODE
&redirect_uri=https://yourapp.com/callback
&client_id=my-app
&client_secret=YOUR_CLIENT_SECRET
```

**Response:**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiJ9...",
  "expires_in": 300,
  "refresh_expires_in": 1800,
  "refresh_token": "eyJhbGciOiJIUzI1NiJ9...",
  "token_type": "Bearer",
  "id_token": "eyJhbGciOiJSUzI1NiJ9...",
  "not-before-policy": 0,
  "session_state": "abcdef-1234...",
  "scope": "openid email profile"
}
```

### Step 3 — Resource Owner Password (Direct Login — For Testing)

```
POST http://localhost:8080/realms/my-realm/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=password
&client_id=my-app
&client_secret=YOUR_CLIENT_SECRET
&username=john
&password=john123
&scope=openid
```

> This flow is useful for **Postman testing** since you can't follow browser redirects easily. Do NOT use in production.

### Step 4 — Client Credentials (Postman)

```
POST http://localhost:8080/realms/my-realm/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=my-service
&client_secret=SERVICE_SECRET
&scope=openid
```

### Step 5 — Token Introspection (Postman)

```
POST http://localhost:8080/realms/my-realm/protocol/openid-connect/token/introspect
Content-Type: application/x-www-form-urlencoded

client_id=my-app
&client_secret=YOUR_CLIENT_SECRET
&token=ACCESS_TOKEN_TO_CHECK
```

**Response:**

```json
{
  "active": true,
  "sub": "user-uuid-here",
  "email_verified": true,
  "preferred_username": "john",
  "email": "john@example.com",
  "realm_access": {
    "roles": ["admin", "user"]
  },
  "resource_access": {
    "my-app": {
      "roles": ["app-admin"]
    }
  },
  "scope": "openid email profile",
  "client_id": "my-app",
  "token_type": "Bearer",
  "exp": 1700003600
}
```

### Step 6 — Refresh Token (Postman)

```
POST http://localhost:8080/realms/my-realm/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=YOUR_REFRESH_TOKEN
&client_id=my-app
&client_secret=YOUR_CLIENT_SECRET
```

### Step 7 — Revoke Token (Postman)

```
POST http://localhost:8080/realms/my-realm/protocol/openid-connect/revoke
Content-Type: application/x-www-form-urlencoded

client_id=my-app
&client_secret=YOUR_CLIENT_SECRET
&token=TOKEN_TO_REVOKE
&token_type_hint=refresh_token
```

### Step 8 — Logout (Postman)

```
POST http://localhost:8080/realms/my-realm/protocol/openid-connect/logout
Content-Type: application/x-www-form-urlencoded

client_id=my-app
&client_secret=YOUR_CLIENT_SECRET
&refresh_token=YOUR_REFRESH_TOKEN
```

### Keycloak Roles in Tokens

Keycloak includes roles in the JWT access token:

```json
{
  "realm_access": {
    "roles": ["admin", "user", "offline_access"]
  },
  "resource_access": {
    "my-app": {
      "roles": ["app-admin", "app-user"]
    },
    "account": {
      "roles": ["manage-account"]
    }
  }
}
```

| Role Type | Meaning | Token Claim |
|-----------|---------|-------------|
| **Realm Role** | Global role across all clients | `realm_access.roles` |
| **Client Role** | Role specific to a client/app | `resource_access.{client}.roles` |

### Keycloak Admin REST API (Manage Users, Roles, Clients)

First, get an admin token:

```
POST http://localhost:8080/realms/master/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=password
&client_id=admin-cli
&username=admin
&password=admin
```

Then use the admin API:

```
# List all users
GET http://localhost:8080/admin/realms/my-realm/users
Authorization: Bearer ADMIN_TOKEN

# Create a user
POST http://localhost:8080/admin/realms/my-realm/users
Authorization: Bearer ADMIN_TOKEN
Content-Type: application/json

{
  "username": "newuser",
  "email": "new@example.com",
  "enabled": true,
  "credentials": [
    {
      "type": "password",
      "value": "password123",
      "temporary": false
    }
  ]
}

# List realm roles
GET http://localhost:8080/admin/realms/my-realm/roles
Authorization: Bearer ADMIN_TOKEN

# Assign role to user
POST http://localhost:8080/admin/realms/my-realm/users/{user-id}/role-mappings/realm
Authorization: Bearer ADMIN_TOKEN
Content-Type: application/json

[
  {
    "id": "role-uuid",
    "name": "admin"
  }
]
```

### Keycloak-Specific Notes

- Keycloak access tokens **expire in 5 minutes** by default (configurable).
- Keycloak refresh tokens expire in **30 minutes** by default.
- Supports **all OAuth2/OIDC flows** including Device Authorization.
- Supports **social login** (Google, GitHub, Facebook, etc.) as identity brokers.
- Supports **LDAP/Active Directory** integration.
- Supports **fine-grained authorization** (UMA 2.0, resource-based permissions).
- Has a built-in **user management console**.
- Supports **multi-tenancy** via Realms.

---

## 22. PROVIDER: OKTA

### What is Okta?

- **Cloud-based** Identity and Access Management (IDaaS) platform.
- Supports OAuth2, OIDC, SAML 2.0.
- Fully managed — no self-hosting needed.
- Supports user management, social login, MFA, LDAP/AD integration.
- Popular in enterprise environments.
- Has a free developer tier.

### Setup

1. Sign up at [Okta Developer](https://developer.okta.com/signup/)
2. Go to **Applications** → **Create App Integration**
3. Choose **OIDC - OpenID Connect** → **Web Application** (or SPA/Native)
4. Set sign-in redirect URI (e.g., `https://yourapp.com/callback`)
5. Set sign-out redirect URI (e.g., `https://yourapp.com`)
6. Note down: `client_id`, `client_secret`, `Okta domain` (e.g., `dev-12345.okta.com`)

### Endpoints (Replace `{okta-domain}` with your domain, e.g., `dev-12345.okta.com`)

| Endpoint | URL |
|----------|-----|
| Discovery | `https://{okta-domain}/.well-known/openid-configuration` |
| Authorization | `https://{okta-domain}/oauth2/default/v1/authorize` |
| Token | `https://{okta-domain}/oauth2/default/v1/token` |
| UserInfo | `https://{okta-domain}/oauth2/default/v1/userinfo` |
| JWKS | `https://{okta-domain}/oauth2/default/v1/keys` |
| Introspection | `https://{okta-domain}/oauth2/default/v1/introspect` |
| Revocation | `https://{okta-domain}/oauth2/default/v1/revoke` |
| Logout | `https://{okta-domain}/oauth2/default/v1/logout` |
| Users API | `https://{okta-domain}/api/v1/users` |

> **Note**: `/oauth2/default` is the **default authorization server**. You can create custom authorization servers for different APIs.

### Step 1 — Authorization Code Flow (Browser)

```
https://{okta-domain}/oauth2/default/v1/authorize
    ?response_type=code
    &client_id=YOUR_CLIENT_ID
    &redirect_uri=https://yourapp.com/callback
    &scope=openid email profile
    &state=random_state_string
```

### Step 2 — Exchange Code for Tokens (Postman)

```
POST https://{okta-domain}/oauth2/default/v1/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=AUTHORIZATION_CODE
&redirect_uri=https://yourapp.com/callback
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
```

**Response:**

```json
{
  "access_token": "eyJraWQiOiJxMm5...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "nkMZ8wReF_Ztse...",
  "id_token": "eyJraWQiOiJxMm5...",
  "scope": "openid email profile"
}
```

### Step 3 — Get User Info (Postman)

```
GET https://{okta-domain}/oauth2/default/v1/userinfo
Authorization: Bearer eyJraWQiOiJxMm5...
```

**Response:**

```json
{
  "sub": "00u1a2b3c4d5e6f7g8",
  "name": "John Doe",
  "given_name": "John",
  "family_name": "Doe",
  "email": "john@example.com",
  "email_verified": true,
  "locale": "en-US",
  "preferred_username": "john@example.com",
  "zoneinfo": "America/Los_Angeles",
  "updated_at": 1700000000
}
```

### Step 4 — Client Credentials (Machine-to-Machine)

```
POST https://{okta-domain}/oauth2/default/v1/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&scope=custom_api_scope
```

> **Note**: For client credentials, you must use **custom scopes** defined in your authorization server. Standard OIDC scopes (`openid`, `profile`, `email`) are NOT allowed with client credentials in Okta.

### Step 5 — Token Introspection (Postman)

```
POST https://{okta-domain}/oauth2/default/v1/introspect
Content-Type: application/x-www-form-urlencoded

client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&token=ACCESS_TOKEN_TO_CHECK
&token_type_hint=access_token
```

**Response:**

```json
{
  "active": true,
  "sub": "john@example.com",
  "client_id": "YOUR_CLIENT_ID",
  "scope": "openid email profile",
  "exp": 1700003600,
  "iat": 1700000000,
  "iss": "https://{okta-domain}/oauth2/default",
  "uid": "00u1a2b3c4d5e6f7g8",
  "username": "john@example.com",
  "token_type": "Bearer"
}
```

### Step 6 — Refresh Token (Postman)

```
POST https://{okta-domain}/oauth2/default/v1/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=YOUR_REFRESH_TOKEN
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&scope=openid email profile
```

### Step 7 — Revoke Token (Postman)

```
POST https://{okta-domain}/oauth2/default/v1/revoke
Content-Type: application/x-www-form-urlencoded

client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&token=TOKEN_TO_REVOKE
&token_type_hint=refresh_token
```

### Step 8 — Resource Owner Password (Testing Only)

```
POST https://{okta-domain}/oauth2/default/v1/token
Content-Type: application/x-www-form-urlencoded

grant_type=password
&username=john@example.com
&password=secret123
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&scope=openid email profile
```

> **Note**: This flow must be explicitly enabled in Okta. Go to **Security** → **API** → **Authorization Servers** → **default** → **Access Policies** → edit the rule to allow `Resource Owner Password` grant.

### Custom Authorization Servers

Okta supports multiple authorization servers for different APIs:

```
Default server:
  https://{okta-domain}/oauth2/default/v1/token

Custom server (e.g., for your API):
  https://{okta-domain}/oauth2/{custom-server-id}/v1/token

Org-level server (for Okta APIs only):
  https://{okta-domain}/oauth2/v1/token
```

| Server | Use Case |
|--------|----------|
| `default` | General-purpose — good for most apps |
| Custom | Specific API with custom scopes and claims |
| Org-level | Accessing Okta's own APIs (user management, etc.) |

### Adding Custom Claims to Tokens

In Okta Admin → Security → API → Authorization Servers → Claims:

- You can add custom claims (e.g., `department`, `role`) to access tokens and ID tokens.
- Claims can be sourced from user profile attributes, groups, or expressions.

Example token with custom claims:

```json
{
  "sub": "john@example.com",
  "groups": ["admin", "developers"],
  "department": "Engineering",
  "scp": ["openid", "email", "profile"],
  "iss": "https://dev-12345.okta.com/oauth2/default",
  "exp": 1700003600
}
```

### Okta Users API (Admin Operations)

First, create an API token: **Security** → **API** → **Tokens** → **Create Token**

```
# List all users
GET https://{okta-domain}/api/v1/users
Authorization: SSWS YOUR_API_TOKEN

# Get a specific user
GET https://{okta-domain}/api/v1/users/john@example.com
Authorization: SSWS YOUR_API_TOKEN

# Create a user
POST https://{okta-domain}/api/v1/users?activate=true
Authorization: SSWS YOUR_API_TOKEN
Content-Type: application/json

{
  "profile": {
    "firstName": "John",
    "lastName": "Doe",
    "email": "john@example.com",
    "login": "john@example.com"
  },
  "credentials": {
    "password": {
      "value": "Password123!"
    }
  }
}

# Deactivate a user
POST https://{okta-domain}/api/v1/users/{userId}/lifecycle/deactivate
Authorization: SSWS YOUR_API_TOKEN

# List groups
GET https://{okta-domain}/api/v1/groups
Authorization: SSWS YOUR_API_TOKEN
```

> **Note**: Okta admin API uses `SSWS` token prefix (not `Bearer`).

### Okta-Specific Notes

- Okta access tokens expire in **1 hour** by default (configurable per authorization server).
- Okta refresh tokens expire in **90 days** by default.
- Supports **all OAuth2/OIDC flows** including Device Authorization and PKCE.
- Supports **social login** (Google, Facebook, Apple, Microsoft, etc.).
- Supports **LDAP/Active Directory** integration.
- Has a **free developer tier** (up to 15,000 monthly active users).
- Supports **Okta Groups** as claims in tokens — useful for role-based access.
- Admin API uses `SSWS` authorization scheme (NOT Bearer).
- Supports **Inline Hooks** (webhooks triggered during auth flows for custom logic).
- Supports **DPoP** (Demonstration of Proof of Possession) for token binding.

---

## 23. WELL-KNOWN ENDPOINTS (DISCOVERY)

### What is the Discovery Document?

OIDC providers publish a JSON document at `/.well-known/openid-configuration` that tells you all the endpoints, supported features, and signing algorithms.

```
GET https://authorization-server.com/.well-known/openid-configuration
```

### Example Response (Keycloak)

```json
{
  "issuer": "http://localhost:8080/realms/my-realm",
  "authorization_endpoint": "http://localhost:8080/realms/my-realm/protocol/openid-connect/auth",
  "token_endpoint": "http://localhost:8080/realms/my-realm/protocol/openid-connect/token",
  "userinfo_endpoint": "http://localhost:8080/realms/my-realm/protocol/openid-connect/userinfo",
  "jwks_uri": "http://localhost:8080/realms/my-realm/protocol/openid-connect/certs",
  "introspection_endpoint": "http://localhost:8080/realms/my-realm/protocol/openid-connect/token/introspect",
  "revocation_endpoint": "http://localhost:8080/realms/my-realm/protocol/openid-connect/revoke",
  "end_session_endpoint": "http://localhost:8080/realms/my-realm/protocol/openid-connect/logout",
  "grant_types_supported": [
    "authorization_code",
    "client_credentials",
    "refresh_token",
    "password",
    "urn:ietf:params:oauth:grant-type:device_code"
  ],
  "response_types_supported": ["code", "id_token", "code id_token"],
  "scopes_supported": ["openid", "email", "profile", "address", "phone"],
  "token_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post", "private_key_jwt"],
  "id_token_signing_alg_values_supported": ["RS256", "ES256"],
  "claims_supported": ["sub", "name", "email", "preferred_username"]
}
```

### Discovery URLs by Provider

| Provider | Discovery URL |
|----------|-------------- |
| Google | `https://accounts.google.com/.well-known/openid-configuration` |
| Microsoft | `https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration` |
| Keycloak | `http://localhost:8080/realms/{realm}/.well-known/openid-configuration` |
| Okta | `https://{okta-domain}/oauth2/default/.well-known/openid-configuration` |
| GitHub | **Not available** (GitHub is not OIDC) |
| Auth0 | `https://{domain}/.well-known/openid-configuration` |

> **Pro tip**: Always start by fetching the discovery document. It tells you exactly which endpoints to use, what scopes are supported, and what algorithms are available.

---

## 24. SECURITY BEST PRACTICES

### Token Security

1. **Use short-lived access tokens** (5–60 minutes).
2. **Use refresh token rotation** — issue a new refresh token each time and invalidate the old one.
3. **Store tokens securely**:
   - Backend: In-memory or encrypted storage
   - Browser: `httpOnly` + `Secure` + `SameSite` cookies (NOT localStorage)
   - Mobile: Secure storage (Keychain on iOS, Keystore on Android)
4. **Never put tokens in URLs** — they end up in browser history and server logs.
5. **Validate tokens properly**:
   - Check signature (using JWKS public key)
   - Check `exp` (not expired)
   - Check `iss` (correct issuer)
   - Check `aud` (intended for your app)

### Flow Security

6. **Always use PKCE** — even for confidential clients (recommended by OAuth 2.1).
7. **Always validate the `state` parameter** — prevents CSRF attacks.
8. **Use exact redirect URI matching** — no wildcards.
9. **Never use the Implicit flow** — use Authorization Code + PKCE instead.
10. **Never use the Resource Owner Password flow** — use Authorization Code instead.

### Client Security

11. **Keep client secrets on the server** — never in frontend code or mobile apps.
12. **Use client certificates** for machine-to-machine when possible (more secure than client_secret).
13. **Register specific redirect URIs** — avoid patterns like `https://yourapp.com/*`.

### Infrastructure

14. **Always use HTTPS** — tokens in transit must be encrypted.
15. **Implement token revocation** — especially on logout.
16. **Monitor for token abuse** — unusual patterns, multiple geographies, etc.
17. **Rotate client secrets periodically**.

---

## 25. COMMON ERRORS & TROUBLESHOOTING

| Error | Cause | Fix |
|-------|-------|-----|
| `invalid_client` | Wrong `client_id` or `client_secret` | Double-check credentials |
| `invalid_grant` | Code expired, already used, or wrong `redirect_uri` | Get a new code, verify redirect_uri matches exactly |
| `invalid_scope` | Requested scope not allowed for this client | Check allowed scopes in provider's console |
| `unauthorized_client` | Client not allowed to use this grant type | Enable the grant type in provider settings |
| `invalid_redirect_uri` | Redirect URI doesn't match registered URIs | Register the exact URI (including trailing slash) |
| `access_denied` | User denied consent | User clicked "Deny" — nothing you can do |
| `invalid_token` | Token expired, revoked, or malformed | Refresh the token or re-authenticate |
| `insufficient_scope` | Token doesn't have required permissions | Request additional scopes |
| CORS errors | Browser blocking cross-origin token request | Token exchange must happen server-side, not from browser |
| `interaction_required` | User action needed (consent, MFA, password change) | Redirect user to authorization endpoint again |

### Debugging Tips

1. **Decode JWT tokens** at [jwt.io](https://jwt.io) to inspect claims.
2. **Check the discovery document** for correct endpoint URLs.
3. **Compare `redirect_uri`** exactly (including protocol, trailing slash).
4. **Check token expiry** — `exp` claim in the JWT.
5. **Enable debug logging** in your app for OAuth-related requests.
6. **Use Postman's OAuth 2.0 authorization type** — it handles the flow automatically.

---

## 26. QUICK REFERENCE — ALL ENDPOINTS CHEAT SHEET

### Generic OAuth2/OIDC Endpoints

| Purpose | Endpoint | Method |
|---------|----------|--------|
| Discovery | `/.well-known/openid-configuration` | GET |
| Authorization | `/authorize` | GET (browser redirect) |
| Token | `/token` | POST |
| UserInfo | `/userinfo` | GET |
| JWKS (public keys) | `/certs` or `/jwks.json` | GET |
| Introspection | `/introspect` | POST |
| Revocation | `/revoke` | POST |
| Logout | `/logout` | POST or GET |

### Provider Comparison

```
┌──────────────┬──────────────┬────────────────┬──────────────┬──────────────┬──────────────┐
│              │   Google     │   Microsoft    │   GitHub     │   Keycloak   │   Okta       │
├──────────────┼──────────────┼────────────────┼──────────────┼──────────────┼──────────────┤
│ OIDC Support │ ✅ Yes       │ ✅ Yes         │ ❌ No        │ ✅ Yes       │ ✅ Yes       │
│ ID Token     │ ✅ Yes       │ ✅ Yes         │ ❌ No        │ ✅ Yes       │ ✅ Yes       │
│ Refresh Token│ ✅ Yes       │ ✅ Yes         │ ❌ No*       │ ✅ Yes       │ ✅ Yes       │
│ PKCE         │ ✅ Yes       │ ✅ Yes         │ ❌ No        │ ✅ Yes       │ ✅ Yes       │
│ Client Creds │ ✅ Yes       │ ✅ Yes         │ ❌ No        │ ✅ Yes       │ ✅ Yes       │
│ Device Flow  │ ✅ Yes       │ ✅ Yes         │ ✅ Yes       │ ✅ Yes       │ ✅ Yes       │
│ Introspection│ ❌ No        │ ❌ No          │ ❌ No        │ ✅ Yes       │ ✅ Yes       │
│ Token Format │ JWT          │ JWT            │ Opaque       │ JWT          │ JWT          │
│ Self-Hosted  │ ❌ No        │ ❌ No          │ ❌ No        │ ✅ Yes       │ ❌ No        │
│ Token Expiry │ 1 hour       │ 1 hour         │ No expiry*   │ 5 min (def)  │ 1 hour (def) │
│ Free         │ ✅ Yes       │ ✅ Basic       │ ✅ Yes       │ ✅ Yes       │ ✅ Dev tier  │
└──────────────┴──────────────┴────────────────┴──────────────┴──────────────┴──────────────┘
* GitHub Apps support refresh tokens (8hr expiry). Regular OAuth apps don't.
```

### Token Request Templates (Copy-Paste for Postman)

**Authorization Code Exchange:**
```
POST {token_endpoint}
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code={CODE}
&redirect_uri={REDIRECT_URI}
&client_id={CLIENT_ID}
&client_secret={CLIENT_SECRET}
```

**Refresh Token:**
```
POST {token_endpoint}
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token={REFRESH_TOKEN}
&client_id={CLIENT_ID}
&client_secret={CLIENT_SECRET}
```

**Client Credentials:**
```
POST {token_endpoint}
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={CLIENT_ID}
&client_secret={CLIENT_SECRET}
&scope={SCOPES}
```

**Resource Owner Password (Testing Only):**
```
POST {token_endpoint}
Content-Type: application/x-www-form-urlencoded

grant_type=password
&username={USERNAME}
&password={PASSWORD}
&client_id={CLIENT_ID}
&client_secret={CLIENT_SECRET}
&scope={SCOPES}
```

---

## 27. WHY AUTHORIZATION CODE INSTEAD OF DIRECT TOKEN?

One of the most important security design decisions in OAuth 2.0.

### The Core Idea

The **browser is not trusted**, but the **backend server is trusted**. So the access token should only go to the backend server, never appear in the browser URL.

```
UNSAFE (token in browser):
  User Browser → Authorization Server → Access Token → Browser → App
  Token exposed in URL ❌

SAFE (authorization code first):
  User Browser → Authorization Server → Authorization Code → Browser → Backend Server
  Backend Server → Authorization Server → Access Token
  Token never in browser ✅
```

### What Would Happen If Token Was Returned Directly?

If the token came back in the URL like `https://app.com/callback?access_token=ABC123`:

| Attack Vector | How Token Gets Leaked |
|--------------|----------------------|
| **Browser History** | URL saved in history: `callback?access_token=ABC123` — anyone with device access can steal it |
| **Server Logs** | Web servers log full URLs — token stored in log files |
| **Referrer Header** | If page loads external resources (images, scripts), browser sends `Referer: callback?access_token=ABC123` to third-party servers |
| **JavaScript Theft** | Malicious JS can read `window.location.href` and send token to attacker |
| **Shared Computers** | Token visible in URL bar on shared/public machines |

### Why Authorization Code Is Safe

The authorization code has these properties:

| Property | Meaning |
|----------|--------|
| **Short-lived** | Expires in seconds (usually 30-60 sec) |
| **One-time use** | Cannot be reused — server invalidates it after first exchange |
| **Useless alone** | Requires `client_secret` to exchange for token — attacker cannot use it without the secret |

### The Exchange Happens Server-Side

```
Browser only sees:
  /callback?code=XYZ123     ← useless without client_secret

Backend server calls:
  POST /token
    code=XYZ123
    client_id=...
    client_secret=...        ← only server knows this

Response (server-to-server, not through browser):
  access_token = real_token
  refresh_token = ...
```

The access token **never touches the browser**.

### Visual Comparison

```
Without Authorization Code (Implicit Flow — DEPRECATED):

  User ──→ Auth Server ──→ Token in URL ──→ Browser ──→ App
                              │
                              └── Token exposed in URL bar
                              └── Token in browser history
                              └── Token leaked via Referrer
                              └── Token stolen by malicious JS

With Authorization Code (Secure):

  User ──→ Auth Server ──→ Code in URL ──→ Browser ──→ Backend
                                                         │
                                                         └── Backend exchanges code + secret
                                                         └── Token returned server-to-server
                                                         └── Token NEVER in browser
```

---

## 28. REAL-WORLD OAUTH LOGIN — HOW APPS HANDLE USERS

This section explains what happens **inside the application** after OAuth login succeeds — something most OAuth tutorials skip.

### Key Insight

OAuth providers (Google, Microsoft) only **verify identity**. Your app manages the user account internally.

```
Google → "This user is ritesh@gmail.com with ID 109239487234987"
Your App → creates local user record, creates session/JWT

After login, Google is NOT involved in every request.
Your app handles authentication using its own session or JWT.
```

### Database Design for OAuth Users

Production systems use **two tables** to support multiple login providers.

**Users table** (your app's internal user):

| id | email | name |
|----|-------|------|
| 1 | ritesh@gmail.com | Ritesh Kumar Singh |

**OAuth Accounts table** (links providers to internal user):

| id | user_id | provider | provider_user_id |
|----|---------|----------|------------------|
| 1 | 1 | google | 109239487234987 |
| 2 | 1 | microsoft | a22b024a-e695-40e4-... |

This allows **one user to login with multiple providers** — all map to the same internal user.

### Provider User ID (`sub`) — Important Properties

| Property | Meaning |
|----------|--------|
| **Unique** | Each user has a unique ID per provider |
| **Stable** | Does not change across logins |
| **Permanent** | Remains same even if email changes |
| **Independent of token** | Different tokens → same `sub` value |

> **Always store `provider + provider_user_id`** as the identity key — NOT email alone. Users can change email, have multiple emails, or email may be unverified.

### First-Time Signup Flow (Login with Google for the First Time)

```
User clicks "Login with Google"
        ↓
Redirect to Google → User logs in → Consent screen
        ↓
Google returns authorization code
        ↓
Backend exchanges code → gets access_token + id_token
        ↓
Backend calls /userinfo (or decodes id_token)
        ↓
Gets: { sub: "109239487234987", email: "ritesh@gmail.com", name: "Ritesh" }
        ↓
Backend checks: SELECT * FROM oauth_accounts
                WHERE provider='google' AND provider_user_id='109239487234987'
        ↓
Not found → First time user
        ↓
INSERT INTO users (email, name) → user_id = 1
INSERT INTO oauth_accounts (user_id, provider, provider_user_id)
        ↓
Create app session (cookie) or app JWT
        ↓
User is logged in ✅
```

### Returning User Login Flow (Login with Google Again)

```
User clicks "Login with Google"
        ↓
Redirect to Google
        ↓
Google session still active → skips password screen (feels instant)
        ↓
Google returns authorization code
        ↓
Backend exchanges code → gets tokens → calls /userinfo
        ↓
Gets: { sub: "109239487234987", ... }
        ↓
Backend checks: SELECT * FROM oauth_accounts
                WHERE provider='google' AND provider_user_id='109239487234987'
        ↓
Found → user_id = 1 → existing user
        ↓
Create new session → User logged in ✅
No new signup — just login.
```

### Multiple Provider Account Linking

**Problem**: User signs up with Facebook, then later tries to login with Google. How does the app know it's the same person?

The app uses a **decision flow**:

```
OAuth Login Complete → provider returns (provider, provider_user_id, email)
        ↓
Step 1: Check provider mapping
        SELECT user_id FROM oauth_accounts
        WHERE provider=? AND provider_user_id=?
        ↓
        Found? → Login ✅
        ↓
Step 2: Not found → Check email
        SELECT id FROM users WHERE email=?
        ↓
        Email exists? → Link new provider to existing user
        INSERT INTO oauth_accounts (user_id, provider, provider_user_id)
        → Login ✅
        ↓
Step 3: Email also not found → Create new user
        INSERT INTO users → INSERT INTO oauth_accounts
        → Signup + Login ✅
```

**Three strategies used in real systems:**

| Strategy | How It Works | Used By |
|----------|-------------|--------|
| **Auto-link by email** | If same email → automatically link accounts | Most apps (GitHub, Notion) |
| **Ask user to confirm** | Shows: "Account with this email exists. Link Google to it?" | Medium-security apps |
| **Manual linking only** | User must login with first provider, then go to Settings → Link Google | High-security apps (banks) |

> **Edge case**: If Facebook returns `ritesh@facebookmail.com` and Google returns `ritesh@gmail.com`, the app **cannot auto-detect** it's the same person. Two separate users will be created unless manually linked.

### What Happens After OAuth Login (Session vs JWT)

After OAuth login, the app creates its **own** authentication mechanism:

```
Traditional Web App (coupled UI + backend):
  OAuth login → create local user → create SESSION COOKIE
  All future requests: Cookie: SESSION_ID=abc123
  Server checks session → identifies user
  Google NOT involved after login.

SPA + API Architecture (Angular + Spring Boot):
  OAuth login → create local user → issue APP JWT
  All future requests: Authorization: Bearer APP_JWT
  API validates JWT → identifies user
  Google NOT involved after login.
```

> The app does NOT use the Google/Microsoft access token for every request. It creates its own session/JWT after the first OAuth login.

### Google OAuth Playground (Testing Tool)

Google provides a tool to test OAuth flows without writing code:

```
URL: https://developers.google.com/oauthplayground

Steps:
  1. Select scope: openid, email, profile
  2. Click "Authorize APIs"
  3. Login with Google account
  4. Exchange code for token
  5. Call: GET https://openidconnect.googleapis.com/v1/userinfo
  6. See your sub, email, name, picture
```

Useful for testing token generation and seeing what user info Google returns.

### UserInfo API vs Decoding Token — When to Use Which?

| Approach | When to Use |
|----------|------------|
| **Decode JWT token** | When access/ID token is JWT format and contains user claims |
| **Call /userinfo API** | When token is opaque (cannot decode) OR when you need extra profile data not in token |

Many providers put minimal info in tokens but return full profile via /userinfo:

```
Token may contain:        /userinfo returns:
  { "sub": "123" }         { "sub": "123",
                              "email": "user@gmail.com",
                              "picture": "https://...",
                              "locale": "en" }
```

> In practice, most apps decode the ID token for user info during login. The /userinfo endpoint is a fallback for opaque tokens or extra data.

---

## 29. ID TOKEN VS ACCESS TOKEN — DEEP DIVE

This is the most common confusion in OAuth/OIDC. Understanding who uses which token and why changes everything.

### The One Rule That Fixes All Confusion

```
ID Token    → proves WHO the user is     → consumed by the CLIENT APP
Access Token → grants access to an API   → consumed by the RESOURCE SERVER (API)
```

### Who Consumes Each Token?

```
┌─────────────┐     ID Token      ┌────────────────┐
│   Auth      │ ─────────────────→│  Client App    │  Client reads ID token
│   Server    │                   │  (Angular/     │  to know WHO logged in
│  (Google)   │                   │   React)       │
│             │     Access Token  │                │
│             │ ─────────────────→│                │
└─────────────┘                   └───────┬────────┘
                                          │
                                          │ Sends ACCESS TOKEN only
                                          │ (NOT ID token)
                                          ▼
                                  ┌────────────────┐
                                  │  Backend API   │  API validates access token
                                  │  (Spring Boot) │  to AUTHORIZE the request
                                  └────────────────┘
```

### Detailed Comparison

| Aspect | ID Token | Access Token |
|--------|----------|-------------|
| **Purpose** | Authentication (who is this user?) | Authorization (what can this user do?) |
| **Who consumes it** | Client app (frontend) | Resource server (API) |
| **Contains user identity** | Always (name, email, sub) | Sometimes |
| **Format** | Always JWT | JWT or opaque |
| **Used to call APIs** | **NO** | **YES** |
| **`aud` (audience)** | = Client app's client_id | = Resource API identifier |
| **Sent to backend API** | **NO** (wrong design) | **YES** |

### Common Mistake: Sending ID Token to APIs

Many developers do this (incorrect):

```
❌ WRONG:
  GET /api/orders
  Authorization: Bearer ID_TOKEN    ← ID token is NOT for APIs

✅ CORRECT:
  GET /api/orders
  Authorization: Bearer ACCESS_TOKEN  ← access token IS for APIs
```

ID tokens are for the client app to display user info ("Welcome, Ritesh!"). Access tokens are for API authorization.

### Audience (`aud`) — The Key Concept

The `aud` claim tells **who the token is intended for**. Think of tokens like entry tickets:

```
Ticket for Movie A  → cannot enter Movie B
Token for Graph API → cannot access your backend API
Token for your API  → cannot access Graph API
```

| Token | `aud` Value | Meant For |
|-------|------------|----------|
| ID Token | Your app's client_id | Your client app |
| Access Token (Graph) | `00000003-0000-0000-c000-000000000000` | Microsoft Graph API |
| Access Token (your API) | `api://your-api-client-id` | Your backend API |

> **One access token = one resource API**. You cannot have a single token that works for both Graph API and your backend API. You need separate tokens.

### Scope Determines the Access Token Audience

The audience of the access token depends on **which scopes you request**:

```
Scope requested                          → Token audience (aud)
─────────────────────────────────────────────────────────────
User.Read, Mail.Send                     → Microsoft Graph API
api://backend-id/access_as_user          → Your backend API
https://outlook.office.com/.default      → Outlook API
```

Common scopes vs resource-specific scopes:

```
Common (identity only, don't affect aud):
  openid, profile, email, offline_access

Resource-specific (determine the aud):
  User.Read                              → Graph API
  api://client-id/access_as_user         → Your API
  Sites.Read.All                         → SharePoint
```

### Calling Multiple APIs? Get Multiple Tokens

If your app needs to call both Microsoft Graph and your backend API:

```
Angular SPA
   │
   ├── Token 1: aud = Graph API      → call graph.microsoft.com
   │   (requested with scope: User.Read)
   │
   └── Token 2: aud = Your API       → call your-backend.com
       (requested with scope: api://backend-id/access_as_user)
```

### Real-World Production Pattern

```
┌─────────────┐
│   Angular   │
│   SPA       │
│             │
│  Stores:    │
│  - ID token │ → decode to show "Welcome Ritesh"
│  - Graph    │ → use when calling Graph APIs
│    token    │
│  - API      │ → use when calling backend
│    token    │
└──────┬──────┘
       │
       │  Authorization: Bearer API_TOKEN
       ▼
┌─────────────┐
│  Spring Boot│
│  Backend    │
│             │
│  Validates: │
│  - signature│
│  - issuer   │
│  - audience │ ← must match MY API identifier
│  - expiry   │
│  - scopes   │
└─────────────┘
```

---

## 30. SPA + BACKEND API ARCHITECTURE (DECOUPLED APPS)

In modern apps, the frontend (Angular/React) and backend (Spring Boot/Node) are **separate applications**. This changes how OAuth works.

### Why Sessions Don't Work for SPAs

```
Traditional (coupled) app:
  Browser → Backend (same server) → Session cookie works fine

Decoupled app:
  Angular (localhost:4200) → Spring Boot (localhost:8080) → Different origins

Problems with sessions:
  ❌ CORS issues (different origins)
  ❌ Cannot scale horizontally (session sticky to one server)
  ❌ Microservices cannot share sessions easily
  ❌ Mobile apps cannot use session cookies
```

**Solution**: Use **tokens** instead of sessions.

### Complete SPA + API OAuth Flow

```
┌──────────────┐                     ┌──────────────────┐
│   Angular    │                     │  Auth Server     │
│   SPA        │                     │  (Google/Azure)  │
│              │   1. Redirect       │                  │
│              │ ──────────────────→ │                  │
│              │                     │  2. User login   │
│              │                     │  3. Consent      │
│              │   4. Auth code      │                  │
│              │ ←────────────────── │                  │
│              │                     │                  │
│              │   5. Exchange code  │                  │
│              │      + PKCE         │                  │
│              │ ──────────────────→ │                  │
│              │                     │                  │
│              │   6. Returns:       │                  │
│              │      access_token   │                  │
│              │      id_token       │                  │
│              │ ←────────────────── │                  │
└──────┬───────┘                     └──────────────────┘
       │
       │  7. API call with access token
       │  Authorization: Bearer ACCESS_TOKEN
       ▼
┌──────────────┐
│  Spring Boot │
│  Backend API │
│              │
│  8. Validate │
│     JWT      │
│  9. Return   │
│     data     │
└──────────────┘
```

> SPAs use **Authorization Code + PKCE** because they cannot safely store a `client_secret`.

### How the Backend API Validates the Token

The backend API does NOT participate in login. It only:

1. Receives access token in `Authorization: Bearer` header
2. Validates the JWT:

```
Validation steps:
  1. Verify SIGNATURE    → fetch public key from JWKS endpoint
  2. Verify ISSUER (iss) → must be the expected auth server
  3. Verify AUDIENCE (aud) → must be MY API identifier
  4. Verify EXPIRY (exp)  → token not expired
  5. Read SCOPES (scp)    → check permissions

If all pass → request authorized ✅
If any fail → return 401 Unauthorized ❌
```

### Token Storage in SPA

| Storage | Security | Recommendation |
|---------|---------|---------------|
| **In-memory** (JS variable) | Most secure — lost on page refresh | Best for high-security apps |
| **Session storage** | Cleared on tab close | Good balance |
| **Local storage** | Persists across tabs/sessions | Risky — accessible to XSS attacks |
| **HTTP-only cookie** | Not accessible to JS | Best when backend sets the cookie |

> **Never store tokens in local storage** in production. Use in-memory or HTTP-only cookies.

### How Frontend Extracts User Info from ID Token

The ID token is a JWT. Frontend decodes it to show user info:

```javascript
// Option 1: Manual decode (no library needed)
const payload = JSON.parse(atob(idToken.split('.')[1]));
console.log(payload.name);     // "Ritesh Kumar Singh"
console.log(payload.email);    // "ritesh@gmail.com"

// Option 2: Using jwt-decode library
import jwtDecode from 'jwt-decode';
const user = jwtDecode(idToken);
console.log(user.name);
console.log(user.email);
```

Typical frontend uses:

- Show `Welcome, Ritesh!`
- Display user email/avatar
- Store user info in app state (Redux/NgRx/Zustand)
- Determine UI permissions (admin vs regular user)

---

## 31. MICROSOFT-SPECIFIC — OID VS SUB, CUSTOM API SCOPES, OBO FLOW

### Microsoft `oid` vs `sub` — Which Is the User ID?

Microsoft tokens contain **both** `oid` and `sub`, which confuses many developers.

| Claim | Full Name | Meaning | Scope |
|-------|-----------|---------|-------|
| `sub` | Subject | OIDC standard user identifier | **Per-app** (different for each app registration) |
| `oid` | Object ID | Azure AD directory user object ID | **Per-tenant** (same across all apps in the tenant) |

**Example**: Same user logs into two apps in the same tenant:

```
App A:  sub = "A123XYZ"    oid = "a22b024a-e695-40e4-..."
App B:  sub = "B456XYZ"    oid = "a22b024a-e695-40e4-..."
                            └── same oid
```

**Which should you store?**

| Scenario | Use |
|----------|-----|
| OIDC-standard, multi-provider app | `sub` (OIDC standard) |
| Microsoft-only enterprise app | `oid` (recommended by Microsoft) |
| Calling Microsoft Graph API | `oid` (Graph API `id` = `oid`) |

> For most Microsoft enterprise apps, use `oid` as the provider user ID. For Google, always use `sub`.

### Google vs Microsoft Identity Model

```
Google:
  Mostly consumer identity
  Global accounts: user@gmail.com
  No tenant needed for basic OAuth
  (Google Workspace has tenants, but optional)

Microsoft:
  Mostly organization identity
  Tenant-based: company.onmicrosoft.com → employees
  Every user belongs to a tenant
  Tenant ID appears in token issuer URL
```

| Aspect | Google | Microsoft |
|--------|--------|-----------|
| Identity model | Consumer (global) | Organization (tenant-based) |
| User ID claim | `sub` | `oid` (preferred) |
| Login URL | `accounts.google.com` | `login.microsoftonline.com/{tenant}` |
| Tenant required? | No | Yes |
| Issuer format | `https://accounts.google.com` | `https://login.microsoftonline.com/{tenant}/v2.0` |

### Custom API Scope Setup in Microsoft Entra ID

To get an access token **for your own backend API** (not Graph API), you must register your API as a resource:

```
Step 1: Go to Azure Portal → App registrations → Select your API app

Step 2: Click "Expose an API"
        → Set Application ID URI (e.g., api://your-client-id)

Step 3: Click "Add a scope"
        → Scope name: access_as_user
        → Admin consent display name: Access my API
        → State: Enabled
        → Save

Now the scope exists:
  api://your-client-id/access_as_user
```

**Request token for your API:**

```
scope=openid profile api://your-client-id/access_as_user
```

**Resulting access token:**

```json
{
  "aud": "api://your-client-id",
  "iss": "https://login.microsoftonline.com/{tenant}/v2.0",
  "scp": "access_as_user",
  "sub": "user-id",
  "exp": 1710000000
}
```

Now `aud` = your API → your backend will accept this token.

> **Without this setup**, requesting `api://client-id/access_as_user` gives error `AADSTS65005: scope doesn't exist`.

### Microsoft Graph API Audience

When you request Graph scopes (like `User.Read`, `Mail.Send`), the token always has:

```
aud = 00000003-0000-0000-c000-000000000000
```

This is Microsoft Graph's fixed resource ID. Your backend API must **reject** tokens with this audience — they are meant for Graph, not for you.

### On-Behalf-Of (OBO) Flow

Common enterprise pattern: Frontend calls your backend, and your backend needs to call Microsoft Graph on behalf of the user.

```
Angular SPA
     │
     │  Token (aud = your API)
     ▼
Your Backend API
     │
     │  "I need to call Graph on behalf of this user"
     │
     │  POST /token
     │    grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
     │    assertion=USER_ACCESS_TOKEN
     │    scope=User.Read
     │    client_id=YOUR_API_CLIENT_ID
     │    client_secret=YOUR_API_SECRET
     │    requested_token_use=on_behalf_of
     ▼
Microsoft Entra
     │
     │  Returns new token (aud = Graph API)
     ▼
Your Backend → Calls Graph API with new token
```

**OBO request:**

```
POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
&assertion=USER_ACCESS_TOKEN_FOR_YOUR_API
&client_id=YOUR_API_CLIENT_ID
&client_secret=YOUR_API_CLIENT_SECRET
&scope=https://graph.microsoft.com/User.Read
&requested_token_use=on_behalf_of
```

**Response:**

```json
{
  "access_token": "eyJ0eXAiOi...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "User.Read"
}
```

Now your backend can call Graph API using this new token.

**When to use OBO:**

| Scenario | Flow |
|----------|------|
| Frontend calls Graph directly | No OBO needed — frontend uses Graph token |
| Frontend calls your API, and your API calls Graph | OBO flow |
| Backend cron/service calls Graph (no user) | Client Credentials flow |

### Microsoft Token Validation with Java (Nimbus JOSE JWT)

For backend APIs that need to validate Microsoft JWT tokens:

**Maven dependency:**

```xml
<dependency>
  <groupId>com.nimbusds</groupId>
  <artifactId>nimbus-jose-jwt</artifactId>
  <version>9.37</version>
</dependency>
```

**Validation code:**

```java
import com.nimbusds.jose.crypto.RSASSAVerifier;
import com.nimbusds.jose.jwk.JWK;
import com.nimbusds.jose.jwk.JWKSet;
import com.nimbusds.jwt.JWTClaimsSet;
import com.nimbusds.jwt.SignedJWT;
import java.net.URL;
import java.util.Date;

public class MicrosoftTokenValidator {

    private static final String JWKS_URL =
        "https://login.microsoftonline.com/common/discovery/v2.0/keys";

    // Step 1: Parse the JWT
    public static SignedJWT parseToken(String token) throws Exception {
        return SignedJWT.parse(token);
    }

    // Step 2: Fetch matching public key from Microsoft JWKS
    public static JWK fetchSigningKey(SignedJWT jwt) throws Exception {
        String kid = jwt.getHeader().getKeyID();
        JWKSet jwkSet = JWKSet.load(new URL(JWKS_URL));
        for (JWK key : jwkSet.getKeys()) {
            if (key.getKeyID().equals(kid)) return key;
        }
        throw new RuntimeException("Matching key not found for kid: " + kid);
    }

    // Step 3: Verify signature
    public static boolean verifySignature(SignedJWT jwt, JWK jwk) throws Exception {
        return jwt.verify(new RSASSAVerifier(jwk.toRSAKey()));
    }

    // Step 4: Validate claims
    public static void validateClaims(SignedJWT jwt, String expectedAudience) throws Exception {
        JWTClaimsSet claims = jwt.getJWTClaimsSet();

        // Check expiry
        if (claims.getExpirationTime() == null || claims.getExpirationTime().before(new Date()))
            throw new RuntimeException("Token expired");

        // Check audience
        if (!claims.getAudience().contains(expectedAudience))
            throw new RuntimeException("Invalid audience");

        // Check issuer
        if (!claims.getIssuer().contains("login.microsoftonline.com"))
            throw new RuntimeException("Invalid issuer");
    }
}
```

**Python equivalent (PyJWT):**

```python
import jwt
import requests
from jwt.algorithms import RSAAlgorithm

def validate_microsoft_token(token, expected_audience):
    # Get header to find key ID
    header = jwt.get_unverified_header(token)

    # Fetch Microsoft public keys
    jwks_url = "https://login.microsoftonline.com/common/discovery/v2.0/keys"
    jwks = requests.get(jwks_url).json()

    # Find matching key
    key = None
    for jwk in jwks["keys"]:
        if jwk["kid"] == header["kid"]:
            key = RSAAlgorithm.from_jwk(jwk)
            break

    if not key:
        raise Exception("Matching key not found")

    # Verify and decode
    decoded = jwt.decode(
        token,
        key=key,
        algorithms=["RS256"],
        audience=expected_audience
    )
    return decoded
```

> In Spring Boot production apps, all this is done automatically by `spring-boot-starter-oauth2-resource-server` — you just configure the issuer URI.

### Microsoft Refresh Token Expiry

| Client Type | Refresh Token Expiry |
|-------------|---------------------|
| Web app (confidential client) | **90 days** (default) |
| SPA (public client) | **24 hours** |
| Mobile/Desktop app | **90 days** |

Refresh tokens can be revoked by admins or when the user changes password.

---
