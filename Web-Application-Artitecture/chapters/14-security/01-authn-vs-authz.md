# Authentication vs Authorization — Who Are You & What Can You Do?

> **What you'll learn**: The fundamental difference between proving your identity (authentication) and proving your permissions (authorization) — two concepts that are the bedrock of every secure web application.

---

## Real-Life Analogy

Imagine you're going to a **concert**.

1. **At the gate**, a security guard checks your **ticket and ID**. They verify you are who you claim to be. This is **authentication** — "Who are you?"

2. **Once inside**, you try to go backstage. Another guard checks your ticket type — do you have a **VIP pass** or a **general admission** ticket? This is **authorization** — "What are you allowed to do?"

```
┌─────────────────────────────────────────────────────┐
│                    CONCERT VENUE                      │
│                                                       │
│  ┌──────────┐        ┌──────────────┐               │
│  │  GATE    │        │  BACKSTAGE   │               │
│  │          │        │              │               │
│  │ "Show me │        │ "Do you have │               │
│  │  your ID"│        │  VIP access?"│               │
│  │          │        │              │               │
│  │ (AuthN)  │        │   (AuthZ)    │               │
│  └──────────┘        └──────────────┘               │
└─────────────────────────────────────────────────────┘
```

- **Authentication (AuthN)** = "Prove you are who you say you are"
- **Authorization (AuthZ)** = "Prove you have permission to do this thing"

They always happen in this order: first AuthN, then AuthZ. You can't check someone's permissions if you don't know who they are!

---

## Core Concept Explained Step-by-Step

### Step 1: Authentication — Proving Your Identity

Authentication answers: **"Who is this person?"**

Common ways to authenticate:

| Method | How It Works | Example |
|--------|-------------|---------|
| **Username + Password** | You prove identity with a secret only you know | Logging into Gmail |
| **Multi-Factor (MFA)** | Something you know + something you have/are | Password + OTP on phone |
| **Biometrics** | Physical characteristics | Fingerprint, Face ID |
| **API Keys** | A unique string that identifies a client | `X-API-Key: abc123` |
| **Certificates** | A digital certificate proving identity | mTLS between services |
| **SSO (Single Sign-On)** | Login once, access many apps | Login with Google |

### Step 2: Authorization — Checking Permissions

Authorization answers: **"What is this person allowed to do?"**

Common authorization models:

| Model | How It Works | Example |
|-------|-------------|---------|
| **Role-Based (RBAC)** | Users have roles, roles have permissions | Admin, Editor, Viewer |
| **Attribute-Based (ABAC)** | Permissions based on attributes | "Users in India can access Indian data" |
| **Policy-Based (PBAC)** | Explicit policy rules | AWS IAM policies |
| **Access Control Lists (ACL)** | Per-resource permission lists | File permissions in Linux |
| **Ownership-Based** | You can only access what you own | "Edit YOUR posts only" |

### Step 3: How They Work Together

```
┌──────────┐       ┌───────────────┐       ┌───────────────┐       ┌──────────┐
│  Client  │──────▶│ Authentication│──────▶│ Authorization │──────▶│ Resource │
│ (User)   │       │   Service     │       │   Service     │       │ (Data)   │
└──────────┘       └───────────────┘       └───────────────┘       └──────────┘
                         │                        │
                    "Who are you?"          "Can you do this?"
                         │                        │
                    Checks:                  Checks:
                    - Password               - Role
                    - Token                  - Policy
                    - Certificate            - Ownership
```

---

## How It Works Internally

### Authentication Flow (Detailed)

```
┌────────┐                    ┌────────────┐                  ┌──────────┐
│ Client │                    │ Auth Server│                  │ Database │
└───┬────┘                    └─────┬──────┘                  └────┬─────┘
    │                               │                              │
    │  1. POST /login               │                              │
    │  {username, password}         │                              │
    │──────────────────────────────▶│                              │
    │                               │  2. Query user by username   │
    │                               │─────────────────────────────▶│
    │                               │                              │
    │                               │  3. Return user record       │
    │                               │◀─────────────────────────────│
    │                               │                              │
    │                               │  4. Hash input password      │
    │                               │     Compare with stored hash │
    │                               │                              │
    │  5. Return token/session      │                              │
    │◀──────────────────────────────│                              │
    │                               │                              │
```

