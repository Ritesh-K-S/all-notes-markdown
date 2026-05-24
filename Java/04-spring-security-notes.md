# Spring Security — Complete Notes (Beginner to Advanced)

---

## Table of Contents

1. [Introduction to Spring Security](#1-introduction-to-spring-security)
2. [Architecture & Core Components](#2-architecture--core-components)
3. [Project Setup](#3-project-setup)
4. [Default Security Behavior](#4-default-security-behavior)
5. [SecurityFilterChain Configuration](#5-securityfilterchain-configuration)
6. [Authentication](#6-authentication)
7. [UserDetailsService & UserDetails](#7-userdetailsservice--userdetails)
8. [Password Encoding](#8-password-encoding)
9. [In-Memory Authentication](#9-in-memory-authentication)
10. [JDBC Authentication](#10-jdbc-authentication)
11. [JPA / Custom UserDetailsService Authentication](#11-jpa--custom-userdetailsservice-authentication)
12. [Authorization — URL-Based](#12-authorization--url-based)
13. [Authorization — Method-Level](#13-authorization--method-level)
14. [Roles vs Authorities](#14-roles-vs-authorities)
15. [Form-Based Login](#15-form-based-login)
16. [HTTP Basic Authentication](#16-http-basic-authentication)
17. [Remember Me](#17-remember-me)
18. [Logout](#18-logout)
19. [Session Management](#19-session-management)
20. [CSRF Protection](#20-csrf-protection)
21. [CORS Configuration](#21-cors-configuration)
22. [Exception Handling (AuthenticationEntryPoint & AccessDeniedHandler)](#22-exception-handling)
23. [Custom Filters](#23-custom-filters)
24. [JWT Authentication](#24-jwt-authentication)
25. [Security Context & SecurityContextHolder](#25-security-context--securitycontextholder)
26. [LDAP Authentication](#26-ldap-authentication)
27. [Multi-Factor Authentication (MFA)](#27-multi-factor-authentication-mfa)
28. [Security Events & Auditing](#28-security-events--auditing)
29. [Security Headers](#29-security-headers)
30. [Spring Security with Thymeleaf](#30-spring-security-with-thymeleaf)
31. [Testing Spring Security](#31-testing-spring-security)
32. [Common Patterns & Best Practices](#32-common-patterns--best-practices)
33. [Important Annotations Reference](#33-important-annotations-reference)
34. [Common Configuration Properties](#34-common-configuration-properties)
35. [Migration Guide — WebSecurityConfigurerAdapter to Component-Based](#35-migration-guide)

**PART 2 — Deep Dive (Internal Flow & Advanced Understanding)**

36. [Deep Dive — How Spring Security Initializes (Servlet Level)](#36-deep-dive--how-spring-security-initializes-servlet-level)
37. [Deep Dive — Which Filters Run for Which Request?](#37-deep-dive--which-filters-run-for-which-request)
38. [Deep Dive — AuthenticationManager & Provider Selection](#38-deep-dive--authentication-manager--provider-selection)
39. [Deep Dive — Custom Authentication (OTP Example)](#39-deep-dive--custom-authentication-otp-example)
40. [Deep Dive — SecurityContextHolder: Stateful vs Stateless](#40-deep-dive--securitycontextholder-stateful-vs-stateless)
41. [Deep Dive — Complete Request Lifecycle (Every Filter)](#41-deep-dive--complete-request-lifecycle-every-filter)
42. [Key Mental Model — Cheat Sheet](#42-key-mental-model--cheat-sheet)

---

## 1. INTRODUCTION TO SPRING SECURITY

### What is Spring Security?
- **Spring Security** = a powerful, highly customizable authentication and access-control framework for Java applications.
- Part of the **Spring Ecosystem** — integrates seamlessly with Spring Boot, Spring MVC, Spring Data.
- Originally called **Acegi Security** (renamed in 2008).
- Current version: **Spring Security 6.x** (requires Spring Framework 6.x, Java 17+, Spring Boot 3.x).

### Why Spring Security?
| Feature | Benefit |
|---------|---------|
| Authentication | Verify *who* the user is |
| Authorization | Control *what* the user can access |
| Protection against attacks | CSRF, XSS, session fixation, clickjacking |
| Servlet & Reactive support | Works with Spring MVC and Spring WebFlux |
| Extensible | Plug in custom auth providers, filters, handlers |
| Integration | LDAP, OAuth2, SAML, JWT, OpenID Connect |
| Production-ready | Battle-tested in thousands of enterprise apps |

### What Spring Security Protects Against
| Attack | Protection |
|--------|-----------|
| **CSRF** (Cross-Site Request Forgery) | CSRF tokens (enabled by default) |
| **Session Fixation** | New session on login |
| **Clickjacking** | X-Frame-Options header |
| **XSS** (Cross-Site Scripting) | Content-Security-Policy header |
| **Brute Force** | Rate limiting, account locking |
| **Man-in-the-Middle** | HTTPS enforcement (HSTS) |
| **Password Leaks** | BCrypt/Argon2 hashing |

### Spring Security vs Other Frameworks
| Feature | Spring Security | Apache Shiro | JAAS |
|---------|----------------|-------------|------|
| Complexity | Moderate-High | Low-Moderate | High |
| Spring Integration | Native | Manual | Manual |
| OAuth2/OIDC | Built-in | External | No |
| Community | Huge | Moderate | Small |
| Reactive Support | Yes | No | No |
| Customizability | Very High | High | Low |

### What Spring Security Does — Based on Application Type

Spring Security behaves differently depending on whether your application serves UI pages or only REST APIs:

**Web App (UI + API in same app — Stateful)**
- Manages the **login page** (default or custom HTML form).
- Stores authenticated user in **HTTP session** (server-side) → identified by `JSESSIONID` cookie.
- Enables **CSRF protection** (needed because browsers send cookies automatically with forms).
- On authentication failure → **redirects** to `/login`.
- On access denied → **redirects** to `/403` error page.
- Supports **Remember Me** (persistent login across browser restarts).

**REST API only (Stateless)**
- No login page — client sends credentials to a **JSON endpoint** (e.g., `POST /auth/login`) and receives a **JWT token**.
- **No session** — server does not remember anything. Every request must carry the token in the `Authorization: Bearer ...` header.
- **CSRF is disabled** (no browser forms → no CSRF risk).
- On authentication failure → returns **401 JSON response**.
- On access denied → returns **403 JSON response**.
- No redirects — everything is JSON in / JSON out.

| Aspect | Web App (UI + API) | API Only (REST) |
|--------|-------------------|-----------------|
| Login | HTML form → `POST /login` | JSON endpoint → returns JWT |
| Identity stored on | Server (HttpSession) | Client (JWT token) |
| Subsequent requests | Cookie: `JSESSIONID` | Header: `Authorization: Bearer ...` |
| CSRF | Enabled | Disabled |
| Session policy | `IF_REQUIRED` | `STATELESS` |
| On auth failure | Redirect to `/login` | Return `401 JSON` |
| On access denied | Redirect to `/403` page | Return `403 JSON` |
| Remember Me | Supported | Not applicable |

---

## 2. ARCHITECTURE & CORE COMPONENTS

### High-Level Architecture
```
Client Request
      │
      ▼
┌──────────────────────────────────────┐
│         DelegatingFilterProxy         │  ← Servlet Filter (bridge)
├──────────────────────────────────────┤
│       FilterChainProxy                │  ← Manages SecurityFilterChain(s)
├──────────────────────────────────────┤
│      SecurityFilterChain              │  ← Chain of security filters
│  ┌──────────────────────────────┐    │
│  │ DisableEncodeUrlFilter       │    │
│  │ SecurityContextHolderFilter  │    │
│  │ HeaderWriterFilter           │    │
│  │ CsrfFilter                  │    │
│  │ LogoutFilter                 │    │
│  │ UsernamePasswordAuthFilter   │    │
│  │ BasicAuthenticationFilter    │    │
│  │ AuthorizationFilter          │    │
│  │ ExceptionTranslationFilter   │    │
│  └──────────────────────────────┘    │
├──────────────────────────────────────┤
│         DispatcherServlet             │  ← Spring MVC
├──────────────────────────────────────┤
│         Controller / Endpoint         │
└──────────────────────────────────────┘
```

### Core Components

| Component | Role |
|-----------|------|
| **SecurityFilterChain** | Chain of filters applied to incoming requests |
| **AuthenticationManager** | Coordinates authentication — delegates to providers |
| **AuthenticationProvider** | Performs actual authentication (DB, LDAP, etc.) |
| **UserDetailsService** | Loads user data from a source (DB, in-memory) |
| **UserDetails** | Represents the authenticated user |
| **PasswordEncoder** | Hashes and verifies passwords |
| **SecurityContext** | Holds the current `Authentication` object |
| **SecurityContextHolder** | Thread-local storage for `SecurityContext` |
| **GrantedAuthority** | Represents a permission/role |
| **AccessDecisionManager** | Makes authorization decisions |

### Understanding the 3-Layer Architecture (Simple Explanation)

The diagram above shows 3 key layers. Here is what each one does and WHY it exists:

**Layer 1 — DelegatingFilterProxy (The Bridge)**

- The Servlet Container (e.g., Tomcat) only knows about **Servlet Filters** — it does NOT know about Spring beans.
- `DelegatingFilterProxy` IS a Servlet Filter that Tomcat can understand.
- Its only job: look up a Spring bean named `springSecurityFilterChain` and **delegate** every request to it.
- Think of it as a translator between the Tomcat world and the Spring world.

```
Tomcat (Servlet Container)
  │
  │ "I only know Servlet Filters"
  │
  ▼
DelegatingFilterProxy (Servlet Filter)
  │
  │ "Let me find the Spring bean and forward to it"
  │
  ▼
FilterChainProxy (Spring Bean)
  │
  │ "I manage all the security filters"
```

> **What does "registering in the Servlet Container" mean?**
> Tomcat maintains a list of filters. When a request comes in, Tomcat passes it through each filter in that list. "Registering" means adding `DelegatingFilterProxy` to this list so Tomcat calls it for every request. In Spring Boot, this happens automatically. In Spring MVC, you do it by extending `AbstractSecurityWebApplicationInitializer`.

**Layer 2 — FilterChainProxy (The Manager)**

- This is a **Spring-managed bean** (named `springSecurityFilterChain`).
- It holds **one or more** `SecurityFilterChain` objects.
- When a request comes in, it picks the **matching** `SecurityFilterChain` and runs its filters.
- Created automatically by `@EnableWebSecurity`.

**Layer 3 — SecurityFilterChain (The Actual Security)**

- This is what YOU configure using `HttpSecurity`.
- Contains an **ordered list of security filters** (CSRF, Auth, Authorization, etc.).
- Each filter has a specific job and a predefined order number.
- You can have **multiple chains** for different URL patterns (e.g., one for `/api/**`, one for web pages).

### Authentication Flow
```
1. User submits credentials (username + password)
              │
              ▼
2. UsernamePasswordAuthenticationFilter
   creates UsernamePasswordAuthenticationToken (unauthenticated)
              │
              ▼
3. AuthenticationManager.authenticate(token)
              │
              ▼
4. AuthenticationProvider (e.g., DaoAuthenticationProvider)
   ├── calls UserDetailsService.loadUserByUsername(username)
   ├── gets UserDetails from DB/memory
   ├── PasswordEncoder.matches(rawPassword, encodedPassword)
   └── returns authenticated Authentication token
              │
              ▼
5. SecurityContextHolder stores Authentication
              │
              ▼
6. Redirect to success URL / return response
```

### Understanding Key Components in the Flow

**UsernamePasswordAuthenticationToken — Two Versions**

This class has **two constructors** — this is important:

```java
// Constructor 1: UNAUTHENTICATED (before verification)
// Used when credentials are first submitted — "I claim to be admin with password secret"
new UsernamePasswordAuthenticationToken(username, password);
// → isAuthenticated() = false
// → No authorities set

// Constructor 2: AUTHENTICATED (after verification)
// Used after provider verifies credentials — "Yes, this is admin with these roles"
new UsernamePasswordAuthenticationToken(principal, credentials, authorities);
// → isAuthenticated() = true
// → Has GrantedAuthority list
```

The filter creates version 1 (unauthenticated). The provider returns version 2 (authenticated).

**DaoAuthenticationProvider — The Default Provider**

Spring Security has a built-in `AuthenticationProvider` called `DaoAuthenticationProvider`:

```
DaoAuthenticationProvider
  │
  ├── Uses: UserDetailsService  → to load user from DB/memory
  ├── Uses: PasswordEncoder      → to verify password
  ├── Supports: UsernamePasswordAuthenticationToken
  │
  └── Flow:
      1. Receives UsernamePasswordAuthenticationToken (unauthenticated)
      2. Calls userDetailsService.loadUserByUsername(username)
      3. Gets UserDetails object
      4. Calls passwordEncoder.matches(rawPassword, storedPassword)
      5. If match → returns authenticated token with authorities
      6. If no match → throws BadCredentialsException
```

> You do NOT need to create a custom `AuthenticationProvider` in most cases. Just provide a `UserDetailsService` bean and a `PasswordEncoder` bean — Spring auto-creates `DaoAuthenticationProvider` and wires them together.

**AuthenticationManager — Where Does It Come From?**

In Spring Boot, when you define `UserDetailsService` and `PasswordEncoder` beans, Spring auto-creates:
1. A `DaoAuthenticationProvider` (wired with your beans)
2. A `ProviderManager` (the default `AuthenticationManager` implementation)
3. Registers the provider inside the manager

To inject it in a controller (e.g., for JWT login endpoint):
```java
@Bean
public AuthenticationManager authManager(AuthenticationConfiguration config)
        throws Exception {
    return config.getAuthenticationManager();  // gets the auto-configured one
}
```

### Filter Chain Order (Default)
```
Filter                                  Order
──────────────────────────────────────  ─────
DisableEncodeUrlFilter                  100
ForceEagerSessionCreationFilter         200
ChannelProcessingFilter                 300
SecurityContextHolderFilter             400
HeaderWriterFilter                      500
CorsFilter                             600
CsrfFilter                             700
LogoutFilter                           800
OAuth2AuthorizationRequestRedirectFilter 900
UsernamePasswordAuthenticationFilter    1000
BasicAuthenticationFilter               1100
RequestCacheAwareFilter                 1200
SecurityContextHolderAwareRequestFilter 1300
AnonymousAuthenticationFilter           1400
SessionManagementFilter                 1500
ExceptionTranslationFilter              1600
AuthorizationFilter                     1700
```

---

## 3. PROJECT SETUP

### Maven Dependencies
```xml
<!-- Spring Boot Starter Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- For testing -->
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- For Thymeleaf integration (optional) -->
<dependency>
    <groupId>org.thymeleaf.extras</groupId>
    <artifactId>thymeleaf-extras-springsecurity6</artifactId>
</dependency>

<!-- For JWT (optional) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### Gradle Dependencies
```groovy
implementation 'org.springframework.boot:spring-boot-starter-security'
testImplementation 'org.springframework.security:spring-security-test'
```

---

## 4. DEFAULT SECURITY BEHAVIOR

### What Happens When You Add `spring-boot-starter-security`?
- **ALL endpoints** are secured (require authentication).
- A **default login form** is generated at `/login`.
- A **default logout** is available at `/logout`.
- **CSRF protection** is enabled.
- **Security headers** are added (X-Frame-Options, X-Content-Type-Options, etc.).
- A default user is created:
  - Username: `user`
  - Password: auto-generated (printed in console logs)
- **HTTP Basic** authentication is also enabled.
- **Session fixation** protection is enabled.

### Default Generated Password
```
Using generated security password: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

### Override Default User
```properties
# application.properties
spring.security.user.name=admin
spring.security.user.password=secret123
spring.security.user.roles=ADMIN
```

### What `@EnableWebSecurity` Does Behind the Scenes

When you create a security config class with `@EnableWebSecurity`:

```
@EnableWebSecurity internally:
       │
       ├── 1. Imports WebSecurityConfiguration class
       │
       ├── 2. Creates a FilterChainProxy bean (named "springSecurityFilterChain")
       │
       ├── 3. Collects ALL SecurityFilterChain @Bean methods from your config
       │
       ├── 4. Registers them inside FilterChainProxy
       │
       └── 5. Registers DelegatingFilterProxy in the Servlet Container
              (which delegates to FilterChainProxy)
```

> In Spring Boot, `@EnableWebSecurity` is auto-applied when you add `spring-boot-starter-security`. You don't need to write it explicitly (but it's a good practice to include it for clarity).

---

## 5. SECURITYFILTERCHAIN CONFIGURATION

### What is SecurityFilterChain?

`SecurityFilterChain` is a **bean** that holds an ordered list of security filters. When you write configuration code like `http.formLogin()` or `http.csrf()`, you are NOT just "setting options" — you are **registering actual Filter objects** into the chain.

```
Each http.xxx() method → registers one or more Filters:

http.formLogin()             → Adds UsernamePasswordAuthenticationFilter
http.httpBasic()             → Adds BasicAuthenticationFilter
http.csrf()                  → Adds CsrfFilter
http.logout()                → Adds LogoutFilter
http.sessionManagement()     → Adds SessionManagementFilter
http.authorizeHttpRequests() → Adds AuthorizationFilter
http.exceptionHandling()     → Adds ExceptionTranslationFilter
http.cors()                  → Adds CorsFilter
```

When you call `http.build()`, it collects all these filters, sorts them by predefined order numbers, and creates the `SecurityFilterChain`.

> **If you don't call a method, its filter is NOT added.** For example, if you don't call `http.formLogin()`, then `UsernamePasswordAuthenticationFilter` is not in the chain at all.

### Modern Approach (Spring Security 6.x / Spring Boot 3.x)
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/home", "/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").authenticated()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/login?logout")
                .permitAll()
            );

        return http.build();
    }
}
```

### Multiple SecurityFilterChains
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    // API security — evaluated first (lower order = higher priority)
    @Bean
    @Order(1)
    public SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .csrf(csrf -> csrf.disable());

        return http.build();
    }

    // Web security — evaluated second
    @Bean
    @Order(2)
    public SecurityFilterChain webFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/register").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());

        return http.build();
    }
}
```

### `securityMatcher` vs `requestMatchers`
| Method | Scope |
|--------|-------|
| `securityMatcher("/api/**")` | Determines **which requests** this SecurityFilterChain applies to |
| `requestMatchers("/admin/**")` | Within the chain, determines **authorization rules** for specific URLs |

---

## 6. AUTHENTICATION

### How Authentication Works — The Big Picture

Authentication in Spring Security has 3 key players that work together:

```
┌─────────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION FLOW                          │
│                                                                 │
│  Filter (extracts credentials from request)                     │
│     │                                                           │
│     ▼                                                           │
│  AuthenticationManager (coordinator — "who can handle this?")   │
│     │                                                           │
│     ▼                                                           │
│  AuthenticationProvider (verifier — "let me check credentials") │
│     │                                                           │
│     ├── UserDetailsService (loads user from DB)                 │
│     └── PasswordEncoder (verifies password)                     │
└─────────────────────────────────────────────────────────────────┘
```

**When do you need to create what?**

| Scenario | What You Create | What Spring Auto-Creates |
|----------|----------------|--------------------------|
| Standard username/password from DB | `UserDetailsService` + `PasswordEncoder` beans | `DaoAuthenticationProvider` + `ProviderManager` |
| Same as above but need custom logic | Custom `AuthenticationProvider` | `ProviderManager` |
| OTP / biometric / custom auth | Custom Token + Custom Provider + Custom Filter | Nothing — you wire everything |

> **Rule of thumb**: If your authentication is username + password verified against a DB, you only need `UserDetailsService` and `PasswordEncoder`. Spring handles the rest.

### AuthenticationManager
```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication)
        throws AuthenticationException;
}
```

### AuthenticationProvider
```java
public interface AuthenticationProvider {
    Authentication authenticate(Authentication authentication)
        throws AuthenticationException;

    boolean supports(Class<?> authentication);
}
```

### How AuthenticationManager & AuthenticationProvider Work Together

```
AuthenticationManager.authenticate(token)
       │
       │  (default impl = ProviderManager)
       │
       ▼
for each AuthenticationProvider:
  ├── provider.supports(token.getClass())  →  "Can you handle this token type?"
  │     NO  → skip to next provider
  │     YES → provider.authenticate(token) →  "Verify the credentials"
  │              SUCCESS → return authenticated token (stops here)
  │              FAIL    → save exception, try next provider
  │
  └── All failed → throw AuthenticationException
```

- `supports()` → decides **which** provider handles **which** token type
- `authenticate()` → does the **actual verification** (DB lookup, password match, etc.)
- `ProviderManager` (the default `AuthenticationManager`) loops through all registered providers and stops at the **first one that succeeds**

### Custom AuthenticationProvider
```java
@Component
public class CustomAuthProvider implements AuthenticationProvider {

    @Autowired
    private UserRepository userRepo;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public Authentication authenticate(Authentication authentication)
            throws AuthenticationException {

        String username = authentication.getName();
        String rawPassword = authentication.getCredentials().toString();

        User user = userRepo.findByUsername(username)
            .orElseThrow(() -> new BadCredentialsException("User not found"));

        if (!passwordEncoder.matches(rawPassword, user.getPassword())) {
            throw new BadCredentialsException("Invalid password");
        }

        List<GrantedAuthority> authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
            .collect(Collectors.toList());

        return new UsernamePasswordAuthenticationToken(username, null, authorities);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

### Authentication Object
```java
public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities(); // roles/permissions
    Object getCredentials();  // password (cleared after auth)
    Object getDetails();      // additional details (IP, session)
    Object getPrincipal();    // UserDetails or username
    boolean isAuthenticated();
}
```

---

## 7. USERDETAILSSERVICE & USERDETAILS

### UserDetailsService Interface
```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

### UserDetails Interface
```java
public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();
    String getUsername();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

### Custom UserDetails Implementation
```java
public class CustomUserDetails implements UserDetails {

    private final User user;  // your JPA entity

    public CustomUserDetails(User user) {
        this.user = user;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
            .collect(Collectors.toList());
    }

    @Override
    public String getPassword() { return user.getPassword(); }

    @Override
    public String getUsername() { return user.getUsername(); }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return !user.isLocked(); }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return user.isActive(); }

    // expose your user entity if needed
    public User getUser() { return user; }
}
```

### Custom UserDetailsService Implementation
```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() ->
                new UsernameNotFoundException("User not found: " + username));

        return new CustomUserDetails(user);
    }
}
```

---

## 8. PASSWORD ENCODING

### Why Encode Passwords?
- **Never store passwords in plain text** — breaches expose all users.
- Hashing = one-way transformation — can verify but can't reverse.
- Use **adaptive** algorithms that slow down brute-force attacks.

### Available Encoders
| Encoder | Algorithm | Recommended? |
|---------|-----------|-------------|
| `BCryptPasswordEncoder` | BCrypt | **Yes** (default choice) |
| `Argon2PasswordEncoder` | Argon2id | **Yes** (memory-hard) |
| `SCryptPasswordEncoder` | SCrypt | Yes (memory-hard) |
| `Pbkdf2PasswordEncoder` | PBKDF2 | Acceptable |
| `NoOpPasswordEncoder` | Plain text | **NEVER in production** |

### BCrypt Configuration
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // strength 4-31 (default 10)
}
```

### Argon2 Configuration
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new Argon2PasswordEncoder(
        16,    // salt length
        32,    // hash length
        1,     // parallelism
        65536, // memory in KB (64 MB)
        3      // iterations
    );
}
```

### DelegatingPasswordEncoder (Recommended for Migration)
```java
@Bean
public PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    // Stored as: {bcrypt}$2a$10$...
    // Supports multiple formats for migration
}
```

### Usage
```java
// Encode (during registration)
String encoded = passwordEncoder.encode("rawPassword");

// Verify (during login — done automatically by DaoAuthenticationProvider)
boolean matches = passwordEncoder.matches("rawPassword", encoded);
```

---

## 9. IN-MEMORY AUTHENTICATION

### Quick Setup (Testing/Prototyping Only)
```java
@Bean
public InMemoryUserDetailsManager userDetailsService() {
    UserDetails user = User.builder()
        .username("user")
        .password(passwordEncoder().encode("password"))
        .roles("USER")
        .build();

    UserDetails admin = User.builder()
        .username("admin")
        .password(passwordEncoder().encode("admin123"))
        .roles("ADMIN", "USER")
        .build();

    return new InMemoryUserDetailsManager(user, admin);
}
```

> **Warning**: In-memory auth is only for development/testing. Use database-backed auth in production.

---

## 10. JDBC AUTHENTICATION

### Default Schema (Spring Security expects these tables)
```sql
CREATE TABLE users (
    username VARCHAR(50) NOT NULL PRIMARY KEY,
    password VARCHAR(500) NOT NULL,
    enabled BOOLEAN NOT NULL
);

CREATE TABLE authorities (
    username VARCHAR(50) NOT NULL,
    authority VARCHAR(50) NOT NULL,
    CONSTRAINT fk_authorities_users FOREIGN KEY (username) REFERENCES users(username)
);
CREATE UNIQUE INDEX ix_auth_username ON authorities (username, authority);
```

### JDBC UserDetailsManager
```java
@Bean
public JdbcUserDetailsManager userDetailsService(DataSource dataSource) {
    JdbcUserDetailsManager manager = new JdbcUserDetailsManager(dataSource);

    // Optional: customize queries
    manager.setUsersByUsernameQuery(
        "SELECT username, password, enabled FROM app_users WHERE username = ?");
    manager.setAuthoritiesByUsernameQuery(
        "SELECT username, authority FROM app_authorities WHERE username = ?");

    return manager;
}
```

---

## 11. JPA / CUSTOM USERDETAILSSERVICE AUTHENTICATION

### User Entity
```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;

    private String email;
    private boolean active = true;
    private boolean locked = false;

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();

    // getters, setters
}
```

### Role Entity
```java
@Entity
@Table(name = "roles")
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String name;  // ADMIN, USER, MODERATOR

    // getters, setters
}
```

### Repository
```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    boolean existsByUsername(String username);
    boolean existsByEmail(String email);
}
```

### Registration Service
```java
@Service
@Transactional
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private RoleRepository roleRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    public User registerUser(RegisterRequest request) {
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new RuntimeException("Username already taken");
        }

        User user = new User();
        user.setUsername(request.getUsername());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        user.setEmail(request.getEmail());
        user.setActive(true);

        Role userRole = roleRepository.findByName("USER")
            .orElseThrow(() -> new RuntimeException("Default role not found"));
        user.setRoles(Set.of(userRole));

        return userRepository.save(user);
    }
}
```

### Complete Security Config with JPA
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private CustomUserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/register", "/login", "/css/**", "/js/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/login?error=true")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/login?logout")
                .deleteCookies("JSESSIONID")
                .permitAll()
            );

        return http.build();
    }

    @Bean
    public AuthenticationManager authManager(HttpSecurity http) throws Exception {
        AuthenticationManagerBuilder builder =
            http.getSharedObject(AuthenticationManagerBuilder.class);

        builder
            .userDetailsService(userDetailsService)
            .passwordEncoder(passwordEncoder());

        return builder.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## 12. AUTHORIZATION — URL-BASED

### Request Matchers & Authorization Rules
```java
http.authorizeHttpRequests(auth -> auth
    // Exact path
    .requestMatchers("/home").permitAll()

    // Ant patterns
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/api/users/**").hasAnyRole("ADMIN", "MANAGER")

    // HTTP method specific
    .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
    .requestMatchers(HttpMethod.POST, "/api/products/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.DELETE, "/api/**").hasRole("ADMIN")

    // Multiple patterns
    .requestMatchers("/css/**", "/js/**", "/images/**").permitAll()

    // Catch-all (MUST be last)
    .anyRequest().authenticated()
);
```

### Authorization Methods
| Method | Description |
|--------|-------------|
| `permitAll()` | No authentication required |
| `authenticated()` | Must be logged in |
| `denyAll()` | Block all requests |
| `hasRole("ADMIN")` | Must have ROLE_ADMIN |
| `hasAnyRole("ADMIN", "USER")` | Must have any listed role |
| `hasAuthority("READ_PRIVILEGE")` | Must have exact authority |
| `hasAnyAuthority("READ", "WRITE")` | Must have any listed authority |
| `hasIpAddress("192.168.1.0/24")` | Restrict by IP range |
| `access(AuthorizationManager)` | Custom authorization logic |

### Custom Authorization with SpEL (Spring Security 6.x)
```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/data/**")
        .access(new WebExpressionAuthorizationManager(
            "hasRole('ADMIN') and hasIpAddress('192.168.1.0/24')"))
    .anyRequest().authenticated()
);
```

---

## 13. AUTHORIZATION — METHOD-LEVEL

### Enable Method Security
```java
@Configuration
@EnableMethodSecurity  // replaces @EnableGlobalMethodSecurity
public class MethodSecurityConfig {
    // No additional config needed for defaults
}
```

### @PreAuthorize (Before method execution)
```java
@Service
public class UserService {

    @PreAuthorize("hasRole('ADMIN')")
    public List<User> getAllUsers() { ... }

    @PreAuthorize("hasRole('ADMIN') or #username == authentication.name")
    public User getUser(String username) { ... }

    @PreAuthorize("#user.username == authentication.name")
    public void updateProfile(User user) { ... }

    @PreAuthorize("hasAuthority('DELETE_USER')")
    public void deleteUser(Long id) { ... }

    @PreAuthorize("@securityService.isOwner(#resourceId, authentication.name)")
    public Resource getResource(Long resourceId) { ... }
}
```

### @PostAuthorize (After method execution — can filter return value)
```java
@PostAuthorize("returnObject.username == authentication.name or hasRole('ADMIN')")
public User getUserById(Long id) {
    return userRepository.findById(id).orElseThrow();
}
```

### @PreFilter / @PostFilter (Filter collections)
```java
// Filter input list — keep only items owned by current user
@PreFilter("filterObject.owner == authentication.name")
public void processTasks(List<Task> tasks) { ... }

// Filter output list — return only items current user can see
@PostFilter("filterObject.owner == authentication.name or hasRole('ADMIN')")
public List<Document> getDocuments() {
    return documentRepository.findAll();
}
```

### @Secured (Simpler — role-based only)
```java
@Secured("ROLE_ADMIN")
public void adminOnly() { ... }

@Secured({"ROLE_ADMIN", "ROLE_MANAGER"})
public void managementOnly() { ... }
```

### Custom Security Expression
```java
@Component("securityService")
public class SecurityService {

    @Autowired
    private ResourceRepository resourceRepository;

    public boolean isOwner(Long resourceId, String username) {
        return resourceRepository.findById(resourceId)
            .map(r -> r.getOwner().equals(username))
            .orElse(false);
    }
}

// Usage
@PreAuthorize("@securityService.isOwner(#id, authentication.name)")
public Resource getResource(Long id) { ... }
```

---

## 14. ROLES VS AUTHORITIES

### Key Differences
| Aspect | Role | Authority/Permission |
|--------|------|---------------------|
| Prefix | `ROLE_` prefix (auto-added) | No prefix |
| Granularity | Coarse (ADMIN, USER) | Fine (READ_USER, WRITE_USER) |
| Check method | `hasRole("ADMIN")` | `hasAuthority("READ_USER")` |
| Stored as | `ROLE_ADMIN` in GrantedAuthority | `READ_USER` in GrantedAuthority |
| Use case | Group-based access | Permission-based access |

### Role-Based Example
```java
// Assigning roles
List<GrantedAuthority> authorities = List.of(
    new SimpleGrantedAuthority("ROLE_ADMIN"),
    new SimpleGrantedAuthority("ROLE_USER")
);

// Checking
.requestMatchers("/admin/**").hasRole("ADMIN")  // checks for ROLE_ADMIN
```

### Permission-Based Example
```java
// Assigning authorities
List<GrantedAuthority> authorities = List.of(
    new SimpleGrantedAuthority("READ_USER"),
    new SimpleGrantedAuthority("WRITE_USER"),
    new SimpleGrantedAuthority("DELETE_USER")
);

// Checking
@PreAuthorize("hasAuthority('DELETE_USER')")
public void deleteUser(Long id) { ... }
```

### Hierarchical Roles
```java
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.withDefaultRolePrefix()
        .role("ADMIN").implies("MANAGER")
        .role("MANAGER").implies("USER")
        .role("USER").implies("GUEST")
        .build();
}
```

---

## 15. FORM-BASED LOGIN

### What Happens Internally When You Use `formLogin()`

When you call `http.formLogin()`, Spring Security adds `UsernamePasswordAuthenticationFilter` to the chain. Here is the complete internal flow:

```
User submits login form → POST /login
       │
       ▼
UsernamePasswordAuthenticationFilter
  │ 1. Extracts username & password from request parameters
  │ 2. Creates UsernamePasswordAuthenticationToken (unauthenticated)
  │ 3. Calls AuthenticationManager.authenticate(token)
  │         │
  │         ▼
  │    DaoAuthenticationProvider
  │      → UserDetailsService.loadUserByUsername()
  │      → PasswordEncoder.matches()
  │      → Returns authenticated token (with authorities)
  │
  ├── SUCCESS:
  │   → SecurityContextHolder.setAuthentication(authenticatedToken)
  │   → SecurityContext saved into HttpSession
  │     (session.setAttribute("SPRING_SECURITY_CONTEXT", context))
  │   → Redirect to defaultSuccessUrl (e.g., /dashboard)
  │
  └── FAILURE:
      → Redirect to failureUrl (e.g., /login?error)
```

After login, on subsequent requests:
```
GET /dashboard (Cookie: JSESSIONID=abc123)
       │
       ▼
SecurityContextHolderFilter loads SecurityContext from session
       → User is automatically recognized as authenticated
       → No re-login needed
```

> You do NOT write code to save/load from session. Spring Security handles this internally through `SecurityContextHolderFilter` and `HttpSessionSecurityContextRepository`.

### Custom Login Configuration
```java
http.formLogin(form -> form
    .loginPage("/login")                    // custom login page URL
    .loginProcessingUrl("/perform-login")    // form action URL
    .usernameParameter("email")              // custom form field name
    .passwordParameter("pass")               // custom form field name
    .defaultSuccessUrl("/dashboard", true)   // redirect after success
    .failureUrl("/login?error=true")         // redirect after failure
    .successHandler(customSuccessHandler())  // custom success logic
    .failureHandler(customFailureHandler())  // custom failure logic
    .permitAll()
);
```

### Custom Success Handler
```java
@Component
public class CustomSuccessHandler implements AuthenticationSuccessHandler {

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
            HttpServletResponse response, Authentication authentication)
            throws IOException, ServletException {

        String role = authentication.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .findFirst().orElse("ROLE_USER");

        if (role.equals("ROLE_ADMIN")) {
            response.sendRedirect("/admin/dashboard");
        } else {
            response.sendRedirect("/user/dashboard");
        }
    }
}
```

### Custom Failure Handler
```java
@Component
public class CustomFailureHandler implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,
            HttpServletResponse response, AuthenticationException exception)
            throws IOException, ServletException {

        String errorMessage;
        if (exception instanceof BadCredentialsException) {
            errorMessage = "Invalid username or password";
        } else if (exception instanceof LockedException) {
            errorMessage = "Account is locked";
        } else if (exception instanceof DisabledException) {
            errorMessage = "Account is disabled";
        } else {
            errorMessage = "Authentication failed";
        }

        request.getSession().setAttribute("error", errorMessage);
        response.sendRedirect("/login?error");
    }
}
```

### Login Page (Thymeleaf Example)
```html
<form th:action="@{/login}" method="post">
    <div th:if="${param.error}">
        <p style="color:red;">Invalid credentials</p>
    </div>
    <div th:if="${param.logout}">
        <p style="color:green;">Logged out successfully</p>
    </div>

    <label>Username:</label>
    <input type="text" name="username" required />

    <label>Password:</label>
    <input type="password" name="password" required />

    <button type="submit">Login</button>
</form>
```

---

## 16. HTTP BASIC AUTHENTICATION

### Configuration
```java
http.httpBasic(basic -> basic
    .realmName("MyApp API")
    .authenticationEntryPoint(new CustomBasicAuthEntryPoint())
);
```

### Custom Entry Point
```java
@Component
public class CustomBasicAuthEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException) throws IOException {

        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json");
        response.getWriter().write("""
            {"error": "Unauthorized", "message": "Authentication required"}
            """);
    }
}
```

> **Note**: HTTP Basic sends credentials as Base64 encoded (NOT encrypted). Always use with HTTPS.

---

## 17. REMEMBER ME

### Cookie-Based (Simple)
```java
http.rememberMe(remember -> remember
    .key("uniqueAndSecretKey")       // used to sign tokens
    .tokenValiditySeconds(86400 * 7) // 7 days
    .rememberMeParameter("remember") // form checkbox name
    .userDetailsService(userDetailsService)
);
```

### Persistent Token (Database — More Secure)
```java
http.rememberMe(remember -> remember
    .tokenRepository(tokenRepository())
    .tokenValiditySeconds(86400 * 30) // 30 days
    .userDetailsService(userDetailsService)
);

@Bean
public PersistentTokenRepository tokenRepository() {
    JdbcTokenRepositoryImpl repo = new JdbcTokenRepositoryImpl();
    repo.setDataSource(dataSource);
    // repo.setCreateTableOnStartup(true); // first run only
    return repo;
}
```

### Required Table (Persistent Token)
```sql
CREATE TABLE persistent_logins (
    username  VARCHAR(64) NOT NULL,
    series    VARCHAR(64) PRIMARY KEY,
    token     VARCHAR(64) NOT NULL,
    last_used TIMESTAMP NOT NULL
);
```

---

## 18. LOGOUT

### Configuration
```java
http.logout(logout -> logout
    .logoutUrl("/logout")                        // logout trigger URL
    .logoutRequestMatcher(new AntPathRequestMatcher("/logout", "GET"))
    .logoutSuccessUrl("/login?logout")           // redirect after logout
    .logoutSuccessHandler(customLogoutHandler())  // custom handler
    .invalidateHttpSession(true)                  // destroy session
    .clearAuthentication(true)                    // clear auth
    .deleteCookies("JSESSIONID", "remember-me")  // remove cookies
    .addLogoutHandler(customLogoutHandler)         // additional cleanup
    .permitAll()
);
```

### Custom Logout Handler
```java
@Component
public class CustomLogoutHandler implements LogoutHandler {

    @Override
    public void logout(HttpServletRequest request,
            HttpServletResponse response, Authentication authentication) {
        // Custom cleanup — audit logging, token revocation, etc.
        if (authentication != null) {
            String username = authentication.getName();
            // log audit event
        }
    }
}
```

---

## 19. SESSION MANAGEMENT

### Configuration
```java
http.sessionManagement(session -> session
    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
    .maximumSessions(1)                        // max concurrent sessions
    .maxSessionsPreventsLogin(true)            // block new login (vs expire old)
    .expiredUrl("/login?expired")              // redirect when session expires
    .sessionFixation().migrateSession()        // protect against session fixation
    .invalidSessionUrl("/login?invalid")
);
```

### Session Creation Policies
| Policy | Description | Use When |
|--------|------------|----------|
| `ALWAYS` | Always create a session | Rarely needed |
| `IF_REQUIRED` | Create session only when needed (default) | Web apps with form login |
| `NEVER` | Never create — but use if one exists | Transitional |
| `STATELESS` | Never create or use session | **REST APIs / JWT** |

> **Important**: When you set `STATELESS`, Spring Security will NOT create `HttpSession`, will NOT save `SecurityContext` to session, and will NOT use `JSESSIONID` cookie. Every request must carry its own credentials (e.g., JWT token in `Authorization` header).

### Concurrent Session Control
```java
// Also register this listener for session tracking
@Bean
public HttpSessionEventPublisher httpSessionEventPublisher() {
    return new HttpSessionEventPublisher();
}
```

---

## 20. CSRF PROTECTION

### What is CSRF?
- **Cross-Site Request Forgery** — an attacker tricks a user's browser into making unwanted requests.
- Example: User is logged into bank → visits malicious site → hidden form submits transfer request.
- Spring Security prevents this with **CSRF tokens**.

### Default Behavior
- CSRF protection is **enabled by default**.
- A unique token is generated per session.
- All state-changing requests (POST, PUT, DELETE) must include the token.

### CSRF in Forms (Thymeleaf — automatic)
```html
<!-- Thymeleaf adds CSRF token automatically -->
<form th:action="@{/transfer}" method="post">
    <!-- hidden CSRF input added automatically -->
    <input type="text" name="amount" />
    <button type="submit">Transfer</button>
</form>
```

### CSRF in AJAX (JavaScript)
```html
<!-- Include meta tags -->
<meta name="_csrf" th:content="${_csrf.token}" />
<meta name="_csrf_header" th:content="${_csrf.headerName}" />

<script>
const token = document.querySelector('meta[name="_csrf"]').content;
const header = document.querySelector('meta[name="_csrf_header"]').content;

fetch('/api/data', {
    method: 'POST',
    headers: {
        [header]: token,
        'Content-Type': 'application/json'
    },
    body: JSON.stringify(data)
});
</script>
```

### Disable CSRF (For Stateless APIs Only)
```java
http.csrf(csrf -> csrf.disable());
```

### Cookie-Based CSRF (For SPA / Angular / React)
```java
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
);
// Angular automatically reads XSRF-TOKEN cookie and sends X-XSRF-TOKEN header
```

### Ignore CSRF for Specific Paths
```java
http.csrf(csrf -> csrf
    .ignoringRequestMatchers("/api/webhooks/**", "/api/public/**")
);
```

---

## 21. CORS CONFIGURATION

### Global CORS Configuration
```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://frontend.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}

// Enable in SecurityFilterChain
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

### Controller-Level CORS
```java
@RestController
@CrossOrigin(origins = "https://frontend.example.com", maxAge = 3600)
public class ApiController {

    @CrossOrigin(origins = "https://other.example.com")
    @GetMapping("/data")
    public ResponseEntity<String> getData() { ... }
}
```

---

## 22. EXCEPTION HANDLING

### AuthenticationEntryPoint (401 — Unauthenticated)
```java
@Component
public class CustomAuthEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException) throws IOException {

        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json");
        response.getWriter().write("""
            {
                "status": 401,
                "error": "Unauthorized",
                "message": "%s",
                "path": "%s"
            }
            """.formatted(authException.getMessage(), request.getRequestURI()));
    }
}
```

### AccessDeniedHandler (403 — Forbidden)
```java
@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException) throws IOException {

        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.setContentType("application/json");
        response.getWriter().write("""
            {
                "status": 403,
                "error": "Forbidden",
                "message": "You don't have permission to access this resource",
                "path": "%s"
            }
            """.formatted(request.getRequestURI()));
    }
}
```

### Register in SecurityFilterChain
```java
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint(customAuthEntryPoint)
    .accessDeniedHandler(customAccessDeniedHandler)
    .accessDeniedPage("/error/403")  // for web apps (alternative to handler)
);
```

---

## 23. CUSTOM FILTERS

### When and Why to Create Custom Filters

Spring Security provides many built-in filters, but sometimes you need your own:

| Use Case | Custom Filter Type |
|----------|-------------------|
| JWT token validation | Extends `OncePerRequestFilter` |
| Request logging / auditing | Extends `OncePerRequestFilter` |
| Custom form-based authentication (e.g., OTP) | Extends `AbstractAuthenticationProcessingFilter` |
| API key validation | Extends `OncePerRequestFilter` |
| Rate limiting | Extends `OncePerRequestFilter` |

### Two Types of Custom Filters

**`OncePerRequestFilter`** — Use for filters that should run on **every request** (e.g., JWT validation, logging):
- You control the logic: validate something, set authentication, or just pass through
- You call `filterChain.doFilter(request, response)` to continue the chain
- Guaranteed to run only once per request (even if request is forwarded internally)

**`AbstractAuthenticationProcessingFilter`** — Use for filters that handle **authentication at a specific URL** (e.g., form login, OTP login):
- Automatically checks if the request matches a specific URL pattern
- Has built-in success/failure handling
- Works with `AuthenticationManager` to delegate authentication
- Example: `UsernamePasswordAuthenticationFilter` extends this

### Creating a Custom Filter (OncePerRequestFilter)
```java
public class RequestLoggingFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        long startTime = System.currentTimeMillis();

        log.info("Request: {} {} from {}",
            request.getMethod(), request.getRequestURI(), request.getRemoteAddr());

        filterChain.doFilter(request, response);  // continue chain

        long duration = System.currentTimeMillis() - startTime;
        log.info("Response: {} completed in {} ms",
            response.getStatus(), duration);
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        return request.getRequestURI().startsWith("/static/");
    }
}
```

### Registering Custom Filters
```java
http.addFilterBefore(new RequestLoggingFilter(),
    UsernamePasswordAuthenticationFilter.class);

http.addFilterAfter(new AuditFilter(),
    AuthorizationFilter.class);

http.addFilterAt(new CustomAuthFilter(),
    UsernamePasswordAuthenticationFilter.class);
```

### Filter Ordering Methods
| Method | Description |
|--------|------------|
| `addFilterBefore(filter, class)` | Add before the specified filter |
| `addFilterAfter(filter, class)` | Add after the specified filter |
| `addFilterAt(filter, class)` | Replace the specified filter |

---

## 24. JWT AUTHENTICATION

### Why JWT for REST APIs?

In a traditional web app, the server keeps a **session** (stateful). But REST APIs should be **stateless** — the server should NOT remember anything between requests. JWT solves this:

```
Stateful (Form Login):           Stateless (JWT):
  Server stores session            Client stores token
  Server remembers user            Server forgets after response
  Cookie: JSESSIONID               Header: Authorization: Bearer ...
  Good for: web apps with pages    Good for: REST APIs, SPAs, mobile
```

### Why REST APIs Don't Use UsernamePasswordAuthenticationFilter

`UsernamePasswordAuthenticationFilter` is designed for **HTML form-based login**:
- It expects `POST /login` with `username` and `password` form parameters
- It redirects to a success/failure URL after login
- It creates and manages an HTTP session

**None of this makes sense for a REST API** which:
- Receives JSON, not form data
- Returns JSON responses, not redirects
- Is stateless — no sessions

Instead, REST APIs use:
1. **A login controller** (`POST /auth/login`) — validates credentials and returns a JWT token
2. **A JWT filter** — validates the token on every subsequent request

```
REST API Authentication (JWT):

  Step 1: Login (ONE time)
  ┌──────────────────────────────────────────────────┐
  │ POST /auth/login                                  │
  │ Body: { "username": "admin", "password": "secret"}│
  │                                                    │
  │ AuthController calls:                              │
  │   authenticationManager.authenticate(token)        │
  │   → Provider validates credentials                 │
  │   → Generate JWT: jwtService.generateToken()       │
  │   → Return: { "token": "eyJhbGciOi..." }          │
  │                                                    │
  │ NO session, NO redirect, NO SecurityContext stored  │
  └──────────────────────────────────────────────────┘

  Step 2: Every subsequent request
  ┌──────────────────────────────────────────────────┐
  │ GET /api/products                                  │
  │ Header: Authorization: Bearer eyJhbGciOi...        │
  │                                                    │
  │ JwtAuthFilter:                                     │
  │   → Extract token from header                      │
  │   → Validate token (signature, expiry)             │
  │   → Extract username from claims                   │
  │   → Load UserDetails from DB                       │
  │   → Set SecurityContextHolder (for THIS request)   │
  │                                                    │
  │ AuthorizationFilter:                               │
  │   → Check roles from SecurityContextHolder         │
  │   → Allow or deny                                  │
  │                                                    │
  │ After response: SecurityContextHolder cleared       │
  └──────────────────────────────────────────────────┘
```

### Complete Filter-Level Flow (Every Step Explained)

> The diagram above shows the high-level flow. Below is the **complete filter-by-filter flow** showing exactly what happens inside each filter.

#### Login Request — Full Filter Flow

```
POST /auth/login { "username": "admin", "password": "secret" }
│
├── FILTER 1: SecurityContextHolderFilter (Order 400)
│   → No session exists (STATELESS policy)
│   → Creates EMPTY SecurityContext
│   → Puts it in ThreadLocal
│   → SecurityContextHolder.getContext().getAuthentication() == null
│
├── FILTER 2: JwtAuthFilter (your custom filter)
│   → shouldNotFilter() → path is /auth/** → SKIP
│   → Does nothing, calls filterChain.doFilter()
│
├── FILTER 3: AuthorizationFilter (Order 1700)
│   → Checks: Is /auth/login permitAll()? → YES → ALLOWED ✅
│
├── DISPATCHER SERVLET → AuthController.login()
│   │
│   ├── authenticationManager.authenticate(
│   │       new UsernamePasswordAuthenticationToken("admin", "secret"))
│   │   │
│   │   └── DaoAuthenticationProvider:
│   │       ├── UserDetailsService.loadUserByUsername("admin") → loads from DB
│   │       ├── PasswordEncoder.matches("secret", "$2a$10$...") → MATCH ✅
│   │       └── Returns authenticated token (with authorities)
│   │
│   ├── jwtService.generateToken(userDetails) → "eyJhbGciOi..."
│   │       ← TOKEN IS GENERATED HERE, ONLY HERE, ONLY ONCE
│   │
│   └── return ResponseEntity.ok(new AuthResponse("eyJhbGciOi..."))
│
├── CLEANUP:
│   → SecurityContextHolder.clearContext() → ThreadLocal emptied
│   → No session saved (STATELESS)
│
└── Response: { "token": "eyJhbGciOi..." }
    Client stores this token (localStorage / cookie)
```

#### Subsequent Request — Full Filter Flow

```
GET /api/products
Header: Authorization: Bearer eyJhbGciOi...
│
│
├── FILTER 1: SecurityContextHolderFilter (Order 400)
│   │
│   │  → Calls securityContextRepository.loadContext()
│   │  → STATELESS → uses RequestAttributeSecurityContextRepository
│   │  → No session to load from → returns EMPTY SecurityContext
│   │  → Puts EMPTY context in ThreadLocal
│   │
│   │  At this point:
│   │  SecurityContextHolder.getContext().getAuthentication() == null
│   │  (Nobody is authenticated yet)
│   │
│   └── Calls filterChain.doFilter() → moves to next filter
│
│
├── FILTER 2: JwtAuthFilter (your custom filter, before Order 1000)
│   │
│   │  Step 1: Read header
│   │  → authHeader = "Bearer eyJhbGciOi..."
│   │  → Starts with "Bearer " → proceed
│   │
│   │  Step 2: Extract token
│   │  → token = "eyJhbGciOi..."
│   │
│   │  Step 3: Extract username from token
│   │  → jwtService.extractUsername(token) → parses JWT → username = "admin"
│   │
│   │  Step 4: Check if already authenticated
│   │  → SecurityContextHolder.getContext().getAuthentication() == null → proceed
│   │
│   │  Step 5: Load user from DB
│   │  → userDetailsService.loadUserByUsername("admin")
│   │  → Returns UserDetails with roles [ROLE_ADMIN, ROLE_USER]
│   │
│   │  Step 6: Validate token
│   │  → jwtService.isTokenValid(token, userDetails)
│   │    ├── Username in token matches UserDetails? ✅
│   │    └── Token not expired? ✅
│   │
│   │  Step 7: Create AUTHENTICATED token
│   │  → new UsernamePasswordAuthenticationToken(
│   │        userDetails, null, userDetails.getAuthorities())
│   │  → isAuthenticated() = true
│   │
│   │  Step 8: SET in SecurityContextHolder
│   │  → SecurityContextHolder.getContext().setAuthentication(authToken)
│   │
│   │  NOW SecurityContextHolder has:
│   │  ┌────────────────────────────────────────┐
│   │  │ SecurityContextHolder (ThreadLocal)     │
│   │  │  └── SecurityContext                    │
│   │  │       └── Authentication                │
│   │  │            ├── principal: UserDetails    │
│   │  │            ├── authorities: [ROLE_ADMIN] │
│   │  │            └── isAuthenticated: true     │
│   │  └────────────────────────────────────────┘
│   │
│   └── Calls filterChain.doFilter() → moves to next filter
│
│
├── FILTER 3: AuthorizationFilter (Order 1700)
│   │
│   │  Step 1: Read auth from SecurityContextHolder
│   │  → auth is NOT null, isAuthenticated() = true
│   │
│   │  Step 2: Match URL to rule
│   │  → /api/products → anyRequest().authenticated()
│   │
│   │  Step 3: Evaluate
│   │  → Is user authenticated? → YES → ALLOWED ✅
│   │
│   │  (If rule was .hasRole("ADMIN")):
│   │  → Does user have ROLE_ADMIN? → YES → ALLOWED ✅
│   │
│   │  (If rule was .hasRole("SUPERADMIN")):
│   │  → Does user have ROLE_SUPERADMIN? → NO ❌
│   │  → Throws AccessDeniedException → ExceptionTranslationFilter → 403 JSON
│   │
│   └── ALLOWED → goes to DispatcherServlet
│
│
├── DISPATCHER SERVLET → ProductController.getProducts()
│   │
│   │  Controller can access the authenticated user:
│   │  → @AuthenticationPrincipal UserDetails user
│   │  → SecurityContextHolder.getContext().getAuthentication()
│   │
│   └── Returns ResponseEntity with product data
│
│
├── CLEANUP (on the way back):
│   │
│   │  SecurityContextHolderFilter:
│   │  → STATELESS policy → does NOT save to HttpSession
│   │  → SecurityContextHolder.clearContext()
│   │  → ThreadLocal is NOW EMPTY
│   │  → Server remembers NOTHING
│   │
│   └── Response sent to client
│
└── Done. Next request starts from scratch.
```

> **Key Point**: JwtAuthFilter does NOT generate tokens — it only **validates** the token sent by the client. Token generation happens ONLY in `AuthController.login()`. The filter runs on every request, validates the JWT, and sets the `SecurityContext` for that single request.

### Role of SecurityContextHolder in JWT (Stateless) Flow

> "If the app is stateless and nothing is saved, why do we need SecurityContextHolder at all?"

`SecurityContextHolder` is a **temporary shared box** within a single request. It allows different filters and components to **pass authentication data to each other** — they don't call each other directly.

```
Within ONE request:

JwtAuthFilter          → WRITES to SecurityContextHolder  (sets Authentication)
       │
       │  "I validated the token. Here's who this user is."
       │
       ▼
AuthorizationFilter    → READS from SecurityContextHolder  (gets authorities)
       │
       │  "Let me check if this user has the required role."
       │
       ▼
Controller             → READS from SecurityContextHolder  (@AuthenticationPrincipal)
       │
       │  "Let me get the current user's name."
       │
       ▼
@PreAuthorize          → READS from SecurityContextHolder  (checks hasRole())
       │
       ▼
CLEANUP                → CLEARS SecurityContextHolder      (request done, forget everything)
```

Without it, `JwtAuthFilter` has no way to tell `AuthorizationFilter` "this user is admin with ROLE_ADMIN" — they are separate classes. `SecurityContextHolder` (ThreadLocal) is the **communication channel** between them within the same request thread.

### Stateless Means Every Request Starts From Scratch

```
┌─────────────────────────────────────────────────────────────────────┐
│                    STATELESS = NO MEMORY                            │
│                                                                     │
│  Request 1        Request 2        Request 3        Request 4       │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐     │
│  │ Validate │    │ Validate │    │ Validate │    │ Validate │     │
│  │ JWT      │    │ JWT      │    │ JWT      │    │ JWT      │     │
│  │ Set auth │    │ Set auth │    │ Set auth │    │ Set auth │     │
│  │ Authorize│    │ Authorize│    │ Authorize│    │ Authorize│     │
│  │ Handle   │    │ Handle   │    │ Handle   │    │ Handle   │     │
│  │ Clear ⌫  │    │ Clear ⌫  │    │ Clear ⌫  │    │ Clear ⌫  │     │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘     │
│  Forgotten        Forgotten        Forgotten        Forgotten       │
│                                                                     │
│  SecurityContextHolder lives ONLY within each box.                  │
│  Nothing carries over. Token on CLIENT is the only identity proof.  │
└─────────────────────────────────────────────────────────────────────┘
```

### How JWT Works
```
1. Client sends credentials (POST /login)
              │
              ▼
2. Server validates credentials
   Server generates JWT token (signed with secret)
   Returns token to client
              │
              ▼
3. Client stores token (localStorage / httpOnly cookie)
              │
              ▼
4. Client sends token in every request:
   Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
              │
              ▼
5. Server validates token → extracts user info → authenticates
```

### JWT Structure
```
Header.Payload.Signature

Header:  {"alg": "HS256", "typ": "JWT"}
Payload: {"sub": "admin", "roles": ["ADMIN"], "iat": 1700000000, "exp": 1700003600}
Signature: HMACSHA256(base64(header) + "." + base64(payload), secretKey)
```

### Dependencies
```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

### JWT Utility Service
```java
@Service
public class JwtService {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.expiration:3600000}")  // 1 hour default
    private long expiration;

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    public String generateToken(UserDetails userDetails) {
        return generateToken(Map.of(), userDetails);
    }

    public String generateToken(Map<String, Object> extraClaims,
                                 UserDetails userDetails) {
        return Jwts.builder()
            .claims(extraClaims)
            .subject(userDetails.getUsername())
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(getSigningKey())
            .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public <T> T extractClaim(String token, Function<Claims, T> resolver) {
        Claims claims = Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
        return resolver.apply(claims);
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }
}
```

### JWT Authentication Filter
```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Autowired
    private JwtService jwtService;

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7);
        String username = jwtService.extractUsername(token);

        if (username != null &&
                SecurityContextHolder.getContext().getAuthentication() == null) {

            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            if (jwtService.isTokenValid(token, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }
}
```

### JWT Security Configuration

> Each line in this config has a specific reason. Read the comments carefully.

```java
@Configuration
@EnableWebSecurity
public class JwtSecurityConfig {

    @Autowired
    private JwtAuthFilter jwtAuthFilter;

    @Autowired
    private CustomUserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // 1. Disable CSRF — REST APIs are stateless, no session = no CSRF risk
            //    (CSRF attacks exploit session cookies, which we don't use)
            .csrf(csrf -> csrf.disable())

            // 2. STATELESS — don't create HttpSession, don't store SecurityContext
            //    Every request must carry its own credentials (JWT token)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // 3. Define which URLs are public and which need authentication
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()  // login/register = public
                .anyRequest().authenticated()             // everything else = protected
            )

            // 4. Tell Spring to use our DaoAuthenticationProvider
            //    (for validating username/password during login)
            .authenticationProvider(authenticationProvider())

            // 5. Add our JWT filter BEFORE UsernamePasswordAuthenticationFilter
            //    So JWT validation happens first on every request
            .addFilterBefore(jwtAuthFilter,
                UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    // DaoAuthenticationProvider — used during login to verify username/password
    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    // Expose AuthenticationManager bean — needed in AuthController for login
    @Bean
    public AuthenticationManager authManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Auth Controller
```java
@RestController
@RequestMapping("/auth")
public class AuthController {

    @Autowired
    private AuthenticationManager authManager;

    @Autowired
    private JwtService jwtService;

    @Autowired
    private UserDetailsService userDetailsService;

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
        authManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(), request.getPassword()));

        UserDetails userDetails =
            userDetailsService.loadUserByUsername(request.getUsername());
        String token = jwtService.generateToken(userDetails);

        return ResponseEntity.ok(new AuthResponse(token));
    }

    @PostMapping("/register")
    public ResponseEntity<String> register(@RequestBody RegisterRequest request) {
        // registration logic
        return ResponseEntity.ok("User registered successfully");
    }
}
```

### JWT Refresh Token Flow
```java
@Service
public class RefreshTokenService {

    @Value("${jwt.refresh-expiration:604800000}")  // 7 days
    private long refreshExpiration;

    @Autowired
    private RefreshTokenRepository refreshTokenRepo;

    @Autowired
    private UserRepository userRepo;

    public RefreshToken createRefreshToken(String username) {
        RefreshToken token = new RefreshToken();
        token.setUser(userRepo.findByUsername(username).orElseThrow());
        token.setToken(UUID.randomUUID().toString());
        token.setExpiryDate(Instant.now().plusMillis(refreshExpiration));
        return refreshTokenRepo.save(token);
    }

    public RefreshToken verifyExpiration(RefreshToken token) {
        if (token.getExpiryDate().isBefore(Instant.now())) {
            refreshTokenRepo.delete(token);
            throw new RuntimeException("Refresh token expired. Please login again.");
        }
        return token;
    }
}
```

---

## 25. SECURITY CONTEXT & SECURITYCONTEXTHOLDER

### What is SecurityContextHolder?

`SecurityContextHolder` is the **central place** where Spring Security stores the details of who is currently authenticated. Every part of Spring Security reads from it.

```
SecurityContextHolder (static class, uses ThreadLocal)
       │
       └── SecurityContext
              │
              └── Authentication
                    ├── getPrincipal()    → WHO (UserDetails / username)
                    ├── getCredentials()  → PROOF (password — cleared after auth)
                    ├── getAuthorities()  → PERMISSIONS (ROLE_ADMIN, etc.)
                    └── isAuthenticated() → true/false
```

### Critical Concept: ThreadLocal = Per-Request

`SecurityContextHolder` stores the `SecurityContext` in a **ThreadLocal** variable. This means:
- It is **per-thread** → each request runs on its own thread → each request has its own `SecurityContext`
- It does **NOT survive across requests** by itself
- It is **cleared at the end of every request**

So how does the server "remember" a logged-in user? That depends on stateful vs stateless:

```
┌─────────────────────────────────────────────────────────────┐
│ STATEFUL (Form Login):                                       │
│   SecurityContext → saved into HttpSession → survives        │
│   Next request → loaded from session → put back in ThreadLocal│
│   Who does this? SecurityContextHolderFilter automatically    │
│                                                               │
│ STATELESS (JWT):                                              │
│   SecurityContext → NOT saved anywhere on server              │
│   Next request → JWT filter validates token → creates new     │
│   SecurityContext → puts in ThreadLocal for this request only  │
└─────────────────────────────────────────────────────────────┘
```

### Who Loads/Saves SecurityContext Automatically?

| Version | Filter | What It Does |
|---------|--------|-------------|
| Spring Security 5.7+ | `SecurityContextHolderFilter` | Loads context at start, saves at end |
| Before 5.7 | `SecurityContextPersistenceFilter` | Same job, older implementation |

These filters use `SecurityContextRepository` (interface) under the hood:

```
SecurityContextHolderFilter
       │ uses
       ▼
SecurityContextRepository (interface)
       │
       ├── HttpSessionSecurityContextRepository (default for stateful)
       │   ├── loadContext()  → session.getAttribute("SPRING_SECURITY_CONTEXT")
       │   └── saveContext()  → session.setAttribute("SPRING_SECURITY_CONTEXT", ctx)
       │
       └── RequestAttributeSecurityContextRepository (for stateless)
           └── No session involved — context lives only for the request
```

### Getting Current User
```java
// Option 1: SecurityContextHolder (anywhere)
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();

// Option 2: @AuthenticationPrincipal (in controller)
@GetMapping("/profile")
public String profile(@AuthenticationPrincipal UserDetails userDetails) {
    return "Welcome " + userDetails.getUsername();
}

// Option 3: @AuthenticationPrincipal with custom UserDetails
@GetMapping("/profile")
public String profile(@AuthenticationPrincipal CustomUserDetails user) {
    return "Welcome " + user.getUser().getFullName();
}

// Option 4: Principal parameter
@GetMapping("/profile")
public String profile(Principal principal) {
    return "Welcome " + principal.getName();
}

// Option 5: Authentication parameter
@GetMapping("/profile")
public String profile(Authentication authentication) {
    return "Welcome " + authentication.getName();
}
```

### SecurityContext Strategies
| Strategy | Description |
|----------|------------|
| `MODE_THREADLOCAL` | Default — context per thread |
| `MODE_INHERITABLETHREADLOCAL` | Parent thread context passed to child threads |
| `MODE_GLOBAL` | Shared across all threads (standalone apps) |

```java
// Set strategy (at app startup)
SecurityContextHolder.setStrategyName(
    SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
```

### Async Security Context Propagation
```java
// For @Async methods — propagate security context
@Bean
public DelegatingSecurityContextAsyncTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);
    executor.initialize();
    return new DelegatingSecurityContextAsyncTaskExecutor(executor);
}
```

---

## 26. LDAP AUTHENTICATION

### Dependencies
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ldap</groupId>
    <artifactId>spring-ldap-core</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-ldap</artifactId>
</dependency>
```

### Configuration
```java
@Configuration
@EnableWebSecurity
public class LdapSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    public LdapAuthenticationProvider ldapAuthProvider() {
        LdapAuthenticationProvider provider = new LdapAuthenticationProvider(
            authenticator(), authoritiesPopulator());
        return provider;
    }

    @Bean
    public BindAuthenticator authenticator() {
        BindAuthenticator authenticator = new BindAuthenticator(contextSource());
        authenticator.setUserDnPatterns(new String[]{"uid={0},ou=people"});
        return authenticator;
    }

    @Bean
    public DefaultLdapAuthoritiesPopulator authoritiesPopulator() {
        return new DefaultLdapAuthoritiesPopulator(contextSource(), "ou=groups");
    }

    @Bean
    public DefaultSpringSecurityContextSource contextSource() {
        return new DefaultSpringSecurityContextSource("ldap://localhost:8389/dc=example,dc=com");
    }
}
```

---

## 27. MULTI-FACTOR AUTHENTICATION (MFA)

### TOTP Implementation Concept
```java
// Dependencies: dev.samstevens.totp:totp
@Service
public class TotpService {

    public String generateSecret() {
        SecretGenerator secretGenerator = new DefaultSecretGenerator();
        return secretGenerator.generate();
    }

    public String getQrCodeUri(String secret, String username) {
        QrData data = new QrData.Builder()
            .label(username)
            .secret(secret)
            .issuer("MyApp")
            .algorithm(HashingAlgorithm.SHA1)
            .digits(6)
            .period(30)
            .build();

        QrGenerator generator = new ZxingPngQrGenerator();
        byte[] imageData = generator.generate(data);
        return getDataUriForImage(imageData, generator.getImageMimeType());
    }

    public boolean verifyCode(String secret, String code) {
        TimeProvider timeProvider = new SystemTimeProvider();
        CodeGenerator codeGenerator = new DefaultCodeGenerator();
        CodeVerifier verifier = new DefaultCodeVerifier(codeGenerator, timeProvider);
        return verifier.isValidCode(secret, code);
    }
}
```

### MFA Authentication Flow
```
Step 1: Username + Password → validates credentials
Step 2: If valid → redirect to TOTP page
Step 3: User enters 6-digit TOTP code
Step 4: Server verifies code → grants full authentication
```

---

## 28. SECURITY EVENTS & AUDITING

### Listening to Authentication Events
```java
@Component
public class AuthenticationEventListener {

    private static final Logger log = LoggerFactory.getLogger(AuthenticationEventListener.class);

    @EventListener
    public void onSuccess(AuthenticationSuccessEvent event) {
        String username = event.getAuthentication().getName();
        log.info("Successful login: {}", username);
    }

    @EventListener
    public void onFailure(AbstractAuthenticationFailureEvent event) {
        String username = event.getAuthentication().getName();
        log.warn("Failed login attempt: {} — Reason: {}",
            username, event.getException().getMessage());
    }
}
```

### Authorization Events
```java
@Component
public class AuthorizationEventListener {

    @EventListener
    public void onAccessDenied(AuthorizationDeniedEvent event) {
        Authentication auth = event.getAuthentication().get();
        log.warn("Access denied for user: {} to resource: {}",
            auth.getName(), event.getSource());
    }
}
```

### Enable Authorization Event Publishing
```java
@Bean
public AuthorizationEventPublisher authorizationEventPublisher(
        ApplicationEventPublisher publisher) {
    return new SpringAuthorizationEventPublisher(publisher);
}
```

---

## 29. SECURITY HEADERS

### Default Headers (Added Automatically)
```
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0
Strict-Transport-Security: max-age=31536000; includeSubDomains (HTTPS only)
```

### Custom Header Configuration
```java
http.headers(headers -> headers
    .frameOptions(frame -> frame.sameOrigin())     // allow iframes from same origin
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; script-src 'self' 'unsafe-inline'"))
    .httpStrictTransportSecurity(hsts -> hsts
        .maxAgeInSeconds(31536000)
        .includeSubDomains(true))
    .permissionsPolicy(permissions -> permissions
        .policy("camera=(), microphone=(), geolocation=()"))
);
```

---

## 30. SPRING SECURITY WITH THYMELEAF

### Dependency
```xml
<dependency>
    <groupId>org.thymeleaf.extras</groupId>
    <artifactId>thymeleaf-extras-springsecurity6</artifactId>
</dependency>
```

### Thymeleaf Security Expressions
```html
<html xmlns:sec="http://www.thymeleaf.org/extras/spring-security">

<!-- Show only for authenticated users -->
<div sec:authorize="isAuthenticated()">
    Welcome, <span sec:authentication="name">Username</span>!
</div>

<!-- Show only for anonymous users -->
<div sec:authorize="isAnonymous()">
    <a href="/login">Login</a>
</div>

<!-- Role-based content -->
<div sec:authorize="hasRole('ADMIN')">
    <a href="/admin">Admin Panel</a>
</div>

<!-- Authority-based content -->
<div sec:authorize="hasAuthority('WRITE_USER')">
    <button>Edit User</button>
</div>

<!-- Display user roles -->
<span sec:authentication="authorities">[ROLE_ADMIN, ROLE_USER]</span>
```

---

## 31. TESTING SPRING SECURITY

### Test Dependencies
```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

### Testing with MockMvc
```java
@SpringBootTest
@AutoConfigureMockMvc
class SecurityTest {

    @Autowired
    private MockMvc mockMvc;

    // Test unauthenticated access
    @Test
    void whenUnauthenticated_thenRedirectToLogin() throws Exception {
        mockMvc.perform(get("/dashboard"))
            .andExpect(status().is3xxRedirection())
            .andExpect(redirectedUrlPattern("**/login"));
    }

    // Test with mock user
    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void whenAdmin_thenAccessGranted() throws Exception {
        mockMvc.perform(get("/admin/dashboard"))
            .andExpect(status().isOk());
    }

    // Test with custom UserDetails
    @Test
    @WithUserDetails("admin")  // loads from UserDetailsService
    void withRealUser_thenAccessGranted() throws Exception {
        mockMvc.perform(get("/admin/dashboard"))
            .andExpect(status().isOk());
    }

    // Test access denied
    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    void whenUser_accessAdmin_thenForbidden() throws Exception {
        mockMvc.perform(get("/admin/dashboard"))
            .andExpect(status().isForbidden());
    }

    // Test login
    @Test
    void testLogin() throws Exception {
        mockMvc.perform(formLogin("/login")
                .user("admin")
                .password("admin123"))
            .andExpect(authenticated().withUsername("admin"))
            .andExpect(redirectedUrl("/dashboard"));
    }

    // Test logout
    @Test
    @WithMockUser
    void testLogout() throws Exception {
        mockMvc.perform(logout("/logout"))
            .andExpect(status().is3xxRedirection())
            .andExpect(unauthenticated());
    }

    // Test with CSRF
    @Test
    @WithMockUser
    void testPostWithCsrf() throws Exception {
        mockMvc.perform(post("/api/data")
                .with(csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"name\": \"test\"}"))
            .andExpect(status().isOk());
    }

    // Test with JWT
    @Test
    void testWithJwt() throws Exception {
        mockMvc.perform(get("/api/protected")
                .with(jwt().authorities(new SimpleGrantedAuthority("ROLE_ADMIN"))))
            .andExpect(status().isOk());
    }
}
```

### Custom @WithMockUser Annotation
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@WithMockUser(username = "admin", roles = {"ADMIN"})
public @interface WithMockAdmin {
}

// Usage
@Test
@WithMockAdmin
void adminTest() throws Exception { ... }
```

---

## 32. COMMON PATTERNS & BEST PRACTICES

### Security Best Practices
1. **Always use HTTPS** in production — enforce with HSTS.
2. **Never store plain text passwords** — use BCrypt or Argon2.
3. **Enable CSRF** for browser-based apps — disable only for stateless APIs.
4. **Use parameterized queries** — prevent SQL injection.
5. **Validate & sanitize input** — prevent XSS.
6. **Principle of least privilege** — grant minimum necessary permissions.
7. **Use `@PreAuthorize`** over URL-based rules for fine-grained control.
8. **Set secure cookie flags** — `HttpOnly`, `Secure`, `SameSite`.
9. **Implement rate limiting** — prevent brute-force attacks.
10. **Audit authentication events** — log successes and failures.
11. **Rotate secrets regularly** — JWT keys, API keys.
12. **Use short-lived tokens** — pair with refresh tokens.

### Common Security Config Pattern (REST API)
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfig()))
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)
                .accessDeniedHandler(accessDeniedHandler))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**", "/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

### Common Security Config Pattern (Web App)
```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/register", "/css/**", "/js/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/login?error")
                .permitAll())
            .logout(logout -> logout
                .logoutSuccessUrl("/login?logout")
                .deleteCookies("JSESSIONID")
                .permitAll())
            .rememberMe(remember -> remember
                .key("remember-me-key")
                .tokenValiditySeconds(86400 * 7))
            .sessionManagement(session -> session
                .maximumSessions(1)
                .expiredUrl("/login?expired"));

        return http.build();
    }
}
```

---

## 33. IMPORTANT ANNOTATIONS REFERENCE

| Annotation | Description |
|-----------|-------------|
| `@EnableWebSecurity` | Enables Spring Security's web security support |
| `@EnableMethodSecurity` | Enables method-level security (`@PreAuthorize`, etc.) |
| `@PreAuthorize("expression")` | Check authorization before method execution |
| `@PostAuthorize("expression")` | Check authorization after method execution |
| `@PreFilter("expression")` | Filter collection parameter before method |
| `@PostFilter("expression")` | Filter collection return value after method |
| `@Secured("ROLE_X")` | Simple role-based method security |
| `@RolesAllowed("ROLE_X")` | JSR-250 role-based security |
| `@AuthenticationPrincipal` | Inject current authenticated user |
| `@CurrentSecurityContext` | Inject current SecurityContext |
| `@WithMockUser` | Test annotation — mock authenticated user |
| `@WithUserDetails` | Test annotation — load real UserDetails |

---

## 34. COMMON CONFIGURATION PROPERTIES

```properties
# Default user
spring.security.user.name=user
spring.security.user.password=secret
spring.security.user.roles=USER

# Session
server.servlet.session.timeout=30m
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.secure=true
server.servlet.session.cookie.same-site=strict

# HTTPS
server.ssl.enabled=true
server.ssl.key-store=classpath:keystore.p12
server.ssl.key-store-password=changeit
server.ssl.key-store-type=PKCS12

# Remember Me
# (configured programmatically — no standard properties)

# Logging (for debugging)
logging.level.org.springframework.security=DEBUG
logging.level.org.springframework.security.web.FilterChainProxy=DEBUG
```

---

## 35. MIGRATION GUIDE — WEBSECURITYCONFIGURERADAPTER TO COMPONENT-BASED

### Old Approach (Deprecated — Spring Security 5.x)
```java
// ❌ DEPRECATED — Do NOT use
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            .and()
            .formLogin();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService)
            .passwordEncoder(passwordEncoder());
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
```

### New Approach (Spring Security 6.x / Spring Boot 3.x)
```java
// ✅ CURRENT — Component-based configuration
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        return new CustomUserDetailsService();
    }

    @Bean
    public AuthenticationManager authManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Key Migration Changes
| Old (5.x) | New (6.x) |
|-----------|-----------|
| Extend `WebSecurityConfigurerAdapter` | No base class — use `@Bean` methods |
| `configure(HttpSecurity)` | `SecurityFilterChain` bean |
| `configure(AuthenticationManagerBuilder)` | `UserDetailsService` + `PasswordEncoder` beans |
| `authenticationManagerBean()` | `AuthenticationManager` from `AuthenticationConfiguration` |
| `authorizeRequests()` | `authorizeHttpRequests()` |
| `antMatchers()` | `requestMatchers()` |
| `.and()` chaining | Lambda DSL (no `.and()`) |
| `@EnableGlobalMethodSecurity` | `@EnableMethodSecurity` |
| `access("hasRole('ADMIN')")` | `hasRole("ADMIN")` direct methods |

---

# PART 2 — DEEP DIVE (INTERNAL FLOW & ADVANCED UNDERSTANDING)

> **Sections 1–35** above cover *what to configure and how*. The sections below explain *why things work the way they do* — the internal mechanics. If you read Sections 1–35 and had questions like "but how does DelegatingFilterProxy connect to my SecurityFilterChain?" or "which filter actually runs on my request?" — this part answers all of that.

---

## 36. DEEP DIVE — HOW SPRING SECURITY INITIALIZES (SERVLET LEVEL)

> This section explains what happens **behind the scenes** when Spring Security starts up — at the Servlet container level. This is the foundation that many developers skip but is critical for understanding the full picture.

### The Big Picture: How Security Gets Registered

```
Application Starts
       │
       ▼
┌─────────────────────────────────────────────┐
│  AbstractSecurityWebApplicationInitializer   │  ← Registers DelegatingFilterProxy
│  (or Spring Boot auto-config)                │     in the Servlet Container
└─────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│  DelegatingFilterProxy                       │  ← A standard Servlet Filter
│  (registered in web.xml or via initializer)  │     Bridges Servlet world → Spring world
└─────────────────────────────────────────────┘
       │ delegates to
       ▼
┌─────────────────────────────────────────────┐
│  FilterChainProxy                            │  ← Spring-managed bean
│  (named "springSecurityFilterChain")         │     Holds all SecurityFilterChain(s)
└─────────────────────────────────────────────┘
       │ selects matching chain
       ▼
┌─────────────────────────────────────────────┐
│  SecurityFilterChain                         │  ← Contains ordered list of
│  (configured via HttpSecurity)               │     security filters
└─────────────────────────────────────────────┘
```

### Step-by-Step: What Happens at Startup

**Step 1 — Registration in Servlet Container**

In **Spring MVC** (without Spring Boot), you extend `AbstractSecurityWebApplicationInitializer`:
```java
public class SecurityInitializer extends AbstractSecurityWebApplicationInitializer {
    // This class does ONE thing:
    // Registers DelegatingFilterProxy with filter name "springSecurityFilterChain"
    // into the Servlet Container (like Tomcat)
}
```

In **Spring Boot**, this step is automatic — `SecurityAutoConfiguration` handles it.

**Step 2 — DelegatingFilterProxy**

- It is a **standard Servlet Filter** (implements `javax.servlet.Filter`).
- The Servlet Container (Tomcat) knows about Servlet Filters, NOT Spring beans.
- `DelegatingFilterProxy` is the **bridge** — it looks up a Spring bean named `springSecurityFilterChain` and delegates all requests to it.

```
Servlet Container (Tomcat)
       │
       │  knows about
       ▼
DelegatingFilterProxy (Servlet Filter)
       │
       │  looks up Spring bean named "springSecurityFilterChain"
       ▼
FilterChainProxy (Spring Bean)
```

**Step 3 — @EnableWebSecurity Creates the Beans**

When you annotate your config with `@EnableWebSecurity`:
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        // configure security rules
        return http.build();
    }
}
```

What `@EnableWebSecurity` does internally:
1. Imports `WebSecurityConfiguration`
2. Creates `FilterChainProxy` bean (named `springSecurityFilterChain`)
3. Collects all `SecurityFilterChain` beans you defined
4. Registers them inside `FilterChainProxy`

```
@EnableWebSecurity
       │
       ├── Creates FilterChainProxy bean ("springSecurityFilterChain")
       │
       ├── Collects all SecurityFilterChain @Bean methods
       │
       └── Registers filters inside FilterChainProxy
```

### How http.formLogin(), http.csrf(), etc. Work

Each method on `HttpSecurity` does NOT just "configure" — it **registers one or more Filters** into the `SecurityFilterChain`:

```
http.formLogin()        → Adds UsernamePasswordAuthenticationFilter
http.httpBasic()        → Adds BasicAuthenticationFilter
http.csrf()             → Adds CsrfFilter
http.logout()           → Adds LogoutFilter
http.sessionManagement()→ Adds SessionManagementFilter
http.authorizeHttpRequests() → Adds AuthorizationFilter
http.exceptionHandling()→ Adds ExceptionTranslationFilter
http.cors()             → Adds CorsFilter
```

So when you write:
```java
http.formLogin(Customizer.withDefaults())
    .csrf(Customizer.withDefaults())
    .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
```

You are actually saying:
```
"Add UsernamePasswordAuthenticationFilter to the chain"
"Add CsrfFilter to the chain"
"Add AuthorizationFilter to the chain"
```

Spring Security then orders them automatically using predefined priority numbers.

### Old vs New: WebSecurityConfigurerAdapter Flow

```
OLD WAY (before 5.7):
┌──────────────────────────────┐
│ @EnableWebSecurity            │
│ extends WebSecurityConfigurerAdapter │
│                              │
│ configure(HttpSecurity http)  │ ← Override to set rules
│ configure(AuthManagerBuilder) │ ← Override to set auth
│ authenticationManagerBean()   │ ← Override to expose bean
└──────────────────────────────┘
       │
       │  @EnableWebSecurity internally creates
       │  SecurityFilterChain from your configure() method
       ▼
FilterChainProxy holds the chain

NEW WAY (5.7+):
┌──────────────────────────────┐
│ @EnableWebSecurity            │
│                              │
│ @Bean SecurityFilterChain     │ ← You directly create the bean
│ @Bean PasswordEncoder         │ ← You directly create the bean
│ @Bean UserDetailsService      │ ← You directly create the bean
└──────────────────────────────┘
       │
       │  @EnableWebSecurity collects your
       │  SecurityFilterChain beans
       ▼
FilterChainProxy holds the chain(s)
```

The key difference: In old way, `WebSecurityConfigurerAdapter` was a helper that created `SecurityFilterChain` for you from your `configure()` overrides. In new way, you create the `SecurityFilterChain` bean directly — no middleman.

---

## 37. DEEP DIVE — WHICH FILTERS RUN FOR WHICH REQUEST?

> A common confusion: "Does every filter in the SecurityFilterChain run for every request?"

### Answer: NO — Each Filter Decides Whether to Act

Most filters run for every request BUT they **decide internally** whether to actually DO something or just pass through.

### Example: UsernamePasswordAuthenticationFilter

This filter does **NOT** execute its authentication logic on every request. It only acts when:
- The request matches **POST /login** (by default)

For all other requests, it simply calls `filterChain.doFilter(request, response)` — passing the request to the next filter without doing anything.

```
GET /home
       │
       ▼
UsernamePasswordAuthenticationFilter
  → Checks: Is this POST /login?
  → NO → Just pass through (doFilter)
       │
       ▼
Next Filter...

POST /login
       │
       ▼
UsernamePasswordAuthenticationFilter
  → Checks: Is this POST /login?
  → YES → Extract username/password
  → Create UsernamePasswordAuthenticationToken
  → Call AuthenticationManager.authenticate()
  → Handle success/failure
```

### How Each Filter Decides

```
┌─────────────────────────────────────┬──────────────────────────────────┐
│ Filter                              │ Acts Only When                   │
├─────────────────────────────────────┼──────────────────────────────────┤
│ SecurityContextHolderFilter         │ EVERY request (loads context)    │
│ HeaderWriterFilter                  │ EVERY request (adds headers)     │
│ CsrfFilter                         │ State-changing requests           │
│                                     │ (POST, PUT, DELETE)              │
│ LogoutFilter                        │ POST /logout (default)           │
│ UsernamePasswordAuthenticationFilter│ POST /login (default)            │
│ BasicAuthenticationFilter           │ Request has Authorization:Basic  │
│ AnonymousAuthenticationFilter       │ When no auth exists yet          │
│ ExceptionTranslationFilter          │ EVERY request (catches exceptions)│
│ AuthorizationFilter                 │ EVERY request (checks access)    │
└─────────────────────────────────────┴──────────────────────────────────┘
```

### Visual: Full Request Flow Through Filters

```
GET /api/products (with JWT token)
       │
       ▼
SecurityContextHolderFilter ── Loads SecurityContext (empty for stateless)
       │
       ▼
HeaderWriterFilter ────────── Adds security headers
       │
       ▼
CsrfFilter ───────────────── GET request → skip CSRF check
       │
       ▼
LogoutFilter ─────────────── Not /logout → skip
       │
       ▼
JwtAuthFilter (custom) ───── Has Bearer token → validate → set Authentication
       │
       ▼
UsernamePasswordAuthFilter ── Not POST /login → skip
       │
       ▼
AnonymousAuthenticationFilter── Auth exists → skip
       │
       ▼
ExceptionTranslationFilter ── Wraps remaining chain in try/catch
       │
       ▼
AuthorizationFilter ────────── Check: Does user have access to /api/products?
       │
       ▼
DispatcherServlet → Controller
```

---

## 38. DEEP DIVE — AUTHENTICATION MANAGER & PROVIDER SELECTION

### The Full Authentication Architecture

```
UsernamePasswordAuthenticationFilter (or any filter/controller)
       │
       │ creates Authentication object (unauthenticated)
       │ e.g., UsernamePasswordAuthenticationToken("admin", "pass123")
       ▼
AuthenticationManager (interface)
       │
       │ default implementation = ProviderManager
       │
       ▼
ProviderManager
       │
       │ iterates over List<AuthenticationProvider>
       │
       ├── Provider 1: supports(UsernamePasswordAuthenticationToken.class)?
       │   └── YES → authenticate() → success? → return
       │                             → fail? → try next
       │
       ├── Provider 2: supports(UsernamePasswordAuthenticationToken.class)?
       │   └── YES → authenticate() → success? → return
       │                             → fail? → try next
       │
       └── No provider succeeded → throw AuthenticationException
```

### How `supports()` Method Works

Every `AuthenticationProvider` has a `supports(Class<?> authentication)` method that tells the `ProviderManager`: "I can handle this type of authentication token."

```java
// DaoAuthenticationProvider (built-in)
@Override
public boolean supports(Class<?> authentication) {
    return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
}
```

The `ProviderManager` uses this to **filter** which providers to try:

```
ProviderManager.authenticate(Authentication auth) {
    for (AuthenticationProvider provider : providers) {
        if (provider.supports(auth.getClass())) {   // ← Ask: Can you handle this?
            try {
                result = provider.authenticate(auth);  // ← Try it
                if (result != null) return result;       // ← Success!
            } catch (AuthenticationException e) {
                lastException = e;                       // ← Failed, try next
            }
        }
    }
    throw lastException;  // ← All failed
}
```

### What If Two Providers Support the Same Token Type?

If both `DaoAuthenticationProvider` and your `CustomAuthProvider` support `UsernamePasswordAuthenticationToken`:

```
ProviderManager iterates in order:
       │
       ├── Provider 1 (DaoAuthenticationProvider)
       │   supports(UsernamePasswordAuthenticationToken)? → YES
       │   authenticate() → tries → SUCCESS → RETURN ✅ (stops here)
       │                          → FAIL → continue to next
       │
       ├── Provider 2 (CustomAuthProvider)
       │   supports(UsernamePasswordAuthenticationToken)? → YES
       │   authenticate() → tries → SUCCESS → RETURN ✅
       │                          → FAIL → throw exception
       │
       └── Both failed → AuthenticationException thrown
```

**Key Rule**: The **first provider that succeeds wins**. Order matters!

To control order, register providers explicitly:
```java
@Bean
public AuthenticationManager authManager(HttpSecurity http) throws Exception {
    AuthenticationManagerBuilder builder =
        http.getSharedObject(AuthenticationManagerBuilder.class);
    builder.authenticationProvider(customProvider);   // checked first
    builder.authenticationProvider(daoProvider);      // checked second
    return builder.build();
}
```

---

## 39. DEEP DIVE — CUSTOM AUTHENTICATION (OTP EXAMPLE)

> What if you want OTP-based login instead of username/password? You need a custom Authentication Token, Filter, and Provider.

### Step 1 — Create Custom Authentication Token

```java
public class OtpAuthenticationToken extends AbstractAuthenticationToken {

    private final String phoneNumber;   // principal
    private final String otp;           // credentials

    // Unauthenticated (before verification)
    public OtpAuthenticationToken(String phoneNumber, String otp) {
        super(null);
        this.phoneNumber = phoneNumber;
        this.otp = otp;
        setAuthenticated(false);
    }

    // Authenticated (after verification)
    public OtpAuthenticationToken(String phoneNumber,
            Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.phoneNumber = phoneNumber;
        this.otp = null;
        setAuthenticated(true);
    }

    @Override
    public Object getCredentials() { return otp; }

    @Override
    public Object getPrincipal() { return phoneNumber; }
}
```

### Step 2 — Create Custom Authentication Provider

```java
@Component
public class OtpAuthenticationProvider implements AuthenticationProvider {

    @Autowired
    private OtpService otpService;

    @Autowired
    private UserRepository userRepository;

    @Override
    public Authentication authenticate(Authentication authentication)
            throws AuthenticationException {

        String phoneNumber = authentication.getPrincipal().toString();
        String otp = authentication.getCredentials().toString();

        // Verify OTP
        if (!otpService.verifyOtp(phoneNumber, otp)) {
            throw new BadCredentialsException("Invalid OTP");
        }

        // Load user
        User user = userRepository.findByPhoneNumber(phoneNumber)
            .orElseThrow(() -> new BadCredentialsException("User not found"));

        List<GrantedAuthority> authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
            .collect(Collectors.toList());

        return new OtpAuthenticationToken(phoneNumber, authorities);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return OtpAuthenticationToken.class.isAssignableFrom(authentication);
        // ↑ Only handles OtpAuthenticationToken, NOT UsernamePasswordAuthenticationToken
    }
}
```

### Step 3 — Create Custom Filter (Optional — for form-based OTP)

```java
public class OtpAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

    public OtpAuthenticationFilter() {
        super(new AntPathRequestMatcher("/otp-login", "POST"));
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request,
            HttpServletResponse response) throws AuthenticationException {

        String phoneNumber = request.getParameter("phoneNumber");
        String otp = request.getParameter("otp");

        OtpAuthenticationToken token = new OtpAuthenticationToken(phoneNumber, otp);
        return getAuthenticationManager().authenticate(token);
    }
}
```

### How It All Connects

```
POST /otp-login { phoneNumber: "9876543210", otp: "123456" }
       │
       ▼
OtpAuthenticationFilter
       │ creates OtpAuthenticationToken (unauthenticated)
       ▼
AuthenticationManager (ProviderManager)
       │
       │ iterates providers:
       │
       ├── DaoAuthenticationProvider
       │   supports(OtpAuthenticationToken)? → NO → skip
       │
       ├── OtpAuthenticationProvider
       │   supports(OtpAuthenticationToken)? → YES
       │   authenticate() → verify OTP → load user → return authenticated token
       │
       ▼
SecurityContextHolder.setAuthentication(authenticatedToken)
       │
       ▼
Success → return JWT token or redirect
```

**Key Insight**: Each authentication method gets its own:
1. **Token class** — defines what credentials look like
2. **Provider** — knows how to verify those credentials
3. **Filter** (optional) — extracts credentials from the HTTP request

The `supports()` method ensures the right provider handles the right token type.

---

## 40. DEEP DIVE — SECURITYCONTEXTHOLDER: STATEFUL vs STATELESS

> This is one of the most confusing topics. Understanding this clears up 90% of Spring Security confusion.

### What is SecurityContextHolder?

```
SecurityContextHolder (static class)
       │
       │ holds (via ThreadLocal)
       ▼
SecurityContext (interface)
       │
       │ holds
       ▼
Authentication (interface)
       │
       ├── getPrincipal()    → UserDetails / username
       ├── getCredentials()  → password (usually null after auth)
       ├── getAuthorities()  → roles/permissions
       └── isAuthenticated() → true/false
```

**Critical Point**: `SecurityContextHolder` uses `ThreadLocal` → it is **per-thread, per-request**. It does NOT survive across requests on its own.

### What Survives Across Requests?

| What | Survives? | How? |
|------|-----------|------|
| `SecurityContextHolder` | NO | ThreadLocal — cleared after each request |
| `SecurityContext` | Depends | In stateful → saved in HttpSession |
| `HttpSession` | YES | Stored on server, identified by JSESSIONID cookie |
| JWT Token | YES | Stored on client, sent with every request |

---

### STATEFUL FLOW (Form Login) — Complete Lifecycle

#### Login Request

```
POST /login (username=admin, password=secret)
       │
       ▼
UsernamePasswordAuthenticationFilter
       │ extracts username & password
       │ creates UsernamePasswordAuthenticationToken (unauthenticated)
       ▼
AuthenticationManager → AuthenticationProvider
       │ validates credentials
       │ returns authenticated token
       ▼
SecurityContextHolder.getContext().setAuthentication(authToken)
       │
       ▼
HttpSession stores SecurityContext
       │  session.setAttribute("SPRING_SECURITY_CONTEXT", securityContext)
       ▼
Response sent with JSESSIONID cookie
```

#### Next Request (Already Logged In)

```
GET /profile (Cookie: JSESSIONID=abc123)
       │
       ▼
SecurityContextHolderFilter (runs FIRST in the chain)
       │
       │ calls securityContextRepository.loadContext(request)
       │ which does: session.getAttribute("SPRING_SECURITY_CONTEXT")
       │ found? → SecurityContextHolder.setContext(loadedContext)
       ▼
Now Authentication is available in ThreadLocal for this request
       │
       ▼
AuthorizationFilter → checks roles → allowed
       │
       ▼
Controller → can use @AuthenticationPrincipal, Principal, etc.
       │
       ▼
SecurityContextHolderFilter (at END of request)
       │ saves context back: securityContextRepository.saveContext(...)
       │ clears ThreadLocal: SecurityContextHolder.clearContext()
       ▼
Response sent
```

#### Key Classes in Stateful Flow

```
SecurityContextHolderFilter
       │ uses
       ▼
SecurityContextRepository (interface)
       │ default implementation
       ▼
HttpSessionSecurityContextRepository
       │
       ├── loadContext()  → session.getAttribute("SPRING_SECURITY_CONTEXT")
       └── saveContext()  → session.setAttribute("SPRING_SECURITY_CONTEXT", ctx)
```

> In older Spring Security versions (before 5.7), `SecurityContextPersistenceFilter` did this job. Now it's `SecurityContextHolderFilter`.

---

### STATELESS FLOW (JWT) — Complete Lifecycle

#### Login Request (Get Token)

```
POST /auth/login { "username": "admin", "password": "secret" }
       │
       ▼
AuthController.login()
       │
       │ authenticationManager.authenticate(
       │     new UsernamePasswordAuthenticationToken(username, password)
       │ )
       ▼
AuthenticationManager → AuthenticationProvider → validates
       │
       ▼
Generate JWT token: jwtService.generateToken(userDetails)
       │
       ▼
Return token in response body: { "token": "eyJhbGciOi..." }
       │
       ▼
NO session created. NO SecurityContext stored.
Client saves token (localStorage / cookie).
```

#### Next Request (With Token)

```
GET /api/products
Header: Authorization: Bearer eyJhbGciOi...
       │
       ▼
JwtAuthFilter (custom filter, runs before UsernamePasswordAuthFilter)
       │
       ├── Extract token from Authorization header
       ├── Parse & validate token (signature, expiration)
       ├── Extract username from token claims
       ├── Load UserDetails from DB
       ├── Create authenticated UsernamePasswordAuthenticationToken
       ├── SecurityContextHolder.getContext().setAuthentication(authToken)
       │
       ▼
AuthorizationFilter → checks roles → allowed
       │
       ▼
Controller handles request
       │
       ▼
SecurityContextHolder.clearContext()  ← cleaned at end
       │
       ▼
Response sent. Nothing stored on server.
```

#### Why We Set SecurityContextHolder in JWT Flow

> "If SecurityContextHolder is per-request and JWT is stateless, why set it at all?"

Because **within the same request**, other parts of Spring Security need it:

```
JwtAuthFilter sets SecurityContextHolder
       │
       │  Same request, same thread:
       │
       ├── AuthorizationFilter checks: Does user have ROLE_ADMIN?
       │   └── Reads from SecurityContextHolder ✅
       │
       ├── @PreAuthorize("hasRole('ADMIN')") in service layer
       │   └── Reads from SecurityContextHolder ✅
       │
       ├── Controller: @AuthenticationPrincipal UserDetails user
       │   └── Reads from SecurityContextHolder ✅
       │
       └── After response → SecurityContextHolder cleared
```

Without setting it, authorization checks would fail even though the JWT was valid.

---

### SIDE-BY-SIDE COMPARISON

```
┌──────────────────────────────┬──────────────────────────────┐
│      FORM LOGIN (Stateful)   │         JWT (Stateless)       │
├──────────────────────────────┼──────────────────────────────┤
│                              │                              │
│  LOGIN:                      │  LOGIN:                      │
│  POST /login                 │  POST /auth/login            │
│       │                      │       │                      │
│       ▼                      │       ▼                      │
│  AuthManager validates       │  AuthManager validates       │
│       │                      │       │                      │
│       ▼                      │       ▼                      │
│  Store in Session            │  Generate JWT token          │
│  (server remembers)          │  (client stores)             │
│       │                      │       │                      │
│       ▼                      │       ▼                      │
│  Set JSESSIONID cookie       │  Return token in JSON body   │
│                              │                              │
├──────────────────────────────┼──────────────────────────────┤
│                              │                              │
│  NEXT REQUEST:               │  NEXT REQUEST:               │
│  GET /profile                │  GET /api/products           │
│  Cookie: JSESSIONID=abc      │  Header: Bearer eyJ...       │
│       │                      │       │                      │
│       ▼                      │       ▼                      │
│  SecurityContextHolderFilter │  JwtAuthFilter               │
│  loads from session          │  validates token             │
│       │                      │       │                      │
│       ▼                      │       ▼                      │
│  Set SecurityContextHolder   │  Set SecurityContextHolder   │
│       │                      │       │                      │
│       ▼                      │       ▼                      │
│  AuthorizationFilter         │  AuthorizationFilter         │
│       │                      │       │                      │
│       ▼                      │       ▼                      │
│  Controller                  │  Controller                  │
│       │                      │       │                      │
│       ▼                      │       ▼                      │
│  Save context to session     │  Clear context (nothing      │
│  Clear ThreadLocal           │  saved on server)            │
│                              │                              │
├──────────────────────────────┼──────────────────────────────┤
│  Identity stored: SERVER     │  Identity stored: CLIENT     │
│  Session: YES                │  Session: NO                 │
│  JSESSIONID cookie: YES      │  Authorization header: YES   │
│  Scalability: Limited        │  Scalability: High           │
│  Policy: IF_REQUIRED         │  Policy: STATELESS           │
└──────────────────────────────┴──────────────────────────────┘
```

---

## 41. DEEP DIVE — COMPLETE REQUEST LIFECYCLE (EVERY FILTER)

> Trace of a real request through ALL filters, from start to end.

### Scenario: GET /profile (Authenticated user, Form Login, with session)

```
Browser sends: GET /profile
Cookie: JSESSIONID=abc123
       │
       ▼
═══════════════════════════════════════════════════
FILTER 1: DisableEncodeUrlFilter (Order 100)
  → Prevents session ID from appearing in URLs
  → Always runs, no condition
═══════════════════════════════════════════════════
       │
       ▼
═══════════════════════════════════════════════════
FILTER 2: SecurityContextHolderFilter (Order 400)
  → Loads SecurityContext from HttpSession
  → session.getAttribute("SPRING_SECURITY_CONTEXT")
  → Sets into SecurityContextHolder (ThreadLocal)
  → NOW authentication is available for this thread
═══════════════════════════════════════════════════
       │
       ▼
═══════════════════════════════════════════════════
FILTER 3: HeaderWriterFilter (Order 500)
  → Adds security response headers:
    X-Content-Type-Options: nosniff
    X-Frame-Options: DENY
    Cache-Control: no-cache
═══════════════════════════════════════════════════
       │
       ▼
═══════════════════════════════════════════════════
FILTER 4: CsrfFilter (Order 700)
  → GET request → no CSRF validation needed → pass through
  → (Only validates on POST/PUT/DELETE)
═══════════════════════════════════════════════════
       │
       ▼
═══════════════════════════════════════════════════
FILTER 5: LogoutFilter (Order 800)
  → Request is GET /profile, not POST /logout → skip
═══════════════════════════════════════════════════
       │
       ▼
═══════════════════════════════════════════════════
FILTER 6: UsernamePasswordAuthenticationFilter (Order 1000)
  → Request is GET /profile, not POST /login → skip
  → (Only acts on POST /login by default)
═══════════════════════════════════════════════════
       │
       ▼
═══════════════════════════════════════════════════
FILTER 7: RequestCacheAwareFilter (Order 1200)
  → Checks if there was a saved request (e.g., redirect after login)
  → No saved request → pass through
═══════════════════════════════════════════════════
       │
       ▼
═══════════════════════════════════════════════════
FILTER 8: AnonymousAuthenticationFilter (Order 1400)
  → Authentication already exists in SecurityContextHolder → skip
  → (Only creates AnonymousAuthenticationToken if no auth exists)
═══════════════════════════════════════════════════
       │
       ▼
═══════════════════════════════════════════════════
FILTER 9: ExceptionTranslationFilter (Order 1600)
  → Wraps the remaining filters in try-catch
  → If AuthenticationException → redirect to login
  → If AccessDeniedException → return 403
═══════════════════════════════════════════════════
       │
       ▼
═══════════════════════════════════════════════════
FILTER 10: AuthorizationFilter (Order 1700)
  → Checks: Is /profile accessible to this user?
  → Reads Authentication from SecurityContextHolder
  → User has ROLE_USER → rule says authenticated() → ALLOWED ✅
═══════════════════════════════════════════════════
       │
       ▼
═══════════════════════════════════════════════════
DispatcherServlet → ProfileController.getProfile()
       │
       ▼
Response generated
       │
       ▼
═══════════════════════════════════════════════════
CLEANUP (on way back through filter chain):
  → SecurityContextHolder saves context back to session
  → SecurityContextHolder.clearContext() → ThreadLocal cleaned
═══════════════════════════════════════════════════
       │
       ▼
Response sent to browser
```

### Scenario: POST /login (Unauthenticated user, Form Login)

```
Browser sends: POST /login
Body: username=admin&password=secret
       │
       ▼
SecurityContextHolderFilter
  → No session yet → empty SecurityContext
       │
       ▼
HeaderWriterFilter → adds headers
       │
       ▼
CsrfFilter
  → POST request → validates CSRF token from form → VALID ✅
       │
       ▼
LogoutFilter → Not /logout → skip
       │
       ▼
UsernamePasswordAuthenticationFilter ← THIS IS WHERE THE ACTION IS
  → Is this POST /login? → YES!
  → Extract username & password from request
  → Create UsernamePasswordAuthenticationToken (unauthenticated)
  │
  ├── Call AuthenticationManager.authenticate(token)
  │        │
  │        ▼
  │   ProviderManager
  │        │
  │        ▼
  │   DaoAuthenticationProvider
  │        ├── UserDetailsService.loadUserByUsername("admin")
  │        ├── Get UserDetails from DB
  │        ├── PasswordEncoder.matches("secret", "$2a$10$...")
  │        ├── Password matches → Create authenticated token
  │        └── Return authenticated UsernamePasswordAuthenticationToken
  │
  ├── SUCCESS:
  │   SecurityContextHolder.setAuthentication(authenticatedToken)
  │   Save SecurityContext to HttpSession
  │   Redirect to /dashboard (defaultSuccessUrl)
  │   ← Request processing STOPS here (redirect sent)
  │
  └── FAILURE:
      Redirect to /login?error
      ← Request processing STOPS here
```

---

## 42. KEY MENTAL MODEL — CHEAT SHEET

### The 4 Core Questions Spring Security Answers

```
For EVERY request, Spring Security answers:

Q1. WHO are you?
    → Authentication (login, JWT, session)

Q2. Can I trust that you are who you say you are?
    → AuthenticationProvider (password check, token validation)

Q3. WHAT are you allowed to do?
    → Authorization (roles, authorities, @PreAuthorize)

Q4. Are you doing something suspicious?
    → Protection filters (CSRF, CORS, headers)
```

### When to Use What

```
┌─────────────────────────────────────────────────────────────────┐
│ USE CASE                         │ APPROACH                     │
├──────────────────────────────────┼──────────────────────────────┤
│ Web app with login page          │ formLogin() + session        │
│ REST API consumed by SPA         │ JWT + stateless              │
│ Microservice-to-microservice     │ JWT or OAuth2                │
│ Admin panel with remember me     │ formLogin() + rememberMe()   │
│ Public API + some protected      │ permitAll() + authenticated()│
│ Multi-tenant with different auth │ Multiple SecurityFilterChains│
│ OTP / biometric login            │ Custom Token + Provider      │
└──────────────────────────────────┴──────────────────────────────┘
```

### Quick Reference: Object Relationships

```
SecurityContextHolder
  └── SecurityContext
        └── Authentication
              ├── Principal      → WHO (UserDetails / username)
              ├── Credentials    → PROOF (password / token — cleared after auth)
              ├── Authorities    → PERMISSIONS (ROLE_ADMIN, READ_USER)
              └── Details        → EXTRA INFO (IP address, session ID)

AuthenticationManager
  └── ProviderManager
        └── List<AuthenticationProvider>
              ├── DaoAuthenticationProvider      → username/password from DB
              ├── LdapAuthenticationProvider      → LDAP directory
              ├── OtpAuthenticationProvider       → custom OTP
              └── JwtAuthenticationProvider       → custom JWT
                    └── Each has supports() method
                          → tells which Authentication token it handles

SecurityFilterChain
  └── List<Filter>  (ordered by priority)
        ├── SecurityContextHolderFilter   → load/save context
        ├── CsrfFilter                    → CSRF protection
        ├── LogoutFilter                  → handle logout
        ├── UsernamePasswordAuthFilter    → form login
        ├── BasicAuthFilter               → HTTP Basic
        ├── JwtAuthFilter (custom)        → JWT validation
        ├── ExceptionTranslationFilter    → error handling
        └── AuthorizationFilter           → access control
```

---
