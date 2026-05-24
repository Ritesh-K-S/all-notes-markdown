# Spring OAuth2 — Complete Notes (Beginner to Advanced)

---

## Table of Contents

1. [Introduction to OAuth2](#1-introduction-to-oauth2)
2. [OAuth2 Terminology](#2-oauth2-terminology)
3. [OAuth2 Grant Types (Flows)](#3-oauth2-grant-types-flows)
4. [OpenID Connect (OIDC)](#4-openid-connect-oidc)
5. [Spring Security OAuth2 Architecture](#5-spring-security-oauth2-architecture)
6. [Project Setup](#6-project-setup)
7. [OAuth2 Client — Social Login (Google, GitHub, etc.)](#7-oauth2-client--social-login)
8. [Custom OAuth2 User Service](#8-custom-oauth2-user-service)
9. [OAuth2 Client — Login Success & Failure Handlers](#9-oauth2-client--login-success--failure-handlers)
10. [OAuth2 Client — Accessing Protected Resources](#10-oauth2-client--accessing-protected-resources)
11. [OAuth2 Resource Server — JWT](#11-oauth2-resource-server--jwt)
12. [OAuth2 Resource Server — Opaque Tokens](#12-oauth2-resource-server--opaque-tokens)
13. [Custom JWT Decoder & Validators](#13-custom-jwt-decoder--validators)
14. [Roles & Authorities from JWT Claims](#14-roles--authorities-from-jwt-claims)
15. [Spring Authorization Server](#15-spring-authorization-server)
16. [Authorization Server — Client Registration](#16-authorization-server--client-registration)
17. [Authorization Server — Token Customization](#17-authorization-server--token-customization)
18. [Authorization Server — Consent Flow](#18-authorization-server--consent-flow)
19. [PKCE (Proof Key for Code Exchange)](#19-pkce-proof-key-for-code-exchange)
20. [Token Relay & Propagation](#20-token-relay--propagation)
21. [OAuth2 with Microservices (API Gateway Pattern)](#21-oauth2-with-microservices-api-gateway-pattern)
22. [Token Revocation & Introspection](#22-token-revocation--introspection)
23. [OAuth2 with React / Angular SPA](#23-oauth2-with-react--angular-spa)
24. [Security Best Practices for OAuth2](#24-security-best-practices-for-oauth2)
25. [Testing OAuth2](#25-testing-oauth2)
26. [Common Configuration Properties Reference](#26-common-configuration-properties-reference)
27. [Important Annotations Reference](#27-important-annotations-reference)
28. [Troubleshooting & Common Errors](#28-troubleshooting--common-errors)

---

## 1. INTRODUCTION TO OAUTH2

### What is OAuth2?
- **OAuth 2.0** = an **authorization framework** that enables third-party applications to obtain limited access to a user's resources **without exposing their credentials**.
- Defined in **RFC 6749** (2012).
- OAuth2 is about **authorization** (what you can access), NOT authentication (who you are).
- **OpenID Connect (OIDC)** adds authentication on top of OAuth2.

### Real-World Analogy
```
You want a valet to park your car:
  - You give them a VALET KEY (limited access — drives but can't open trunk)
  - NOT your MASTER KEY (full access)

OAuth2 is the valet key for web APIs.
```

### Why OAuth2?
| Problem | OAuth2 Solution |
|---------|----------------|
| Sharing passwords with third-party apps | Tokens instead of passwords |
| Revoking single app access | Revoke token, not password |
| Limiting permissions | Scopes define what app can do |
| Standard protocol | Interoperable across providers |
| Session management complexity | Token-based, stateless |

### OAuth2 vs OAuth 1.0
| Aspect | OAuth 1.0 | OAuth 2.0 |
|--------|----------|-----------|
| Complexity | High (signatures) | Lower (bearer tokens) |
| Transport security | Built-in signing | Relies on HTTPS |
| Token types | Single | Access + Refresh tokens |
| Mobile support | Poor | First-class |
| Flows | One | Multiple grant types |

---

## 2. OAUTH2 TERMINOLOGY

### Key Actors
```
┌───────────────┐          ┌───────────────────┐
│ Resource Owner │          │   Client           │
│ (User)         │          │   (Your App)       │
└───────┬───────┘          └────────┬──────────┘
        │                           │
    Grants                     Requests
    Access                      Access
        │                           │
        ▼                           ▼
┌───────────────────┐      ┌───────────────────┐
│ Authorization      │      │ Resource Server    │
│ Server             │      │ (API)              │
│ (Google, Keycloak) │      │ (Holds user data)  │
└───────────────────┘      └───────────────────┘
```

| Term | Description | Example |
|------|-------------|---------|
| **Resource Owner** | The user who owns the data | You (the end user) |
| **Client** | The application requesting access | Your Spring Boot app |
| **Authorization Server** | Issues tokens after authenticating user | Google, Keycloak, Auth0, Okta |
| **Resource Server** | Hosts protected resources (APIs) | Your REST API |
| **Access Token** | Short-lived token to access resources | JWT or opaque string |
| **Refresh Token** | Long-lived token to get new access tokens | UUID stored server-side |
| **Scope** | Permission that the client requests | `read`, `write`, `profile`, `email` |
| **Redirect URI** | Where the auth server sends the user back | `https://myapp.com/callback` |
| **Client ID** | Public identifier for the client app | `abc123` |
| **Client Secret** | Confidential key for the client app | `secret456` (server-side only) |
| **Grant Type** | Method of obtaining a token | `authorization_code`, `client_credentials` |
| **Consent** | User approval of requested permissions | "Allow MyApp to access your email?" |

### Token Types
| Token | Lifetime | Purpose | Format |
|-------|---------|---------|--------|
| **Access Token** | Minutes-Hours | Access APIs | JWT or opaque |
| **Refresh Token** | Days-Weeks | Get new access tokens | Opaque (usually) |
| **ID Token** (OIDC) | Minutes | User identity info | JWT (always) |
| **Authorization Code** | Seconds | Exchange for tokens | Short random string |

---

## 3. OAUTH2 GRANT TYPES (FLOWS)

### Overview
| Grant Type | Use Case | Security |
|-----------|---------|---------|
| **Authorization Code** | Server-side web apps | Most secure |
| **Authorization Code + PKCE** | SPAs, mobile apps | Secure (no client secret) |
| **Client Credentials** | Machine-to-machine | No user involved |
| **Device Code** | Smart TVs, CLI tools | Limited input devices |
| ~~Implicit~~ | ~~SPAs~~ | **Deprecated** (use PKCE) |
| ~~Password~~ | ~~Trusted first-party apps~~ | **Deprecated** |

### 1. Authorization Code Flow (Most Common)
```
┌──────┐       ┌────────┐       ┌──────────────┐       ┌──────────────┐
│ User │       │ Client  │       │  Auth Server  │       │  Resource    │
│      │       │ (App)   │       │ (Google/KC)   │       │  Server (API)│
└──┬───┘       └───┬────┘       └──────┬───────┘       └──────┬───────┘
   │               │                    │                      │
   │  1. Click     │                    │                      │
   │  "Login"      │                    │                      │
   │──────────────▶│                    │                      │
   │               │                    │                      │
   │               │ 2. Redirect to     │                      │
   │               │ Auth Server        │                      │
   │◀──────────────│───────────────────▶│                      │
   │               │                    │                      │
   │ 3. User logs  │                    │                      │
   │ in & consents │                    │                      │
   │──────────────────────────────────▶│                      │
   │               │                    │                      │
   │               │ 4. Auth code       │                      │
   │               │◀───────────────────│                      │
   │◀──────────────│                    │                      │
   │               │                    │                      │
   │               │ 5. Exchange code   │                      │
   │               │ for tokens         │                      │
   │               │───────────────────▶│                      │
   │               │                    │                      │
   │               │ 6. Access Token    │                      │
   │               │ + Refresh Token    │                      │
   │               │◀───────────────────│                      │
   │               │                    │                      │
   │               │ 7. API call with   │                      │
   │               │ access token       │                      │
   │               │──────────────────────────────────────────▶│
   │               │                    │                      │
   │               │ 8. Protected       │                      │
   │               │ resource           │                      │
   │               │◀──────────────────────────────────────────│
   │  9. Response   │                    │                      │
   │◀──────────────│                    │                      │
```

### 2. Client Credentials Flow (Machine-to-Machine)
```
┌────────────┐             ┌──────────────┐             ┌──────────────┐
│   Client    │             │  Auth Server  │             │  Resource    │
│ (Service A) │             │               │             │  Server      │
└─────┬──────┘             └──────┬───────┘             └──────┬───────┘
      │                           │                            │
      │ 1. POST /token            │                            │
      │ client_id + client_secret │                            │
      │──────────────────────────▶│                            │
      │                           │                            │
      │ 2. Access Token           │                            │
      │◀──────────────────────────│                            │
      │                           │                            │
      │ 3. API call with token    │                            │
      │───────────────────────────────────────────────────────▶│
      │                           │                            │
      │ 4. Response               │                            │
      │◀───────────────────────────────────────────────────────│
```

### 3. Authorization Code + PKCE Flow (SPA / Mobile)
```
1. Client generates:
   - code_verifier  = random string (43-128 chars)
   - code_challenge = SHA256(code_verifier) → Base64URL encoded

2. Authorization request includes code_challenge

3. Token request includes code_verifier

4. Auth server verifies: SHA256(code_verifier) == stored code_challenge

Why? No client secret needed — prevents auth code interception attacks.
```

### 4. Device Code Flow
```
1. Device shows: "Go to https://auth.example.com/device and enter code: ABCD-1234"
2. User goes to URL on phone/laptop
3. User enters code and logs in
4. Device polls auth server until approved
5. Device gets access token
```

---

## 4. OPENID CONNECT (OIDC)

### What is OIDC?
- **OpenID Connect** = an **identity layer** on top of OAuth2.
- OAuth2 = authorization ("What can you do?")
- OIDC = authentication ("Who are you?")
- Adds the **ID Token** (JWT) containing user identity.

### OIDC vs OAuth2
| Feature | OAuth2 | OIDC |
|---------|--------|------|
| Purpose | Authorization | Authentication + Authorization |
| User info | Not provided | ID Token (JWT) |
| Standard scopes | None specified | `openid`, `profile`, `email`, `address`, `phone` |
| Discovery | No | `/.well-known/openid-configuration` |
| UserInfo endpoint | No | Yes (`/userinfo`) |

### OIDC Standard Scopes
| Scope | Claims Returned |
|-------|----------------|
| `openid` | `sub` (required for OIDC) |
| `profile` | `name`, `given_name`, `family_name`, `picture`, `locale` |
| `email` | `email`, `email_verified` |
| `address` | `address` (formatted, street, city, etc.) |
| `phone` | `phone_number`, `phone_number_verified` |

### ID Token (JWT) Structure
```json
{
  "iss": "https://accounts.google.com",      // issuer
  "sub": "1234567890",                        // subject (user ID)
  "aud": "your-client-id",                    // audience
  "exp": 1700003600,                          // expiration
  "iat": 1700000000,                          // issued at
  "nonce": "abc123",                          // replay protection
  "name": "John Doe",
  "email": "john@example.com",
  "picture": "https://example.com/photo.jpg"
}
```

### Discovery Endpoint
```
GET https://accounts.google.com/.well-known/openid-configuration

Response:
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "scopes_supported": ["openid", "email", "profile"],
  "response_types_supported": ["code", "id_token", "token"],
  ...
}
```

---

## 5. SPRING SECURITY OAUTH2 ARCHITECTURE

### Spring Security OAuth2 Modules
```
┌───────────────────────────────────────────────────────┐
│               Spring Security OAuth2                   │
├─────────────────┬─────────────────┬───────────────────┤
│  OAuth2 Client  │ OAuth2 Resource │ Authorization     │
│                 │ Server          │ Server            │
│                 │                 │ (Separate project)│
├─────────────────┼─────────────────┼───────────────────┤
│ - Social login  │ - Validate JWT  │ - Issue tokens    │
│ - Auth code     │ - Validate      │ - Manage clients  │
│   flow          │   opaque tokens │ - Consent UI      │
│ - Token mgmt    │ - Extract roles │ - PKCE support    │
│ - WebClient     │ - Resource      │ - OIDC support    │
│   integration   │   protection    │                   │
└─────────────────┴─────────────────┴───────────────────┘
```

### Which Module Do You Need?
| Scenario | Module |
|----------|--------|
| "Login with Google/GitHub" | **OAuth2 Client** |
| "My API accepts JWTs" | **OAuth2 Resource Server** |
| "I want to issue my own tokens" | **Spring Authorization Server** |
| "SPA calls my API via Google login" | Client (SPA) + Resource Server (API) |
| "Microservice-to-microservice" | Client Credentials + Resource Server |

---

## 6. PROJECT SETUP

### Maven Dependencies — OAuth2 Client
```xml
<!-- Social login / OAuth2 login -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

### Maven Dependencies — OAuth2 Resource Server
```xml
<!-- Protect APIs with JWT / opaque tokens -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### Maven Dependencies — Spring Authorization Server
```xml
<!-- Build your own authorization server -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
```

### Gradle Dependencies
```groovy
// Client
implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'

// Resource Server
implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'

// Authorization Server
implementation 'org.springframework.boot:spring-boot-starter-oauth2-authorization-server'
```

---

## 7. OAUTH2 CLIENT — SOCIAL LOGIN

### Google Login Setup

#### Step 1: Register App at Google Cloud Console
1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project
3. Navigate to **APIs & Services > Credentials**
4. Create **OAuth 2.0 Client ID**
5. Set **Authorized redirect URI**: `http://localhost:8080/login/oauth2/code/google`
6. Note down **Client ID** and **Client Secret**

#### Step 2: Application Properties
```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: your-google-client-id
            client-secret: your-google-client-secret
            scope: openid, profile, email
          github:
            client-id: your-github-client-id
            client-secret: your-github-client-secret
            scope: user:email, read:user
```

#### Step 3: Security Configuration
```java
@Configuration
@EnableWebSecurity
public class OAuth2LoginConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login", "/error", "/css/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth -> oauth
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/login?error")
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService())
                    .oidcUserService(customOidcUserService())
                )
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .permitAll()
            );

        return http.build();
    }
}
```

#### Step 4: Controller
```java
@Controller
public class LoginController {

    @GetMapping("/login")
    public String loginPage() {
        return "login";
    }

    @GetMapping("/dashboard")
    public String dashboard(@AuthenticationPrincipal OAuth2User user, Model model) {
        model.addAttribute("name", user.getAttribute("name"));
        model.addAttribute("email", user.getAttribute("email"));
        model.addAttribute("picture", user.getAttribute("picture"));
        return "dashboard";
    }
}
```

### GitHub Login Setup
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: user:email, read:user
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
```

### Custom OAuth2 Provider (e.g., Keycloak)
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: my-app
            client-secret: my-secret
            scope: openid, profile, email
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          keycloak:
            issuer-uri: http://localhost:8180/realms/my-realm
            # OR specify individual endpoints:
            # authorization-uri: http://localhost:8180/realms/my-realm/protocol/openid-connect/auth
            # token-uri: http://localhost:8180/realms/my-realm/protocol/openid-connect/token
            # user-info-uri: http://localhost:8180/realms/my-realm/protocol/openid-connect/userinfo
            # jwk-set-uri: http://localhost:8180/realms/my-realm/protocol/openid-connect/certs
            user-name-attribute: preferred_username
```

### Spring Boot Auto-Configured Providers
| Provider | Registration ID | Redirect URI |
|---------|----------------|-------------|
| Google | `google` | `/login/oauth2/code/google` |
| GitHub | `github` | `/login/oauth2/code/github` |
| Facebook | `facebook` | `/login/oauth2/code/facebook` |
| Okta | `okta` | `/login/oauth2/code/okta` |

> Only `client-id` and `client-secret` needed for these — everything else is auto-configured.

---

## 8. CUSTOM OAUTH2 USER SERVICE

### Why Customize?
- Map OAuth2 user to your local database user.
- Assign roles/authorities from your system.
- Persist OAuth2 users on first login (auto-registration).

### Custom OAuth2UserService (non-OIDC providers like GitHub)
```java
@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest)
            throws OAuth2AuthenticationException {

        OAuth2User oAuth2User = super.loadUser(userRequest);

        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        String email = extractEmail(oAuth2User, registrationId);
        String name = oAuth2User.getAttribute("name");

        // Find or create local user
        User user = userRepository.findByEmail(email)
            .orElseGet(() -> {
                User newUser = new User();
                newUser.setEmail(email);
                newUser.setName(name);
                newUser.setProvider(registrationId);
                newUser.setProviderId(oAuth2User.getName());
                newUser.setRole("USER");
                return userRepository.save(newUser);
            });

        // Build authorities from local user roles
        List<GrantedAuthority> authorities = List.of(
            new SimpleGrantedAuthority("ROLE_" + user.getRole())
        );

        return new DefaultOAuth2User(authorities,
            oAuth2User.getAttributes(),
            userRequest.getClientRegistration()
                .getProviderDetails()
                .getUserInfoEndpoint()
                .getUserNameAttributeName());
    }

    private String extractEmail(OAuth2User user, String registrationId) {
        return switch (registrationId) {
            case "google" -> user.getAttribute("email");
            case "github" -> user.getAttribute("email");
            default -> user.getAttribute("email");
        };
    }
}
```

### Custom OidcUserService (for OIDC providers like Google)
```java
@Service
public class CustomOidcUserService extends OidcUserService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public OidcUser loadUser(OidcUserRequest userRequest)
            throws OAuth2AuthenticationException {

        OidcUser oidcUser = super.loadUser(userRequest);

        String email = oidcUser.getEmail();
        String name = oidcUser.getFullName();

        // Find or create local user
        User user = userRepository.findByEmail(email)
            .orElseGet(() -> {
                User newUser = new User();
                newUser.setEmail(email);
                newUser.setName(name);
                newUser.setProvider("google");
                newUser.setProviderId(oidcUser.getSubject());
                newUser.setRole("USER");
                return userRepository.save(newUser);
            });

        Set<GrantedAuthority> authorities = new HashSet<>(oidcUser.getAuthorities());
        authorities.add(new SimpleGrantedAuthority("ROLE_" + user.getRole()));

        return new DefaultOidcUser(authorities,
            oidcUser.getIdToken(),
            oidcUser.getUserInfo());
    }
}
```

---

## 9. OAUTH2 CLIENT — LOGIN SUCCESS & FAILURE HANDLERS

### Success Handler
```java
@Component
public class OAuth2LoginSuccessHandler implements AuthenticationSuccessHandler {

    @Autowired
    private UserRepository userRepository;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
            HttpServletResponse response, Authentication authentication)
            throws IOException {

        OAuth2User oAuth2User = (OAuth2User) authentication.getPrincipal();
        String email = oAuth2User.getAttribute("email");

        User user = userRepository.findByEmail(email).orElseThrow();

        // Update last login
        user.setLastLoginAt(LocalDateTime.now());
        userRepository.save(user);

        // Redirect based on role
        if (user.getRole().equals("ADMIN")) {
            response.sendRedirect("/admin/dashboard");
        } else {
            response.sendRedirect("/dashboard");
        }
    }
}
```

### Failure Handler
```java
@Component
public class OAuth2LoginFailureHandler implements AuthenticationFailureHandler {

    private static final Logger log = LoggerFactory.getLogger(OAuth2LoginFailureHandler.class);

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,
            HttpServletResponse response, AuthenticationException exception)
            throws IOException {

        log.error("OAuth2 login failed: {}", exception.getMessage());
        response.sendRedirect("/login?error=oauth2");
    }
}
```

### Register Handlers
```java
http.oauth2Login(oauth -> oauth
    .successHandler(oAuth2LoginSuccessHandler)
    .failureHandler(oAuth2LoginFailureHandler)
);
```

---

## 10. OAUTH2 CLIENT — ACCESSING PROTECTED RESOURCES

### Using WebClient with OAuth2 (Recommended)
```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(OAuth2AuthorizedClientManager authorizedClientManager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction filter =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager);

        return WebClient.builder()
            .apply(filter.oauth2Configuration())
            .build();
    }

    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clientRegistrations,
            OAuth2AuthorizedClientRepository authorizedClients) {

        OAuth2AuthorizedClientProvider clientProvider =
            OAuth2AuthorizedClientProviderBuilder.builder()
                .authorizationCode()
                .refreshToken()
                .clientCredentials()
                .build();

        DefaultOAuth2AuthorizedClientManager manager =
            new DefaultOAuth2AuthorizedClientManager(clientRegistrations, authorizedClients);
        manager.setAuthorizedClientProvider(clientProvider);

        return manager;
    }
}
```

### Calling Protected APIs
```java
@Service
public class GitHubApiService {

    @Autowired
    private WebClient webClient;

    public List<Map<String, Object>> getUserRepos() {
        return webClient.get()
            .uri("https://api.github.com/user/repos")
            .attributes(
                ServletOAuth2AuthorizedClientExchangeFilterFunction
                    .clientRegistrationId("github"))
            .retrieve()
            .bodyToMono(new ParameterizedTypeReference<List<Map<String, Object>>>() {})
            .block();
    }
}
```

### Client Credentials — Service-to-Service
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          my-service:
            client-id: service-client-id
            client-secret: service-client-secret
            authorization-grant-type: client_credentials
            scope: api.read
        provider:
          my-service:
            token-uri: http://auth-server:9000/oauth2/token
```

```java
@Service
public class ServiceClient {

    @Autowired
    private WebClient webClient;

    public String callDownstreamService() {
        return webClient.get()
            .uri("http://downstream-service/api/data")
            .attributes(clientRegistrationId("my-service"))
            .retrieve()
            .bodyToMono(String.class)
            .block();
    }
}
```

---

## 11. OAUTH2 RESOURCE SERVER — JWT

### What is a Resource Server?
- A server that **hosts protected resources** (REST APIs).
- Validates access tokens (JWT or opaque) on every request.
- Does NOT handle login — only token validation.

### Configuration — JWT
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://accounts.google.com
          # OR
          jwk-set-uri: https://www.googleapis.com/oauth2/v3/certs
```

### Security Configuration
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthConverter())
                )
            );

        return http.build();
    }
}
```

### How JWT Validation Works
```
1. Client sends: Authorization: Bearer eyJhbGciOi...

2. Spring Security:
   a. Extracts JWT from Authorization header
   b. Downloads public keys from JWK Set URI (cached)
   c. Verifies JWT signature
   d. Checks: issuer, audience, expiration
   e. Extracts claims → builds Authentication

3. If valid → request proceeds
   If invalid → 401 Unauthorized
```

### Controller with JWT Authentication
```java
@RestController
@RequestMapping("/api")
public class ApiController {

    @GetMapping("/public/health")
    public String health() {
        return "OK";
    }

    @GetMapping("/profile")
    public Map<String, Object> profile(@AuthenticationPrincipal Jwt jwt) {
        return Map.of(
            "subject", jwt.getSubject(),
            "issuer", jwt.getIssuer().toString(),
            "claims", jwt.getClaims(),
            "issuedAt", jwt.getIssuedAt(),
            "expiresAt", jwt.getExpiresAt()
        );
    }

    @GetMapping("/admin/users")
    @PreAuthorize("hasRole('ADMIN')")
    public String adminEndpoint() {
        return "Admin data";
    }
}
```

---

## 12. OAUTH2 RESOURCE SERVER — OPAQUE TOKENS

### When to Use Opaque Tokens?
- Token is **not self-contained** — server must call the auth server to validate.
- More secure (tokens can be revoked instantly).
- Less performant (extra network call per request).

### Configuration
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: http://auth-server:9000/oauth2/introspect
          client-id: resource-server-client
          client-secret: resource-server-secret
```

### Security Configuration
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2
            .opaqueToken(opaque -> opaque
                .introspector(customIntrospector())
            )
        );

    return http.build();
}
```

### Custom Introspector
```java
@Bean
public OpaqueTokenIntrospector customIntrospector() {
    OpaqueTokenIntrospector delegate =
        new NimbusOpaqueTokenIntrospector(introspectionUri, clientId, clientSecret);

    return token -> {
        OAuth2AuthenticatedPrincipal principal = delegate.introspect(token);

        // Add custom authorities
        Collection<GrantedAuthority> authorities = extractAuthorities(principal);

        return new DefaultOAuth2AuthenticatedPrincipal(
            principal.getName(), principal.getAttributes(), authorities);
    };
}
```

### JWT vs Opaque Tokens
| Aspect | JWT | Opaque Token |
|--------|-----|-------------|
| Validation | Local (signature check) | Remote (introspection call) |
| Performance | Fast (no network call) | Slower (network call per request) |
| Revocation | Hard (wait for expiry) | Instant (server-side) |
| Size | Large (contains claims) | Small (random string) |
| Self-contained | Yes | No |
| Best for | High-traffic APIs | High-security APIs |

---

## 13. CUSTOM JWT DECODER & VALIDATORS

### Custom JwtDecoder
```java
@Bean
public JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withJwkSetUri("https://auth-server/.well-known/jwks.json")
        .build();

    // Add custom validators
    OAuth2TokenValidator<Jwt> withIssuer =
        JwtValidators.createDefaultWithIssuer("https://auth-server");

    OAuth2TokenValidator<Jwt> audienceValidator = new AudienceValidator("my-api");

    OAuth2TokenValidator<Jwt> combined =
        new DelegatingOAuth2TokenValidator<>(withIssuer, audienceValidator);

    decoder.setJwtValidator(combined);
    return decoder;
}
```

### Custom Audience Validator
```java
public class AudienceValidator implements OAuth2TokenValidator<Jwt> {

    private final String audience;

    public AudienceValidator(String audience) {
        this.audience = audience;
    }

    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        if (jwt.getAudience().contains(audience)) {
            return OAuth2TokenValidatorResult.success();
        }
        OAuth2Error error = new OAuth2Error("invalid_token",
            "Required audience " + audience + " is missing", null);
        return OAuth2TokenValidatorResult.failure(error);
    }
}
```

### JWT with Symmetric Key (HMAC)
```java
@Bean
public JwtDecoder jwtDecoder() {
    SecretKey key = new SecretKeySpec(
        "your-256-bit-secret-key-here-min32chars".getBytes(),
        "HmacSHA256");
    return NimbusJwtDecoder.withSecretKey(key).build();
}
```

### JWT with RSA Public Key
```java
@Bean
public JwtDecoder jwtDecoder(@Value("classpath:public-key.pem") RSAPublicKey key) {
    return NimbusJwtDecoder.withPublicKey(key).build();
}
```

---

## 14. ROLES & AUTHORITIES FROM JWT CLAIMS

### Default Behavior
- Spring Security maps JWT `scope` or `scp` claim to authorities prefixed with `SCOPE_`.
- Example: `"scope": "read write"` → authorities `SCOPE_read`, `SCOPE_write`.

### Custom JWT Authentication Converter
```java
@Bean
public JwtAuthenticationConverter jwtAuthConverter() {
    JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
        new JwtGrantedAuthoritiesConverter();
    grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
    grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");

    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
    return converter;
}
```

### Custom Converter for Complex JWT Claims
```java
// JWT payload example:
// {
//   "realm_access": {
//     "roles": ["admin", "user"]
//   },
//   "resource_access": {
//     "my-app": {
//       "roles": ["app-admin"]
//     }
//   }
// }

@Bean
public JwtAuthenticationConverter jwtAuthConverter() {
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();

    converter.setJwtGrantedAuthoritiesConverter(jwt -> {
        Set<GrantedAuthority> authorities = new HashSet<>();

        // Extract realm roles
        Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
        if (realmAccess != null) {
            List<String> roles = (List<String>) realmAccess.get("roles");
            if (roles != null) {
                roles.forEach(role ->
                    authorities.add(new SimpleGrantedAuthority("ROLE_" + role.toUpperCase())));
            }
        }

        // Extract resource/client-specific roles
        Map<String, Object> resourceAccess = jwt.getClaimAsMap("resource_access");
        if (resourceAccess != null) {
            Map<String, Object> clientAccess =
                (Map<String, Object>) resourceAccess.get("my-app");
            if (clientAccess != null) {
                List<String> clientRoles = (List<String>) clientAccess.get("roles");
                if (clientRoles != null) {
                    clientRoles.forEach(role ->
                        authorities.add(new SimpleGrantedAuthority("ROLE_" + role.toUpperCase())));
                }
            }
        }

        return authorities;
    });

    return converter;
}
```

### Register in Security Config
```java
http.oauth2ResourceServer(oauth2 -> oauth2
    .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()))
);
```

---

## 15. SPRING AUTHORIZATION SERVER

### What is Spring Authorization Server?
- A Spring project that provides **OAuth 2.1** and **OpenID Connect 1.0** authorization server support.
- Replaces the deprecated **Spring Security OAuth** project.
- Built on **Spring Security 6.x**.
- Supports: Authorization Code, Client Credentials, Refresh Token, Device Code, PKCE.

### Minimal Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
```

### Basic Configuration
```java
@Configuration
public class AuthorizationServerConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain authServerFilterChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);

        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            .oidc(Customizer.withDefaults());  // enable OIDC

        http
            .exceptionHandling(ex -> ex
                .defaultAuthenticationEntryPointFor(
                    new LoginUrlAuthenticationEntryPoint("/login"),
                    new MediaTypeRequestMatcher(MediaType.TEXT_HTML)
                )
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));

        return http.build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.builder()
            .username("user")
            .password("{bcrypt}$2a$10$...")
            .roles("USER")
            .build();

        return new InMemoryUserDetailsManager(user);
    }
}
```

---

## 16. AUTHORIZATION SERVER — CLIENT REGISTRATION

### In-Memory Client Registration
```java
@Bean
public RegisteredClientRepository registeredClientRepository() {

    // Web application client (Authorization Code flow)
    RegisteredClient webClient = RegisteredClient.withId(UUID.randomUUID().toString())
        .clientId("web-app")
        .clientSecret("{bcrypt}$2a$10$...")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
        .redirectUri("http://localhost:8080/login/oauth2/code/my-auth-server")
        .postLogoutRedirectUri("http://localhost:8080/")
        .scope(OidcScopes.OPENID)
        .scope(OidcScopes.PROFILE)
        .scope(OidcScopes.EMAIL)
        .scope("api.read")
        .scope("api.write")
        .clientSettings(ClientSettings.builder()
            .requireAuthorizationConsent(true)
            .requireProofKey(false)  // set true for PKCE
            .build())
        .tokenSettings(TokenSettings.builder()
            .accessTokenTimeToLive(Duration.ofMinutes(30))
            .refreshTokenTimeToLive(Duration.ofDays(7))
            .reuseRefreshTokens(false)
            .accessTokenFormat(OAuth2TokenFormat.SELF_CONTAINED) // JWT
            .build())
        .build();

    // Service client (Client Credentials flow)
    RegisteredClient serviceClient = RegisteredClient.withId(UUID.randomUUID().toString())
        .clientId("service-client")
        .clientSecret("{bcrypt}$2a$10$...")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
        .scope("api.read")
        .scope("api.write")
        .tokenSettings(TokenSettings.builder()
            .accessTokenTimeToLive(Duration.ofHours(1))
            .accessTokenFormat(OAuth2TokenFormat.SELF_CONTAINED)
            .build())
        .build();

    // SPA / Public client (Authorization Code + PKCE)
    RegisteredClient spaClient = RegisteredClient.withId(UUID.randomUUID().toString())
        .clientId("spa-client")
        // No client secret — public client
        .clientAuthenticationMethod(ClientAuthenticationMethod.NONE)
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
        .redirectUri("http://localhost:3000/callback")
        .scope(OidcScopes.OPENID)
        .scope(OidcScopes.PROFILE)
        .scope("api.read")
        .clientSettings(ClientSettings.builder()
            .requireProofKey(true)  // require PKCE
            .requireAuthorizationConsent(true)
            .build())
        .build();

    return new InMemoryRegisteredClientRepository(webClient, serviceClient, spaClient);
}
```

### JDBC Client Registration (Production)
```java
@Bean
public RegisteredClientRepository registeredClientRepository(JdbcTemplate jdbcTemplate) {
    return new JdbcRegisteredClientRepository(jdbcTemplate);
}

// Required tables — Spring provides schema:
// classpath:org/springframework/security/oauth2/server/authorization/
//   client/oauth2-registered-client-schema.sql
```

### JWK Source (Signing Keys)
```java
@Bean
public JWKSource<SecurityContext> jwkSource() {
    KeyPair keyPair = generateRsaKey();
    RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
    RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();

    RSAKey rsaKey = new RSAKey.Builder(publicKey)
        .privateKey(privateKey)
        .keyID(UUID.randomUUID().toString())
        .build();

    JWKSet jwkSet = new JWKSet(rsaKey);
    return new ImmutableJWKSet<>(jwkSet);
}

private static KeyPair generateRsaKey() {
    try {
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
        keyPairGenerator.initialize(2048);
        return keyPairGenerator.generateKeyPair();
    } catch (Exception ex) {
        throw new IllegalStateException(ex);
    }
}

@Bean
public JwtDecoder jwtDecoder(JWKSource<SecurityContext> jwkSource) {
    return OAuth2AuthorizationServerConfiguration.jwtDecoder(jwkSource);
}

@Bean
public AuthorizationServerSettings authorizationServerSettings() {
    return AuthorizationServerSettings.builder()
        .issuer("http://localhost:9000")
        .build();
}
```

---

## 17. AUTHORIZATION SERVER — TOKEN CUSTOMIZATION

### Customize JWT Claims (Access Token)
```java
@Bean
public OAuth2TokenCustomizer<JwtEncodingContext> tokenCustomizer() {
    return context -> {
        if (context.getTokenType().equals(OAuth2TokenType.ACCESS_TOKEN)) {
            Authentication principal = context.getPrincipal();
            Set<String> authorities = principal.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toSet());

            context.getClaims().claims(claims -> {
                claims.put("roles", authorities);
                claims.put("custom-claim", "custom-value");
            });
        }

        // Customize ID Token
        if (context.getTokenType().getValue().equals(OidcParameterNames.ID_TOKEN)) {
            context.getClaims().claims(claims -> {
                claims.put("app_roles", List.of("USER"));
            });
        }
    };
}
```

### Customize UserInfo Endpoint
```java
@Bean
public Function<OidcUserInfoAuthenticationContext, OidcUserInfo> userInfoMapper() {
    return context -> {
        OidcUserInfoAuthenticationToken authentication = context.getAuthentication();
        JwtAuthenticationToken principal =
            (JwtAuthenticationToken) authentication.getPrincipal();
        Jwt jwt = principal.getToken();

        return OidcUserInfo.builder()
            .subject(jwt.getSubject())
            .name(jwt.getClaimAsString("name"))
            .email(jwt.getClaimAsString("email"))
            .claim("roles", jwt.getClaimAsStringList("roles"))
            .build();
    };
}
```

---

## 18. AUTHORIZATION SERVER — CONSENT FLOW

### How Consent Works
```
1. Client requests authorization with scopes
2. User logs in
3. Auth server shows consent page:
   "MyApp wants to access:
    ✅ Your profile
    ✅ Your email
    ☐ Write access
    [Allow] [Deny]"
4. User selects scopes and approves
5. Auth server issues tokens with approved scopes only
```

### Custom Consent Page
```java
@Controller
public class ConsentController {

    @GetMapping("/oauth2/consent")
    public String consent(Principal principal, Model model,
            @RequestParam String client_id,
            @RequestParam String scope,
            @RequestParam String state) {

        Set<String> scopes = Set.of(scope.split(" "));

        model.addAttribute("clientId", client_id);
        model.addAttribute("state", state);
        model.addAttribute("scopes", scopes);
        model.addAttribute("principalName", principal.getName());

        return "consent";  // Thymeleaf template
    }
}
```

---

## 19. PKCE (PROOF KEY FOR CODE EXCHANGE)

### What is PKCE?
- **RFC 7636** — protection against authorization code interception.
- Required for **public clients** (SPAs, mobile apps) that can't securely store a client secret.
- Recommended for **all OAuth2 clients** (even confidential ones — OAuth 2.1).

### How PKCE Works
```
1. Client generates:
   code_verifier  = random(43-128 chars, URL-safe)
   code_challenge = BASE64URL(SHA256(code_verifier))

2. Authorization Request:
   GET /authorize?
     response_type=code&
     client_id=spa-client&
     redirect_uri=http://localhost:3000/callback&
     code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
     code_challenge_method=S256

3. Auth server stores code_challenge

4. Token Request:
   POST /token
     grant_type=authorization_code&
     code=AUTH_CODE&
     code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk

5. Auth server verifies:
   BASE64URL(SHA256(code_verifier)) == stored code_challenge
```

### Enable PKCE on Client
```java
RegisteredClient spaClient = RegisteredClient.withId(UUID.randomUUID().toString())
    .clientId("spa-client")
    .clientAuthenticationMethod(ClientAuthenticationMethod.NONE)
    .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
    .redirectUri("http://localhost:3000/callback")
    .scope(OidcScopes.OPENID)
    .scope("api.read")
    .clientSettings(ClientSettings.builder()
        .requireProofKey(true)  // enforces PKCE
        .build())
    .build();
```

---

## 20. TOKEN RELAY & PROPAGATION

### Token Relay in Gateway
```yaml
# Spring Cloud Gateway with OAuth2 login — relay token to downstream services
spring:
  cloud:
    gateway:
      default-filters:
        - TokenRelay  # passes the access token to downstream services
      routes:
        - id: resource-service
          uri: http://localhost:8081
          predicates:
            - Path=/api/**
```

### Token Propagation with RestClient/WebClient
```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(OAuth2AuthorizedClientManager clientManager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction filter =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(clientManager);
        filter.setDefaultClientRegistrationId("my-service");

        return WebClient.builder()
            .apply(filter.oauth2Configuration())
            .build();
    }
}
```

### Manual Token Propagation with RestTemplate
```java
@Component
public class OAuth2RestTemplateInterceptor implements ClientHttpRequestInterceptor {

    @Autowired
    private OAuth2AuthorizedClientService clientService;

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution) throws IOException {

        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof OAuth2AuthenticationToken oauthToken) {
            OAuth2AuthorizedClient client = clientService.loadAuthorizedClient(
                oauthToken.getAuthorizedClientRegistrationId(), oauthToken.getName());

            if (client != null) {
                request.getHeaders().setBearerAuth(
                    client.getAccessToken().getTokenValue());
            }
        }

        return execution.execute(request, body);
    }
}
```

---

## 21. OAUTH2 WITH MICROSERVICES (API GATEWAY PATTERN)

### Architecture
```
┌──────────┐       ┌────────────────┐       ┌──────────────┐
│  Client   │       │   API Gateway   │       │  Auth Server  │
│  (SPA)    │       │ (Spring Cloud   │       │ (Keycloak /   │
│           │       │  Gateway)       │       │  Spring Auth  │
└─────┬────┘       └───────┬────────┘       │  Server)      │
      │                     │                └──────┬───────┘
      │ 1. Login redirect   │                       │
      │────────────────────▶│──────────────────────▶│
      │                     │                       │
      │ 2. Tokens           │◀──────────────────────│
      │◀────────────────────│                       │
      │                     │                       │
      │ 3. API call         │                       │
      │  (with token)       │                       │
      │────────────────────▶│                       │
      │                     │                       │
      │              ┌──────┴──────┐                │
      │              │ Token Relay  │                │
      │              └──────┬──────┘                │
      │                     │                       │
      │              ┌──────▼──────┐                │
      │              │  Microservice│                │
      │              │ (Resource    │                │
      │              │  Server)     │                │
      │              └─────────────┘                │
```

### Gateway Configuration
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: gateway-client
            client-secret: gateway-secret
            scope: openid, profile, email
            authorization-grant-type: authorization_code
        provider:
          keycloak:
            issuer-uri: http://localhost:8180/realms/my-realm
  cloud:
    gateway:
      default-filters:
        - TokenRelay
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
```

### Microservice (Resource Server) Configuration
```yaml
# Each microservice validates JWT independently
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8180/realms/my-realm
```

---

## 22. TOKEN REVOCATION & INTROSPECTION

### Token Revocation Endpoint
```
POST /oauth2/revoke
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

token=ACCESS_TOKEN_VALUE&token_type_hint=access_token
```

### Token Introspection Endpoint
```
POST /oauth2/introspect
Content-Type: application/x-www-form-urlencoded
Authorization: Basic base64(client_id:client_secret)

token=ACCESS_TOKEN_VALUE

Response:
{
  "active": true,
  "sub": "user123",
  "client_id": "my-app",
  "scope": "openid profile email",
  "exp": 1700003600,
  "iat": 1700000000,
  "iss": "http://auth-server:9000"
}
```

### Programmatic Token Revocation
```java
@Service
public class TokenRevocationService {

    @Autowired
    private OAuth2AuthorizationService authorizationService;

    public void revokeAllTokens(String username) {
        // Custom implementation — find and revoke all authorizations for user
        // Useful for "logout everywhere" functionality
    }
}
```

### Authorization Service (Persistence)
```java
// In-memory (development)
@Bean
public OAuth2AuthorizationService authorizationService() {
    return new InMemoryOAuth2AuthorizationService();
}

// JDBC (production)
@Bean
public OAuth2AuthorizationService authorizationService(
        JdbcTemplate jdbcTemplate,
        RegisteredClientRepository clientRepository) {
    return new JdbcOAuth2AuthorizationService(jdbcTemplate, clientRepository);
}
```

---

## 23. OAUTH2 WITH REACT / ANGULAR SPA

### Recommended Architecture
```
                          Public Client (PKCE)
┌──────────────┐          ┌──────────────┐          ┌──────────────┐
│   React App   │  ──────▶ │  Auth Server  │ ◀──────  │ Resource     │
│   (Browser)   │          │  (Keycloak)   │          │ Server (API) │
│               │ ◀──────  │               │          │              │
│  Stores token │          │               │          │              │
│  in memory    │ ─────────────────────────────────▶ │              │
│               │          │               │          │              │
│               │ ◀─────────────────────────────────  │              │
└──────────────┘          └──────────────┘          └──────────────┘
```

### Backend for Frontend (BFF) Pattern (More Secure)
```
┌──────────────┐          ┌──────────────┐          ┌──────────────┐
│   React App   │  ──────▶ │    BFF        │ ──────▶ │  Auth Server  │
│   (Browser)   │          │  (Spring Boot │          │  (Keycloak)   │
│               │ ◀──────  │   Gateway)    │ ◀──────  │               │
│  No tokens    │          │              │          │               │
│  Uses cookies │          │  Stores      │          │               │
│               │          │  tokens      │          │               │
│               │          │  server-side │          │               │
│               │          │              │ ──────▶ ┌──────────────┐
│               │          │  Token Relay │          │ Resource     │
│               │          │              │ ◀──────  │ Server (API) │
└──────────────┘          └──────────────┘          └──────────────┘
```

### SPA Token Storage Best Practices
| Storage | Security | Recommendation |
|---------|---------|----------------|
| `localStorage` | Vulnerable to XSS | **Avoid** |
| `sessionStorage` | Vulnerable to XSS | **Avoid** |
| In-memory (variable) | Safe from XSS, lost on refresh | **Good** |
| `HttpOnly` cookie (BFF) | Safe from XSS | **Best** |

---

## 24. SECURITY BEST PRACTICES FOR OAUTH2

### Token Security
1. **Use short-lived access tokens** (5-30 minutes).
2. **Use refresh tokens** for long sessions — store server-side.
3. **Always use HTTPS** — tokens are bearer credentials.
4. **Never expose tokens in URLs** — use POST bodies or headers.
5. **Validate `iss`, `aud`, `exp` claims** on resource server.
6. **Use asymmetric keys (RSA/EC)** for JWT signing in production.

### Client Security
7. **Use PKCE for all public clients** (SPAs, mobile).
8. **Store client secrets server-side only** — never in frontend code.
9. **Use the BFF pattern** for SPAs when possible.
10. **Validate redirect URIs strictly** — exact match, no wildcards.

### Authorization Server Security
11. **Rate limit token endpoints** — prevent credential stuffing.
12. **Rotate signing keys** periodically.
13. **Implement token revocation** for logout flows.
14. **Use consent screens** for third-party apps.

### General
15. **Prefer Authorization Code + PKCE** over all other flows.
16. **Never use Implicit or Password grant** (deprecated in OAuth 2.1).
17. **Validate scopes** — issue only the minimum required.
18. **Log authentication events** — monitor for anomalies.

---

## 25. TESTING OAUTH2

### Testing OAuth2 Resource Server (JWT)
```java
@SpringBootTest
@AutoConfigureMockMvc
class ResourceServerTest {

    @Autowired
    private MockMvc mockMvc;

    // Test with mock JWT
    @Test
    void whenValidJwt_thenAccessGranted() throws Exception {
        mockMvc.perform(get("/api/data")
                .with(jwt()
                    .jwt(j -> j
                        .subject("user@example.com")
                        .claim("roles", List.of("ROLE_USER"))
                    )
                    .authorities(new SimpleGrantedAuthority("ROLE_USER"))
                ))
            .andExpect(status().isOk());
    }

    // Test without authentication
    @Test
    void whenNoToken_thenUnauthorized() throws Exception {
        mockMvc.perform(get("/api/data"))
            .andExpect(status().isUnauthorized());
    }

    // Test with insufficient permissions
    @Test
    void whenUserAccessAdmin_thenForbidden() throws Exception {
        mockMvc.perform(get("/api/admin")
                .with(jwt().authorities(new SimpleGrantedAuthority("ROLE_USER"))))
            .andExpect(status().isForbidden());
    }
}
```

### Testing OAuth2 Client (Login)
```java
@SpringBootTest
@AutoConfigureMockMvc
class OAuth2LoginTest {

    @Autowired
    private MockMvc mockMvc;

    // Test with mock OAuth2 user
    @Test
    void whenOAuth2Login_thenAccessGranted() throws Exception {
        mockMvc.perform(get("/dashboard")
                .with(oauth2Login()
                    .oauth2User(new DefaultOAuth2User(
                        List.of(new SimpleGrantedAuthority("ROLE_USER")),
                        Map.of("name", "John", "email", "john@example.com"),
                        "name"))
                ))
            .andExpect(status().isOk());
    }

    // Test with mock OIDC user
    @Test
    void whenOidcLogin_thenAccessGranted() throws Exception {
        mockMvc.perform(get("/dashboard")
                .with(oidcLogin()
                    .idToken(token -> token
                        .subject("user123")
                        .claim("name", "John Doe")
                        .claim("email", "john@example.com"))
                    .authorities(new SimpleGrantedAuthority("ROLE_USER"))
                ))
            .andExpect(status().isOk());
    }
}
```

### Testing with Opaque Token
```java
@Test
void whenValidOpaqueToken_thenAccessGranted() throws Exception {
    mockMvc.perform(get("/api/data")
            .with(opaqueToken()
                .attributes(attrs -> attrs.put("sub", "user123"))
                .authorities(new SimpleGrantedAuthority("ROLE_USER"))
            ))
        .andExpect(status().isOk());
}
```

---

## 26. COMMON CONFIGURATION PROPERTIES REFERENCE

### OAuth2 Client Properties
```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          {registrationId}:
            client-id: ...
            client-secret: ...
            scope: openid, profile, email
            authorization-grant-type: authorization_code  # or client_credentials
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            client-name: My App
            client-authentication-method: client_secret_basic
        provider:
          {providerId}:
            issuer-uri: https://auth-server/.well-known/...
            authorization-uri: https://auth-server/authorize
            token-uri: https://auth-server/token
            user-info-uri: https://auth-server/userinfo
            jwk-set-uri: https://auth-server/jwks
            user-name-attribute: sub
```

### OAuth2 Resource Server Properties
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth-server
          jwk-set-uri: https://auth-server/.well-known/jwks.json
          audiences: my-api
        # OR for opaque tokens:
        opaquetoken:
          introspection-uri: https://auth-server/introspect
          client-id: resource-server
          client-secret: secret
```

### Spring Authorization Server Properties
```yaml
spring:
  security:
    oauth2:
      authorizationserver:
        issuer: http://localhost:9000
        endpoint:
          authorization-uri: /oauth2/authorize
          token-uri: /oauth2/token
          jwk-set-uri: /oauth2/jwks
          token-revocation-uri: /oauth2/revoke
          token-introspection-uri: /oauth2/introspect
          oidc:
            user-info-uri: /userinfo
            client-registration-uri: /connect/register
```

---

## 27. IMPORTANT ANNOTATIONS REFERENCE

| Annotation | Description |
|-----------|-------------|
| `@EnableWebSecurity` | Enable Spring Security web configuration |
| `@EnableMethodSecurity` | Enable method-level security (`@PreAuthorize`, etc.) |
| `@AuthenticationPrincipal` | Inject current OAuth2User / OidcUser / Jwt |
| `@RegisteredOAuth2AuthorizedClient` | Inject OAuth2AuthorizedClient |
| `@PreAuthorize("hasRole('ADMIN')")` | Method-level authorization |
| `@WithMockUser` | Test with mock Spring Security user |

### Injection Examples
```java
// Inject OAuth2 user (social login)
@GetMapping("/user")
public String user(@AuthenticationPrincipal OAuth2User user) {
    return user.getAttribute("name");
}

// Inject OIDC user
@GetMapping("/user")
public String user(@AuthenticationPrincipal OidcUser user) {
    return user.getEmail();
}

// Inject JWT (resource server)
@GetMapping("/user")
public String user(@AuthenticationPrincipal Jwt jwt) {
    return jwt.getSubject();
}

// Inject authorized client
@GetMapping("/repos")
public String repos(@RegisteredOAuth2AuthorizedClient("github")
                      OAuth2AuthorizedClient client) {
    String token = client.getAccessToken().getTokenValue();
    // use token to call GitHub API
}
```

---

## 28. TROUBLESHOOTING & COMMON ERRORS

### Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `[invalid_redirect_uri]` | Redirect URI mismatch | Ensure redirect URI in app matches auth server exactly |
| `[invalid_client]` | Wrong client ID/secret | Verify credentials |
| `[access_denied]` | User denied consent | Handle gracefully — redirect to error page |
| `[invalid_token]` | Expired or malformed JWT | Check token expiration, issuer, audience |
| `[insufficient_scope]` | Token missing required scope | Request additional scopes |
| `401 on resource server` | JWT not validated | Check `issuer-uri` / `jwk-set-uri` config |
| `CORS error on token endpoint` | Missing CORS config | Configure CORS on auth server |
| `No AuthenticationProvider` | Missing UserDetailsService | Register UserDetailsService bean |
| `CSRF token missing` | Stateful app not sending CSRF | Add CSRF token to forms |

### Debugging OAuth2
```properties
# Enable comprehensive logging
logging.level.org.springframework.security=DEBUG
logging.level.org.springframework.security.oauth2=TRACE
logging.level.org.springframework.web.client.RestTemplate=DEBUG

# Or target specific components
logging.level.org.springframework.security.oauth2.client=DEBUG
logging.level.org.springframework.security.oauth2.server.resource=DEBUG
logging.level.org.springframework.security.oauth2.server.authorization=DEBUG
```

### OAuth2 Endpoints Reference (Authorization Server)
| Endpoint | Path | Purpose |
|---------|------|---------|
| Authorization | `/oauth2/authorize` | User login & consent |
| Token | `/oauth2/token` | Exchange code for tokens |
| JWK Set | `/oauth2/jwks` | Public keys for JWT verification |
| Token Revocation | `/oauth2/revoke` | Revoke access/refresh tokens |
| Token Introspection | `/oauth2/introspect` | Validate opaque tokens |
| OIDC Discovery | `/.well-known/openid-configuration` | Auto-discovery metadata |
| UserInfo | `/userinfo` | Get authenticated user info |

---