**Key internal steps:**

1. Client sends credentials (username + password)
2. Server looks up the user in the database
3. Server **hashes** the input password using the same algorithm (bcrypt, argon2)
4. Server **compares** the hash with the stored hash (NEVER stores plain passwords!)
5. If match → generate a token (JWT) or create a session
6. Return the token to the client

### Authorization Flow (Detailed)

```
┌────────┐                    ┌────────────┐                  ┌──────────────┐
│ Client │                    │ API Server │                  │ Policy Engine│
└───┬────┘                    └─────┬──────┘                  └──────┬───────┘
    │                               │                                │
    │  1. GET /admin/users          │                                │
    │  Header: Bearer <token>       │                                │
    │──────────────────────────────▶│                                │
    │                               │                                │
    │                               │  2. Decode token → user_id,   │
    │                               │     roles: ["editor"]         │
    │                               │                                │
    │                               │  3. Check: Can "editor" role  │
    │                               │     access GET /admin/users?  │
    │                               │───────────────────────────────▶│
    │                               │                                │
    │                               │  4. DENIED (needs "admin")    │
    │                               │◀───────────────────────────────│
    │                               │                                │
    │  5. 403 Forbidden             │                                │
    │◀──────────────────────────────│                                │
```

### Password Storage — How It Actually Works

**NEVER store passwords in plain text.** Here's the correct approach:

```
Registration:
  plain_password = "MyP@ss123"
  salt = random_bytes(16)                    # Unique per user
  hashed = bcrypt(plain_password + salt)     # One-way hash
  store(username, hashed, salt)              # Save to DB

Login:
  input_password = "MyP@ss123"
  stored_hash = get_from_db(username)
  computed_hash = bcrypt(input_password + stored_salt)
  
  if computed_hash == stored_hash:
      AUTHENTICATED ✓
  else:
      REJECTED ✗
```

---

## Code Examples

### Python — Authentication + Authorization

```python
# authentication_and_authorization.py
import hashlib
import secrets
from functools import wraps

# --- AUTHENTICATION ---

# Simulated user database
users_db = {
    "alice": {
        "password_hash": None,  # Will be set below
        "salt": None,
        "roles": ["admin", "editor"]
    },
    "bob": {
        "password_hash": None,
        "salt": None,
        "roles": ["viewer"]
    }
}

def hash_password(password: str, salt: bytes) -> str:
    """Hash a password with salt using SHA-256 (use bcrypt in production!)"""
    return hashlib.sha256(salt + password.encode()).hexdigest()

def register_user(username: str, password: str, roles: list):
    """Register a new user with hashed password"""
    salt = secrets.token_bytes(16)  # Random 16-byte salt
    password_hash = hash_password(password, salt)
    users_db[username] = {
        "password_hash": password_hash,
        "salt": salt,
        "roles": roles
    }

def authenticate(username: str, password: str) -> dict | None:
    """Verify credentials. Returns user dict or None."""
    user = users_db.get(username)
    if not user:
        return None  # User not found
    
    computed_hash = hash_password(password, user["salt"])
    if computed_hash == user["password_hash"]:
        return {"username": username, "roles": user["roles"]}
    return None  # Wrong password

# --- AUTHORIZATION ---

def authorize(required_role: str):
    """Decorator that checks if user has the required role"""
    def decorator(func):
        @wraps(func)
        def wrapper(user, *args, **kwargs):
            if required_role not in user.get("roles", []):
                raise PermissionError(
                    f"User '{user['username']}' lacks role '{required_role}'"
                )
            return func(user, *args, **kwargs)
        return wrapper
    return decorator

@authorize("admin")
def delete_user(current_user, target_username):
    """Only admins can delete users"""
    print(f"✓ {current_user['username']} deleted user '{target_username}'")

@authorize("viewer")
def view_dashboard(current_user):
    """Any authenticated user with 'viewer' role can view"""
    print(f"✓ {current_user['username']} is viewing the dashboard")

# --- USAGE ---
register_user("alice", "securePass123", ["admin", "editor"])
register_user("bob", "bobPass456", ["viewer"])

# Authenticate
alice = authenticate("alice", "securePass123")  # ✓ Returns user
fake = authenticate("alice", "wrongpass")       # ✗ Returns None

# Authorize
if alice:
    delete_user(alice, "bob")       # ✓ Alice is admin
    view_dashboard(alice)            # ✗ Alice doesn't have "viewer" role!
```

### Java — Authentication + Authorization

```java
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.*;

public class AuthExample {

    // Simulated user database
    static Map<String, UserRecord> usersDb = new HashMap<>();

    record UserRecord(String passwordHash, byte[] salt, List<String> roles) {}

    // --- AUTHENTICATION ---

    static String hashPassword(String password, byte[] salt) throws Exception {
        MessageDigest md = MessageDigest.getInstance("SHA-256");
        md.update(salt);
        byte[] hash = md.digest(password.getBytes());
        return Base64.getEncoder().encodeToString(hash);
    }

    static void registerUser(String username, String password, List<String> roles) 
            throws Exception {
        byte[] salt = new byte[16];
        new SecureRandom().nextBytes(salt);  // Random salt
        String hash = hashPassword(password, salt);
        usersDb.put(username, new UserRecord(hash, salt, roles));
    }

    static Map<String, Object> authenticate(String username, String password) 
            throws Exception {
        UserRecord user = usersDb.get(username);
        if (user == null) return null;  // User not found

        String computedHash = hashPassword(password, user.salt());
        if (computedHash.equals(user.passwordHash())) {
            return Map.of("username", username, "roles", user.roles());
        }
        return null;  // Wrong password
    }

    // --- AUTHORIZATION ---

    static boolean hasRole(Map<String, Object> user, String requiredRole) {
        List<String> roles = (List<String>) user.get("roles");
        return roles.contains(requiredRole);
    }

    static void deleteUser(Map<String, Object> currentUser, String target) {
        // Authorization check
        if (!hasRole(currentUser, "admin")) {
            System.out.println("✗ DENIED: " + currentUser.get("username") 
                + " lacks 'admin' role");
            return;
        }
        System.out.println("✓ " + currentUser.get("username") 
            + " deleted user '" + target + "'");
    }

    public static void main(String[] args) throws Exception {
        // Register users
        registerUser("alice", "securePass123", List.of("admin", "editor"));
        registerUser("bob", "bobPass456", List.of("viewer"));

        // Authenticate
        Map<String, Object> alice = authenticate("alice", "securePass123"); // ✓
        Map<String, Object> fail = authenticate("alice", "wrong");          // null

        // Authorize
        if (alice != null) {
            deleteUser(alice, "bob");  // ✓ Alice is admin
        }

        Map<String, Object> bob = authenticate("bob", "bobPass456");
        if (bob != null) {
            deleteUser(bob, "alice");  // ✗ Bob is not admin
        }
    }
}
```

---

## Infrastructure Examples

### RBAC in PostgreSQL

```sql
-- Create roles
CREATE ROLE app_admin;
CREATE ROLE app_editor;
CREATE ROLE app_viewer;

-- Grant permissions to roles
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_admin;
GRANT SELECT, INSERT, UPDATE ON articles TO app_editor;
GRANT SELECT ON articles TO app_viewer;

-- Assign roles to users
GRANT app_admin TO alice;
GRANT app_editor TO bob;
GRANT app_viewer TO charlie;

-- Row-Level Security (Authorization at the data level!)
ALTER TABLE articles ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_articles ON articles
    FOR ALL
    USING (author_id = current_user_id());  -- Users see only their own articles
```

### Nginx — Basic Authentication

```nginx
# /etc/nginx/conf.d/protected.conf
server {
    listen 443 ssl;
    server_name admin.example.com;

    # Authentication: HTTP Basic Auth
    location /admin {
        auth_basic "Admin Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
        
        proxy_pass http://backend:8080;
    }
    
    # Public endpoints (no auth needed)
    location /public {
        proxy_pass http://backend:8080;
    }
}
```

### AWS IAM Policy — Authorization Example

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::my-bucket/users/${aws:username}/*",
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": "192.168.1.0/24"
                }
            }
        }
    ]
}
```

This policy says: "Allow this user to read/write objects ONLY in their own folder, AND only from the office network."

---

## Real-World Example

### How Netflix Handles AuthN/AuthZ

```
┌──────────┐      ┌─────────────┐      ┌──────────────┐      ┌───────────┐
│  User    │      │   Zuul      │      │   Auth       │      │  Content  │
│  (App)   │─────▶│  (Gateway)  │─────▶│  Service     │─────▶│  Service  │
└──────────┘      └─────────────┘      └──────────────┘      └───────────┘
                        │                      │
                   Every request          Checks:
                   passes through         1. Is token valid? (AuthN)
                   the gateway            2. Is plan active?
                                          3. Is content available
                                             in user's region? (AuthZ)
                                          4. Parental controls?
```

- **Authentication**: When you log in, Netflix verifies your email + password, then issues a token stored on your device.
- **Authorization**: When you try to play "Breaking Bad," Netflix checks:
  - Is your subscription active?
  - Is this content available in your country?
  - Does your profile have age restrictions?
  - Have you exceeded device limits?

### How Google Uses AuthN/AuthZ

- **Authentication**: Google uses advanced risk-based authentication — if you log in from a new device in a new country, it asks for extra verification (MFA).
- **Authorization**: Google uses a system called **Zanzibar** — a planet-scale authorization system that handles billions of permission checks per second for Google Drive, YouTube, Cloud, etc.

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Storing passwords in plain text | One DB breach = all accounts compromised | Use bcrypt/argon2 with salt |
| Confusing 401 and 403 | 401 = not authenticated, 403 = not authorized | Use the correct HTTP status |
| Checking auth only on the frontend | Anyone can bypass frontend checks | ALWAYS check on the server |
| Hardcoding roles in code | Can't change permissions without deployment | Use a policy engine or database |
| Not implementing MFA | Passwords alone are easily compromised | Add TOTP, SMS, or WebAuthn |
| Using same key for all API clients | Can't revoke one without affecting others | Issue per-client API keys |

### The 401 vs 403 Confusion

```
401 Unauthorized = "I don't know who you are. Please log in."
                    (Should really be called "Unauthenticated")

403 Forbidden    = "I know who you are, but you can't do this."
                    (You ARE authenticated, but NOT authorized)
```

---

## When to Use / When NOT to Use

### Authentication Methods — Decision Guide

| Scenario | Best AuthN Method |
|----------|-------------------|
| Public-facing web app | Username/password + MFA |
| Internal microservices | mTLS (mutual TLS certificates) |
| Third-party API access | OAuth 2.0 tokens |
| Mobile app | OAuth 2.0 + biometrics |
| IoT devices | Client certificates |
| Developer APIs | API keys + rate limiting |

### Authorization Models — Decision Guide

| Scenario | Best AuthZ Model |
|----------|------------------|
| Simple app with few roles | RBAC (Role-Based) |
| Complex enterprise with dynamic rules | ABAC (Attribute-Based) |
| Cloud resources | Policy-Based (IAM) |
| File/document sharing | ACL (Access Control Lists) |
| Multi-tenant SaaS | RBAC + tenant isolation |
| Social media (my posts, my photos) | Ownership-based + RBAC |

---

## Key Takeaways

- **Authentication (AuthN)** = "Who are you?" — verifying identity
- **Authorization (AuthZ)** = "What can you do?" — verifying permissions
- AuthN always happens BEFORE AuthZ — you can't check permissions for an unknown user
- **Never store passwords in plain text** — always hash with bcrypt/argon2 + unique salt
- Use **401** for unauthenticated requests, **403** for unauthorized requests
- **Always enforce authorization on the server** — never trust the frontend alone
- Start with RBAC for most apps; move to ABAC/policy-based as complexity grows

---

## What's Next?

Next, we'll dive into **Sessions, Cookies & Tokens (JWT)** — the actual mechanisms used to maintain authentication state across multiple requests. How does the server "remember" that you already logged in? That's Chapter 14.2.
