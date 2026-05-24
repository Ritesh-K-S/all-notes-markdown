# Spring MVC — Complete Notes

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Architecture & Request Lifecycle](#2-architecture--request-lifecycle)
3. [Project Setup & Configuration](#3-project-setup--configuration)
4. [Controllers](#4-controllers)
5. [Request Data Binding](#5-request-data-binding)
6. [Model & Passing Data to View](#6-model--passing-data-to-view)
7. [View Resolvers & View Technologies](#7-view-resolvers--view-technologies)
8. [Form Handling & Data Binding](#8-form-handling--data-binding)
9. [Validation](#9-validation)
10. [Exception Handling](#10-exception-handling)
11. [REST Controllers & Building APIs](#11-rest-controllers--building-apis)
12. [Interceptors](#12-interceptors)
13. [Filters](#13-filters)
14. [CORS (Cross-Origin Resource Sharing)](#14-cors-cross-origin-resource-sharing)
15. [Static Resources](#15-static-resources)
16. [File Upload & Download](#16-file-upload--download)
17. [Session Management](#17-session-management)
18. [Internationalization (i18n)](#18-internationalization-i18n)
19. [Async Request Processing](#19-async-request-processing)
20. [WebMvcConfigurer — Full Customization](#20-webmvcconfigurer--full-customization)
21. [Custom Argument Resolvers](#21-custom-argument-resolvers)
22. [Testing Spring MVC](#22-testing-spring-mvc)
23. [Security Basics (Spring Security Integration)](#23-security-basics-spring-security-integration)
24. [Common Patterns & Best Practices](#24-common-patterns--best-practices)
25. [Important Annotations Reference](#25-important-annotations-reference)
26. [Common Configuration Properties](#26-common-configuration-properties)
27. [Spring MVC vs Spring WebFlux](#27-spring-mvc-vs-spring-webflux)
28. [Quick Reference — Complete CRUD Example](#28-quick-reference--complete-crud-example)
29. [Web Scopes](#29-web-scopes)
30. [Custom HttpMessageConverter](#30-custom-httpmessageconverter)
31. [HTTP Clients — RestTemplate & WebClient](#31-http-clients--resttemplate--webclient)
32. [Problem Details — RFC 9457 (Spring 6+)](#32-problem-details--rfc-9457-spring-6)
33. [OpenAPI & Swagger](#33-openapi--swagger)

---

## 1. INTRODUCTION

### What is Spring MVC?
- **Spring MVC** = Spring Model-View-Controller — a web framework built on top of the **Servlet API**.
- Part of the **Spring Framework** (module: `spring-webmvc`).
- Follows the **Front Controller** design pattern — all requests go through a single `DispatcherServlet`.
- Used to build **web applications** (HTML pages) and **RESTful APIs** (JSON/XML).
- Provides clean separation of concerns between **Model**, **View**, and **Controller**.

### Why Spring MVC?
| Feature | Benefit |
|---------|---------|
| Convention over configuration | Less boilerplate |
| Annotation-driven | `@Controller`, `@RequestMapping` — no XML needed |
| Flexible view resolution | JSP, Thymeleaf, FreeMarker, JSON, etc. |
| Powerful data binding | Request params → Java objects automatically |
| Validation support | JSR-303/380 Bean Validation integration |
| Interceptors | Pre/post processing of requests |
| Exception handling | Centralized `@ControllerAdvice` |
| Testability | MockMvc for integration testing |
| Integration | Works seamlessly with Spring Core (DI, AOP, Security) |

### Spring MVC vs Other Frameworks
| Feature | Spring MVC | JSF | Struts 2 | JAX-RS (Jersey) |
|---------|-----------|-----|----------|-----------------|
| Pattern | Front Controller | Component-based | Front Controller | Resource-based |
| View Tech | Multiple | Facelets | JSP/FreeMarker | N/A (REST only) |
| REST Support | Excellent | Limited | Limited | Excellent |
| Learning Curve | Moderate | Steep | Moderate | Low |
| Community | Huge | Declining | Declining | Moderate |

### Maven Dependencies
```xml
<!-- Standalone Spring MVC (without Spring Boot) -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>6.1.0</version>
</dependency>

<!-- With Spring Boot (recommended) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.2.0</version>
</dependency>
<!-- Includes: spring-webmvc, embedded Tomcat, Jackson (JSON), validation -->
```

---

## 2. ARCHITECTURE & REQUEST LIFECYCLE

### MVC Pattern
```
┌──────────┐       ┌────────────┐       ┌──────────┐
│  Client   │──────▶│ Controller │──────▶│  Model   │
│ (Browser) │◀──────│  (Logic)   │◀──────│  (Data)  │
└──────────┘       └────────────┘       └──────────┘
      ▲                   │
      │                   ▼
      │            ┌──────────┐
      └────────────│   View   │
                   │ (UI/HTML)│
                   └──────────┘
```

- **Model** — Data + business logic. POJOs, entities, DTOs, services.
- **View** — Presentation layer. JSP, Thymeleaf, JSON response.
- **Controller** — Handles incoming requests, invokes services, returns model + view name.

### Front Controller Pattern — DispatcherServlet
```
                        ┌─────────────────────────────────────┐
                        │          DispatcherServlet           │
                        │        (Front Controller)            │
                        └───────┬───────┬───────┬─────────────┘
                                │       │       │
                    ┌───────────┘       │       └───────────┐
                    ▼                   ▼                   ▼
            ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
            │ HandlerMapping│   │HandlerAdapter│   │ViewResolver  │
            └──────────────┘   └──────────────┘   └──────────────┘
```

### Complete Request Lifecycle (Step-by-Step)
```
1. Client sends HTTP Request
        │
        ▼
2. DispatcherServlet receives it (configured in web.xml or auto-configured)
        │
        ▼
3. HandlerMapping determines which Controller method to call
   (based on URL pattern, annotations like @RequestMapping)
        │
        ▼
4. HandlerAdapter invokes the Controller method
        │
        ▼
5. Controller processes the request:
   - Calls Service layer
   - Populates Model with data
   - Returns a view name (String) or ResponseBody (REST)
        │
        ▼
6. ViewResolver resolves the view name to an actual View object
   (e.g., "home" → /WEB-INF/views/home.jsp)
        │
        ▼
7. View renders the response using Model data
        │
        ▼
8. DispatcherServlet sends HTTP Response back to client
```

### Key Components
| Component | Responsibility |
|-----------|---------------|
| `DispatcherServlet` | Front controller — routes all requests |
| `HandlerMapping` | Maps URL → Controller method |
| `HandlerAdapter` | Invokes the matched handler method |
| `Controller` | Processes request, returns model + view |
| `ViewResolver` | Resolves logical view name → actual view |
| `View` | Renders the response (HTML, JSON, etc.) |
| `HandlerInterceptor` | Pre/post processing (like filters) |
| `HandlerExceptionResolver` | Handles exceptions globally |
| `MessageConverter` | Converts request/response bodies (JSON ↔ Java) |
| `LocaleResolver` | Determines locale for i18n |
| `MultipartResolver` | Handles file uploads |

---

## 3. PROJECT SETUP & CONFIGURATION

### 3.1 Spring Boot Setup (Recommended)

#### Project Structure
```
src/
├── main/
│   ├── java/
│   │   └── com/example/demo/
│   │       ├── DemoApplication.java          ← Main class
│   │       ├── controller/
│   │       │   └── HomeController.java
│   │       ├── service/
│   │       │   └── UserService.java
│   │       ├── repository/
│   │       │   └── UserRepository.java
│   │       ├── model/
│   │       │   └── User.java
│   │       ├── dto/
│   │       │   └── UserDTO.java
│   │       ├── config/
│   │       │   └── WebConfig.java
│   │       └── exception/
│   │           └── GlobalExceptionHandler.java
│   └── resources/
│       ├── application.properties
│       ├── static/                            ← CSS, JS, images
│       │   ├── css/
│       │   ├── js/
│       │   └── images/
│       └── templates/                         ← Thymeleaf templates
│           ├── home.html
│           └── user/
│               ├── list.html
│               └── form.html
└── test/
    └── java/
        └── com/example/demo/
            └── controller/
                └── HomeControllerTest.java
```

#### Main Application Class
```java
@SpringBootApplication   // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

#### application.properties
```properties
# Server
server.port=8080
server.servlet.context-path=/myapp

# Thymeleaf
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html
spring.thymeleaf.cache=false

# JSP (if using JSP instead of Thymeleaf)
# spring.mvc.view.prefix=/WEB-INF/views/
# spring.mvc.view.suffix=.jsp

# Static resources
spring.web.resources.static-locations=classpath:/static/

# Logging
logging.level.org.springframework.web=DEBUG
```

### 3.2 Traditional Setup (Without Spring Boot)

#### web.xml Configuration
```xml
<!-- src/main/webapp/WEB-INF/web.xml -->
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" version="4.0">

    <!-- DispatcherServlet Registration -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/spring-mvc-config.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

    <!-- Character Encoding Filter -->
    <filter>
        <filter-name>encodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>encodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

</web-app>
```

#### Spring MVC XML Configuration
```xml
<!-- /WEB-INF/spring-mvc-config.xml -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context">

    <!-- Enable annotation-driven MVC -->
    <mvc:annotation-driven/>

    <!-- Component scanning for controllers -->
    <context:component-scan base-package="com.example.controller"/>

    <!-- View Resolver -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/"/>
        <property name="suffix" value=".jsp"/>
    </bean>

    <!-- Static resources -->
    <mvc:resources mapping="/static/**" location="/static/"/>

</beans>
```

### 3.3 Java-Based Configuration (No XML)

#### Option A — WebApplicationInitializer (Raw Interface)
```java
// WebApplicationInitializer is the low-level interface detected by Spring's
// SpringServletContainerInitializer on startup (via Servlet 3.0 ServletContainerInitializer).
// Spring scans the classpath for implementations and calls onStartup() automatically.
// No web.xml needed.
public class MyWebAppInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {

        // 1. Create the root application context (services, repositories)
        AnnotationConfigWebApplicationContext rootCtx = new AnnotationConfigWebApplicationContext();
        rootCtx.register(RootConfig.class);

        // 2. Register ContextLoaderListener so rootCtx is tied to the ServletContext lifecycle
        servletContext.addListener(new ContextLoaderListener(rootCtx));

        // 3. Create the DispatcherServlet's own application context (MVC beans)
        AnnotationConfigWebApplicationContext dispatcherCtx = new AnnotationConfigWebApplicationContext();
        dispatcherCtx.register(WebConfig.class);

        // 4. Register and map DispatcherServlet
        ServletRegistration.Dynamic dispatcher =
            servletContext.addServlet("dispatcher", new DispatcherServlet(dispatcherCtx));
        dispatcher.setLoadOnStartup(1);
        dispatcher.addMapping("/");

        // 5. Optional — add character encoding filter
        FilterRegistration.Dynamic encodingFilter =
            servletContext.addFilter("encodingFilter", CharacterEncodingFilter.class);
        encodingFilter.setInitParameter("encoding", "UTF-8");
        encodingFilter.setInitParameter("forceEncoding", "true");
        encodingFilter.addMappingForUrlPatterns(null, false, "/*");
    }
}
```

#### Option B — AbstractAnnotationConfigDispatcherServletInitializer (Convenience Subclass)
```java
// Simpler alternative — extends the abstract class that implements WebApplicationInitializer
// and handles all the boilerplate from Option A automatically.
// Replaces web.xml
public class WebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{RootConfig.class};   // Service, Repository beans
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class};    // MVC config
    }

    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }

    @Override
    protected Filter[] getServletFilters() {
        CharacterEncodingFilter filter = new CharacterEncodingFilter();
        filter.setEncoding("UTF-8");
        filter.setForceEncoding(true);
        return new Filter[]{filter};
    }
}
```

```java
// MVC Configuration
@Configuration
@EnableWebMvc
@ComponentScan("com.example")
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        return resolver;
    }

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("/static/");
    }

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
```

---

## 4. CONTROLLERS

### 4.1 @Controller vs @RestController

```java
// @Controller — Returns VIEW names (for HTML pages)
@Controller
public class PageController {

    @GetMapping("/home")
    public String homePage(Model model) {
        model.addAttribute("message", "Welcome!");
        return "home";   // → resolves to /templates/home.html
    }
}

// @RestController — Returns DATA directly (for REST APIs)
// = @Controller + @ResponseBody on every method
@RestController
public class ApiController {

    @GetMapping("/api/users")
    public List<User> getUsers() {
        return userService.findAll();   // → converted to JSON automatically
    }
}
```

| Annotation | Returns | Use Case |
|-----------|---------|----------|
| `@Controller` | View name (String) | Web pages (HTML) |
| `@RestController` | Object → JSON/XML | REST APIs |
| `@Controller` + `@ResponseBody` | Object → JSON/XML | Same as @RestController |

### 4.2 @RequestMapping & HTTP Method Annotations

```java
@Controller
@RequestMapping("/users")   // Base path for all methods in this controller
public class UserController {

    // GET /users
    @GetMapping
    public String listUsers(Model model) {
        model.addAttribute("users", userService.findAll());
        return "user/list";
    }

    // GET /users/123
    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        model.addAttribute("user", userService.findById(id));
        return "user/detail";
    }

    // GET /users/new
    @GetMapping("/new")
    public String showCreateForm(Model model) {
        model.addAttribute("user", new User());
        return "user/form";
    }

    // POST /users
    @PostMapping
    public String createUser(@ModelAttribute User user) {
        userService.save(user);
        return "redirect:/users";
    }

    // GET /users/123/edit
    @GetMapping("/{id}/edit")
    public String showEditForm(@PathVariable Long id, Model model) {
        model.addAttribute("user", userService.findById(id));
        return "user/form";
    }

    // PUT /users/123
    @PutMapping("/{id}")
    public String updateUser(@PathVariable Long id, @ModelAttribute User user) {
        user.setId(id);
        userService.save(user);
        return "redirect:/users";
    }

    // DELETE /users/123
    @DeleteMapping("/{id}")
    public String deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return "redirect:/users";
    }
}
```

### HTTP Method Shortcut Annotations
| Annotation | Equivalent |
|-----------|-----------|
| `@GetMapping("/path")` | `@RequestMapping(value="/path", method=RequestMethod.GET)` |
| `@PostMapping("/path")` | `@RequestMapping(value="/path", method=RequestMethod.POST)` |
| `@PutMapping("/path")` | `@RequestMapping(value="/path", method=RequestMethod.PUT)` |
| `@DeleteMapping("/path")` | `@RequestMapping(value="/path", method=RequestMethod.DELETE)` |
| `@PatchMapping("/path")` | `@RequestMapping(value="/path", method=RequestMethod.PATCH)` |

### 4.3 @RequestMapping Advanced Options
```java
@RequestMapping(
    value = "/users",
    method = RequestMethod.GET,
    params = "active=true",           // only if ?active=true
    headers = "X-API-Key",            // only if header present
    consumes = "application/json",    // only if Content-Type matches
    produces = "application/json"     // sets response Content-Type
)
public List<User> getActiveUsers() { ... }
```

### 4.4 Multiple URL Mappings
```java
@GetMapping({"/", "/home", "/index"})
public String home() {
    return "home";
}
```

---

## 5. REQUEST DATA BINDING

### 5.1 @PathVariable — URL Path Parameters
```java
// GET /users/42
@GetMapping("/users/{id}")
public String getUser(@PathVariable Long id) { ... }

// GET /users/42/orders/7
@GetMapping("/users/{userId}/orders/{orderId}")
public String getOrder(@PathVariable Long userId, @PathVariable Long orderId) { ... }

// Name mismatch — use value attribute
@GetMapping("/users/{user_id}")
public String getUser(@PathVariable("user_id") Long userId) { ... }

// Optional path variable
@GetMapping({"/users", "/users/{id}"})
public String getUser(@PathVariable(required = false) Long id) { ... }

// Regex in path variable
@GetMapping("/files/{filename:.+}")     // matches dots in filename
public String getFile(@PathVariable String filename) { ... }
```

### 5.2 @RequestParam — Query Parameters
```java
// GET /users?page=1&size=10
@GetMapping("/users")
public String listUsers(
    @RequestParam int page,
    @RequestParam int size
) { ... }

// With defaults
@GetMapping("/users")
public String listUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size,
    @RequestParam(required = false) String search
) { ... }

// Multiple values — GET /users?role=ADMIN&role=USER
@GetMapping("/users")
public String listUsers(@RequestParam List<String> role) { ... }

// All params as Map — GET /users?name=John&age=30
@GetMapping("/users")
public String listUsers(@RequestParam Map<String, String> params) { ... }
```

### 5.3 @RequestHeader — HTTP Headers
```java
@GetMapping("/info")
public String info(
    @RequestHeader("User-Agent") String userAgent,
    @RequestHeader("Accept-Language") String lang,
    @RequestHeader(value = "Authorization", required = false) String auth
) { ... }

// All headers
@GetMapping("/info")
public String info(@RequestHeader Map<String, String> headers) { ... }

// Using HttpHeaders
@GetMapping("/info")
public String info(@RequestHeader HttpHeaders headers) {
    String contentType = headers.getFirst("Content-Type");
    ...
}
```

### 5.4 @CookieValue — Cookies
```java
@GetMapping("/profile")
public String profile(
    @CookieValue("JSESSIONID") String sessionId,
    @CookieValue(value = "theme", defaultValue = "light") String theme
) { ... }
```

### 5.5 @RequestBody — JSON/XML Request Body
```java
// POST /api/users
// Body: {"name": "John", "email": "john@example.com"}
@PostMapping("/api/users")
@ResponseBody
public User createUser(@RequestBody User user) {
    return userService.save(user);
}

// With validation
@PostMapping("/api/users")
@ResponseBody
public User createUser(@Valid @RequestBody UserDTO dto) {
    return userService.create(dto);
}
```

### 5.6 @ModelAttribute — Form Data Binding
```java
// Binds form fields to object properties
// POST /users with form data: name=John&email=john@example.com&age=30
@PostMapping("/users")
public String createUser(@ModelAttribute("user") User user) {
    userService.save(user);
    return "redirect:/users";
}

// @ModelAttribute on method — runs BEFORE every handler method in controller
@ModelAttribute("roles")
public List<String> populateRoles() {
    return List.of("ADMIN", "USER", "MANAGER");
    // Available as ${roles} in ALL views rendered by this controller
}

// @ModelAttribute at class level
@ModelAttribute
public void addCommonAttributes(Model model) {
    model.addAttribute("appName", "My Application");
    model.addAttribute("currentYear", Year.now().getValue());
}
```

### 5.7 HttpServletRequest & HttpServletResponse (Direct Access)
```java
@GetMapping("/raw")
public void handleRaw(HttpServletRequest request, HttpServletResponse response) throws IOException {
    String ip = request.getRemoteAddr();
    String method = request.getMethod();
    String uri = request.getRequestURI();

    response.setContentType("text/plain");
    response.getWriter().write("IP: " + ip);
}
```

### 5.8 Flexible Method Signatures — All Supported Parameters
```java
@GetMapping("/demo")
public String demo(
    @PathVariable Long id,                    // URL path segment
    @RequestParam String name,                // Query parameter
    @RequestHeader String host,               // HTTP header
    @CookieValue String sessionId,            // Cookie
    @ModelAttribute UserForm form,            // Form data
    @RequestBody UserDTO body,                // JSON body
    Model model,                              // Add attributes for view
    HttpServletRequest request,               // Raw servlet request
    HttpServletResponse response,             // Raw servlet response
    HttpSession session,                      // HTTP session
    Principal principal,                      // Authenticated user
    Locale locale,                            // Client locale
    @RequestAttribute String attr,            // Request-scoped attribute
    @SessionAttribute String sessionAttr,     // Session-scoped attribute
    RedirectAttributes redirectAttrs,         // Flash attributes
    UriComponentsBuilder uriBuilder,          // URI builder
    Errors errors                             // Binding errors
) { ... }
```

### 5.9 @MatrixVariable — URL Matrix Variables
```java
// Matrix variables appear in URL segments separated by semicolons:
// /users/42;role=admin;active=true

// STEP 1 — Enable matrix variable support (disabled in Spring Boot by default)
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        UrlPathHelper urlPathHelper = new UrlPathHelper();
        urlPathHelper.setRemoveSemicolonContent(false);   // preserve ;key=value parts
        configurer.setUrlPathHelper(urlPathHelper);
    }
}

// STEP 2 — Use @MatrixVariable in controllers

// GET /users/42;role=admin;active=true
@GetMapping("/users/{userId}")
public String getUser(
    @PathVariable Long userId,
    @MatrixVariable String role,
    @MatrixVariable(required = false, defaultValue = "true") boolean active,
    Model model) {
    // userId=42, role="admin", active=true
    ...
}

// Multiple values: GET /users/42;tag=java;tag=spring
@GetMapping("/users/{id}")
public String getUser(@PathVariable Long id,
                      @MatrixVariable List<String> tag) {
    // tag = ["java", "spring"]
    ...
}

// Matrix variables from a specific path segment
// GET /users/42;color=red/orders/7;color=blue
@GetMapping("/users/{userId}/orders/{orderId}")
public String getOrder(
    @PathVariable Long userId, @PathVariable Long orderId,
    @MatrixVariable(name = "color", pathVar = "userId") String userColor,
    @MatrixVariable(name = "color", pathVar = "orderId") String orderColor) {
    // userColor="red", orderColor="blue"
    ...
}

// All matrix variables as a Map
@GetMapping("/users/{id}")
public String getUser(@PathVariable Long id,
                      @MatrixVariable Map<String, String> vars) {
    String role = vars.get("role");
    ...
}
```

### 5.10 @RequestPart — Multipart with Complex Data
```java
// Unlike @RequestParam (which treats parts as Strings or MultipartFile),
// @RequestPart deserialises a part using its declared Content-Type (e.g. JSON → object)

// Request: multipart/form-data
//   Part "user":   Content-Type: application/json  — {"name":"John","email":"j@x.com"}
//   Part "avatar": Content-Type: image/png          — [binary data]

@RestController
@RequestMapping("/api/users")
public class UserRestController {

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<UserDTO> createWithAvatar(
        @RequestPart("user")   @Valid CreateUserRequest user,   // JSON → object
        @RequestPart("avatar")        MultipartFile avatar      // file
    ) {
        UserDTO created = userService.create(user, avatar);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

| | `@RequestParam` | `@RequestPart` |
|--|--|--|
| Target types | `String`, primitives, `MultipartFile` | Any type (JSON/XML → object), `MultipartFile` |
| Part deserialization | As plain String | Via `HttpMessageConverter` (uses part's Content-Type) |
| Use case | Simple form fields / file upload | RESTful multipart with JSON metadata + file |

---

## 6. MODEL & PASSING DATA TO VIEW

### 6.1 Model Interface
```java
@GetMapping("/dashboard")
public String dashboard(Model model) {
    model.addAttribute("user", userService.getCurrentUser());
    model.addAttribute("stats", statsService.getDashboardStats());
    model.addAttribute("notifications", notificationService.getRecent());
    return "dashboard";
}
```

### 6.2 ModelMap (extends LinkedHashMap)
```java
@GetMapping("/dashboard")
public String dashboard(ModelMap model) {
    model.addAttribute("user", currentUser);
    model.put("count", 42);     // Map method also works
    return "dashboard";
}
```

### 6.3 ModelAndView (Combines Model + View)
```java
@GetMapping("/dashboard")
public ModelAndView dashboard() {
    ModelAndView mav = new ModelAndView("dashboard");   // view name
    mav.addObject("user", currentUser);
    mav.addObject("stats", dashboardStats);
    mav.setStatus(HttpStatus.OK);
    return mav;
}

// Setting view conditionally
@GetMapping("/profile")
public ModelAndView profile(@RequestParam Long id) {
    User user = userService.findById(id);
    if (user == null) {
        return new ModelAndView("error/404");
    }
    return new ModelAndView("user/profile", "user", user);
}
```

### 6.4 Map<String, Object> as Model
```java
@GetMapping("/info")
public String info(Map<String, Object> model) {
    model.put("title", "Info Page");
    return "info";
}
```

### 6.5 Redirect & Flash Attributes
```java
@PostMapping("/users")
public String createUser(@ModelAttribute User user, RedirectAttributes redirectAttrs) {
    userService.save(user);

    // Flash attributes survive ONE redirect, then disappear
    redirectAttrs.addFlashAttribute("successMessage", "User created!");

    // Regular attributes go into query string
    redirectAttrs.addAttribute("id", user.getId());

    return "redirect:/users/{id}";   // → /users/42
}

// Receiving flash attributes
@GetMapping("/users/{id}")
public String showUser(@PathVariable Long id, Model model) {
    // Flash attribute "successMessage" is automatically in model
    model.addAttribute("user", userService.findById(id));
    return "user/detail";
}
```

### 6.6 redirect: and forward: View Prefixes
```java
// ─── redirect: ──────────────────────────────────────────────────────────────
// Sends HTTP 302 to client → browser makes a BRAND NEW request to the new URL
// URL in browser address bar CHANGES
// Request attributes are LOST (use RedirectAttributes/flash to pass data)

return "redirect:/users";                       // relative to context root
return "redirect:/users/{id}";                  // supports URI template variables
return "redirect:https://external-site.com";    // external URL

// ─── forward: ───────────────────────────────────────────────────────────────
// Server-side internal forward — NO new HTTP request
// URL in browser address bar does NOT change
// Request attributes are PRESERVED (shared between source & destination)

return "forward:/internal/process";
return "forward:/error/403";
```

| | `redirect:` | `forward:` |
|--|--|--|
| New HTTP request | Yes (browser → server) | No (internal, server-side) |
| Browser URL changes | Yes | No |
| Request attributes | Lost | Preserved |
| Response type | HTTP 302 | Continues same request |
| Use case | After POST (PRG pattern), external URLs | Internal routing, sharing request data |

```java
// Classic POST-Redirect-GET (PRG) pattern using redirect:
@PostMapping("/users")
public String createUser(@Valid @ModelAttribute User user,
                         BindingResult result,
                         RedirectAttributes attr) {
    if (result.hasErrors()) return "user/form";
    userService.save(user);
    attr.addFlashAttribute("message", "User created!");
    return "redirect:/users";          // PRG — prevents double-submit on browser refresh
}

// forward: example — passing to another controller internally
@GetMapping("/admin/login")
public String adminLogin(HttpSession session) {
    if (session.getAttribute("adminUser") != null) {
        return "forward:/admin/dashboard";   // pass same request to dashboard handler
    }
    return "admin/login";
}
```

---

## 7. VIEW RESOLVERS & VIEW TECHNOLOGIES

### 7.1 ViewResolver Types
| Resolver | View Technology |
|----------|----------------|
| `InternalResourceViewResolver` | JSP |
| `ThymeleafViewResolver` | Thymeleaf |
| `FreeMarkerViewResolver` | FreeMarker |
| `ContentNegotiatingViewResolver` | Multiple (picks best based on Accept header) |
| `BeanNameViewResolver` | View beans by name |
| `XmlViewResolver` | XML-configured views |

### 7.2 InternalResourceViewResolver (JSP)
```java
@Bean
public ViewResolver viewResolver() {
    InternalResourceViewResolver resolver = new InternalResourceViewResolver();
    resolver.setPrefix("/WEB-INF/views/");
    resolver.setSuffix(".jsp");
    resolver.setOrder(1);
    return resolver;
}
// Controller returns "home" → /WEB-INF/views/home.jsp
```

### 7.3 Thymeleaf Integration (Spring Boot)
```properties
# application.properties (auto-configured in Spring Boot)
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html
spring.thymeleaf.mode=HTML
spring.thymeleaf.encoding=UTF-8
spring.thymeleaf.cache=false    # disable cache in dev
```

```html
<!-- templates/user/list.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title th:text="${pageTitle}">Default Title</title>
    <link rel="stylesheet" th:href="@{/css/style.css}"/>
</head>
<body>
    <h1 th:text="${heading}">Heading</h1>

    <!-- Conditionals -->
    <div th:if="${successMessage}" class="alert-success">
        <p th:text="${successMessage}">Message</p>
    </div>

    <!-- Iteration -->
    <table>
        <tr th:each="user, stat : ${users}">
            <td th:text="${stat.index + 1}">1</td>
            <td th:text="${user.name}">Name</td>
            <td th:text="${user.email}">Email</td>
            <td>
                <a th:href="@{/users/{id}(id=${user.id})}">View</a>
                <a th:href="@{/users/{id}/edit(id=${user.id})}">Edit</a>
            </td>
        </tr>
    </table>

    <!-- Forms -->
    <form th:action="@{/users}" th:object="${user}" method="post">
        <input type="text" th:field="*{name}"/>
        <span th:if="${#fields.hasErrors('name')}" th:errors="*{name}">Error</span>

        <input type="email" th:field="*{email}"/>
        <span th:if="${#fields.hasErrors('email')}" th:errors="*{email}">Error</span>

        <select th:field="*{role}">
            <option th:each="r : ${roles}" th:value="${r}" th:text="${r}">Role</option>
        </select>

        <button type="submit">Save</button>
    </form>

    <!-- Fragment inclusion -->
    <div th:replace="~{fragments/header :: header}"></div>
    <div th:insert="~{fragments/footer :: footer}"></div>

    <!-- JavaScript -->
    <script th:src="@{/js/app.js}"></script>
    <script th:inline="javascript">
        var userId = [[${user.id}]];
    </script>
</body>
</html>
```

### 7.4 JSP with JSTL
```xml
<!-- pom.xml dependencies for JSP -->
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>jstl</artifactId>
    <version>1.2</version>
</dependency>
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-jasper</artifactId>
    <scope>provided</scope>
</dependency>
```

```jsp
<%@ page contentType="text/html;charset=UTF-8" %>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ taglib uri="http://www.springframework.org/tags/form" prefix="form" %>

<html>
<body>
    <h1>${heading}</h1>

    <c:forEach var="user" items="${users}">
        <p>${user.name} — ${user.email}</p>
    </c:forEach>

    <!-- Spring Form Tags -->
    <form:form modelAttribute="user" action="/users" method="post">
        <form:input path="name"/>
        <form:errors path="name" cssClass="error"/>

        <form:input path="email"/>
        <form:errors path="email" cssClass="error"/>

        <form:select path="role" items="${roles}"/>

        <button type="submit">Save</button>
    </form:form>
</body>
</html>
```

### 7.5 Multiple View Resolvers (Chaining)
```java
@Bean
public ViewResolver thymeleafResolver() {
    ThymeleafViewResolver resolver = new ThymeleafViewResolver();
    resolver.setTemplateEngine(templateEngine());
    resolver.setOrder(1);   // checked first
    return resolver;
}

@Bean
public ViewResolver jspResolver() {
    InternalResourceViewResolver resolver = new InternalResourceViewResolver();
    resolver.setPrefix("/WEB-INF/views/");
    resolver.setSuffix(".jsp");
    resolver.setOrder(2);   // checked second (fallback)
    return resolver;
}
```

### 7.6 ContentNegotiatingViewResolver
```java
@Bean
public ViewResolver contentNegotiatingResolver(ContentNegotiationManager manager) {
    ContentNegotiatingViewResolver resolver = new ContentNegotiatingViewResolver();
    resolver.setContentNegotiationManager(manager);
    resolver.setViewResolvers(List.of(thymeleafResolver(), jsonViewResolver()));
    return resolver;
}
// Picks view based on Accept header: text/html → Thymeleaf, application/json → JSON
```

---

## 8. FORM HANDLING & DATA BINDING

### 8.1 Complete Form Processing Flow

```java
// Model / DTO
public class UserForm {
    private String name;
    private String email;
    private int age;
    private String role;
    private boolean active;
    private Date birthDate;
    private List<String> hobbies;
    // getters + setters
}
```

```java
@Controller
@RequestMapping("/users")
public class UserController {

    // 1. Show empty form
    @GetMapping("/new")
    public String showForm(Model model) {
        model.addAttribute("userForm", new UserForm());
        model.addAttribute("roles", List.of("ADMIN", "USER", "MANAGER"));
        return "user/form";
    }

    // 2. Process submitted form
    @PostMapping
    public String processForm(
        @Valid @ModelAttribute("userForm") UserForm form,
        BindingResult result,       // MUST come right after @ModelAttribute
        Model model,
        RedirectAttributes redirectAttrs
    ) {
        if (result.hasErrors()) {
            model.addAttribute("roles", List.of("ADMIN", "USER", "MANAGER"));
            return "user/form";    // re-render form with errors
        }
        userService.save(form);
        redirectAttrs.addFlashAttribute("message", "User saved!");
        return "redirect:/users";
    }
}
```

### 8.2 Data Binding Details

#### How Spring Binds Form Data → Object
```
1. Spring creates an instance of the target class (UserForm)
2. For each form field name, Spring calls the matching setter:
   - name=John         → form.setName("John")
   - email=j@test.com  → form.setEmail("j@test.com")
   - age=30            → form.setAge(30)        ← auto type conversion
   - active=true       → form.setActive(true)
   - hobbies=reading   → form.setHobbies(List.of("reading", "coding"))
   - hobbies=coding
3. If type conversion fails → BindingResult gets an error
```

#### Custom Type Conversion
```java
// Register custom converters in controller
@InitBinder
public void initBinder(WebDataBinder binder) {
    // Date format
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
    dateFormat.setLenient(false);
    binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, true));

    // Prevent binding of sensitive fields
    binder.setDisallowedFields("id", "password");

    // Whitelist allowed fields
    binder.setAllowedFields("name", "email", "age", "role");

    // Set validator
    binder.addValidators(new UserFormValidator());
}
```

#### Global @InitBinder (applies to all controllers)
```java
@ControllerAdvice
public class GlobalBindingInitializer {

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        StringTrimmerEditor trimmer = new StringTrimmerEditor(true);  // null if empty
        binder.registerCustomEditor(String.class, trimmer);
    }
}
```

### 8.3 PropertyEditor vs Converter vs Formatter

| Mechanism | Scope | Direction | Use Case |
|-----------|-------|-----------|----------|
| `PropertyEditor` | Legacy | String ↔ Object | Old Spring, per-controller |
| `Converter<S,T>` | Global | S → T (one-way) | Type conversion |
| `Formatter<T>` | Global | String ↔ T (two-way) | Locale-aware formatting |

```java
// Converter example
@Component
public class StringToRoleConverter implements Converter<String, Role> {
    @Override
    public Role convert(String source) {
        return Role.valueOf(source.toUpperCase());
    }
}

// Formatter example
@Component
public class DateFormatter implements Formatter<LocalDate> {
    @Override
    public LocalDate parse(String text, Locale locale) {
        return LocalDate.parse(text, DateTimeFormatter.ofPattern("dd/MM/yyyy"));
    }

    @Override
    public String print(LocalDate date, Locale locale) {
        return date.format(DateTimeFormatter.ofPattern("dd/MM/yyyy"));
    }
}

// Register converters/formatters
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToRoleConverter());
        registry.addFormatter(new DateFormatter());
    }
}
```

---

## 9. VALIDATION

### 9.1 Bean Validation (JSR-380) Annotations

```xml
<!-- Dependency (included in spring-boot-starter-web) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

```java
public class UserDTO {

    @NotNull(message = "ID is required")
    private Long id;

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 50, message = "Name must be 2-50 characters")
    private String name;

    @NotEmpty(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @Min(value = 18, message = "Minimum age is 18")
    @Max(value = 150, message = "Maximum age is 150")
    private int age;

    @Pattern(regexp = "^\\+?[0-9]{10,15}$", message = "Invalid phone number")
    private String phone;

    @Past(message = "Birth date must be in the past")
    private LocalDate birthDate;

    @Future(message = "Expiry date must be in the future")
    private LocalDate expiryDate;

    @Positive(message = "Salary must be positive")
    private BigDecimal salary;

    @DecimalMin(value = "0.0", inclusive = false)
    @DecimalMax(value = "100.0")
    private double score;

    @NotNull
    @Size(min = 1, message = "At least one role required")
    private List<@NotBlank String> roles;

    // Nested object validation
    @Valid
    @NotNull
    private Address address;

    // getters + setters
}
```

### All Standard Validation Annotations
| Annotation | Applies To | Description |
|-----------|-----------|-------------|
| `@NotNull` | Any | Not null |
| `@NotEmpty` | String, Collection, Map, Array | Not null and not empty |
| `@NotBlank` | String | Not null, not empty, not whitespace |
| `@Size(min, max)` | String, Collection, Map, Array | Size within bounds |
| `@Min(value)` | Numeric | Minimum value |
| `@Max(value)` | Numeric | Maximum value |
| `@Positive` | Numeric | > 0 |
| `@PositiveOrZero` | Numeric | >= 0 |
| `@Negative` | Numeric | < 0 |
| `@NegativeOrZero` | Numeric | <= 0 |
| `@Email` | String | Valid email |
| `@Pattern(regexp)` | String | Matches regex |
| `@Past` | Date/Time | In the past |
| `@PastOrPresent` | Date/Time | Past or now |
| `@Future` | Date/Time | In the future |
| `@FutureOrPresent` | Date/Time | Future or now |
| `@Digits(integer, fraction)` | Numeric | Digit constraints |
| `@DecimalMin` | Numeric | Min decimal value |
| `@DecimalMax` | Numeric | Max decimal value |
| `@AssertTrue` | boolean | Must be true |
| `@AssertFalse` | boolean | Must be false |

### 9.2 Using Validation in Controllers

```java
// For form-based (web pages)
@PostMapping("/users")
public String createUser(
    @Valid @ModelAttribute("userForm") UserDTO dto,
    BindingResult result,           // MUST be immediately after @Valid param
    Model model
) {
    if (result.hasErrors()) {
        return "user/form";         // re-show form with errors
    }
    userService.save(dto);
    return "redirect:/users";
}

// For REST APIs
@PostMapping("/api/users")
@ResponseBody
public ResponseEntity<User> createUser(@Valid @RequestBody UserDTO dto) {
    // If validation fails → Spring throws MethodArgumentNotValidException
    // Handle it with @ControllerAdvice (see Exception Handling section)
    User user = userService.save(dto);
    return ResponseEntity.status(HttpStatus.CREATED).body(user);
}
```

### 9.3 Custom Validator

```java
// Custom annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Validator implementation
@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    @Autowired
    private UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null) return true;   // let @NotNull handle null
        return !userRepository.existsByEmail(email);
    }
}

// Usage
public class UserDTO {
    @UniqueEmail
    @Email
    private String email;
}
```

### 9.4 Cross-Field Validation (Class-Level)
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchValidator.class)
public @interface PasswordMatch {
    String message() default "Passwords do not match";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class PasswordMatchValidator implements ConstraintValidator<PasswordMatch, RegistrationForm> {
    @Override
    public boolean isValid(RegistrationForm form, ConstraintValidatorContext ctx) {
        if (form.getPassword() == null) return true;
        return form.getPassword().equals(form.getConfirmPassword());
    }
}

@PasswordMatch
public class RegistrationForm {
    @NotBlank private String password;
    @NotBlank private String confirmPassword;
}
```

### 9.5 Validation Groups
```java
// Define marker interfaces
public interface OnCreate {}
public interface OnUpdate {}

public class UserDTO {
    @Null(groups = OnCreate.class, message = "ID must be null for creation")
    @NotNull(groups = OnUpdate.class, message = "ID required for update")
    private Long id;

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    private String name;
}

// Use in controller
@PostMapping
public String create(@Validated(OnCreate.class) @RequestBody UserDTO dto) { ... }

@PutMapping("/{id}")
public String update(@Validated(OnUpdate.class) @RequestBody UserDTO dto) { ... }
```

### 9.6 Manual / Programmatic Validation
```java
@Service
public class UserService {

    @Autowired
    private Validator validator;    // javax.validation.Validator

    public void validateUser(UserDTO dto) {
        Set<ConstraintViolation<UserDTO>> violations = validator.validate(dto);
        if (!violations.isEmpty()) {
            StringBuilder sb = new StringBuilder();
            for (ConstraintViolation<UserDTO> v : violations) {
                sb.append(v.getPropertyPath()).append(": ").append(v.getMessage()).append("\n");
            }
            throw new ValidationException(sb.toString());
        }
    }
}
```

---

## 10. EXCEPTION HANDLING

### 10.1 @ExceptionHandler (Per Controller)
```java
@Controller
public class UserController {

    @GetMapping("/users/{id}")
    public String getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        if (user == null) throw new UserNotFoundException("User not found: " + id);
        return "user/detail";
    }

    // Handles exceptions only in THIS controller
    @ExceptionHandler(UserNotFoundException.class)
    public String handleNotFound(UserNotFoundException ex, Model model) {
        model.addAttribute("error", ex.getMessage());
        return "error/404";
    }

    @ExceptionHandler(Exception.class)
    public String handleGeneral(Exception ex, Model model) {
        model.addAttribute("error", "Something went wrong");
        return "error/500";
    }
}
```

### 10.2 @ControllerAdvice (Global Exception Handler)
```java
@ControllerAdvice     // applies to ALL controllers
public class GlobalExceptionHandler {

    // 404 — Resource Not Found
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public String handleNotFound(ResourceNotFoundException ex, Model model) {
        model.addAttribute("error", ex.getMessage());
        return "error/404";
    }

    // 400 — Validation Errors (Form)
    @ExceptionHandler(BindException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public String handleValidation(BindException ex, Model model) {
        model.addAttribute("errors", ex.getFieldErrors());
        return "error/400";
    }

    // 500 — Generic Error
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public String handleGeneral(Exception ex, Model model) {
        model.addAttribute("error", "Internal Server Error");
        return "error/500";
    }
}
```

### 10.3 @RestControllerAdvice (For REST APIs)
```java
@RestControllerAdvice    // = @ControllerAdvice + @ResponseBody
public class ApiExceptionHandler {

    // Standard error response DTO
    record ErrorResponse(
        int status,
        String message,
        String path,
        LocalDateTime timestamp,
        Map<String, String> fieldErrors
    ) {}

    // 404
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        return new ErrorResponse(
            404, ex.getMessage(), request.getRequestURI(), LocalDateTime.now(), null
        );
    }

    // 400 — Validation Errors (JSON @RequestBody)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            fieldErrors.put(error.getField(), error.getDefaultMessage())
        );
        return new ErrorResponse(
            400, "Validation failed", request.getRequestURI(), LocalDateTime.now(), fieldErrors
        );
    }

    // 400 — Type mismatch (e.g., string where int expected)
    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleTypeMismatch(MethodArgumentTypeMismatchException ex, HttpServletRequest request) {
        String message = String.format("Parameter '%s' should be of type %s",
            ex.getName(), ex.getRequiredType().getSimpleName());
        return new ErrorResponse(
            400, message, request.getRequestURI(), LocalDateTime.now(), null
        );
    }

    // 405 — Method Not Allowed
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    @ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
    public ErrorResponse handleMethodNotAllowed(HttpRequestMethodNotSupportedException ex, HttpServletRequest request) {
        return new ErrorResponse(
            405, "Method " + ex.getMethod() + " not supported", request.getRequestURI(), LocalDateTime.now(), null
        );
    }

    // 415 — Unsupported Media Type
    @ExceptionHandler(HttpMediaTypeNotSupportedException.class)
    @ResponseStatus(HttpStatus.UNSUPPORTED_MEDIA_TYPE)
    public ErrorResponse handleMediaType(HttpMediaTypeNotSupportedException ex, HttpServletRequest request) {
        return new ErrorResponse(
            415, "Media type not supported: " + ex.getContentType(), request.getRequestURI(), LocalDateTime.now(), null
        );
    }

    // 500 — Catch-all
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleAll(Exception ex, HttpServletRequest request) {
        return new ErrorResponse(
            500, "Internal Server Error", request.getRequestURI(), LocalDateTime.now(), null
        );
    }
}
```

### 10.4 @ResponseStatus on Custom Exceptions
```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

@ResponseStatus(HttpStatus.CONFLICT)
public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) {
        super(message);
    }
}
// When thrown → Spring automatically sends the specified HTTP status
```

### 10.5 Custom Error Pages (Spring Boot)
```
src/main/resources/
├── templates/
│   └── error/
│       ├── 404.html    ← auto-used for 404 errors
│       ├── 403.html    ← auto-used for 403 errors
│       └── 500.html    ← auto-used for 500 errors
│   └── error.html      ← fallback for all errors
```

```properties
# application.properties
server.error.whitelabel.enabled=false
server.error.include-message=always
server.error.include-binding-errors=always
server.error.include-stacktrace=never    # NEVER in production
```

### 10.6 ResponseStatusException (Inline)
```java
@GetMapping("/users/{id}")
public String getUser(@PathVariable Long id) {
    User user = userService.findById(id);
    if (user == null) {
        throw new ResponseStatusException(HttpStatus.NOT_FOUND, "User not found: " + id);
    }
    return "user/detail";
}
```

---

## 11. REST CONTROLLERS & BUILDING APIs

### 11.1 Basic REST Controller
```java
@RestController
@RequestMapping("/api/v1/users")
public class UserRestController {

    private final UserService userService;

    public UserRestController(UserService userService) {
        this.userService = userService;
    }

    // GET /api/v1/users
    @GetMapping
    public List<UserDTO> getAll() {
        return userService.findAll();
    }

    // GET /api/v1/users/42
    @GetMapping("/{id}")
    public UserDTO getById(@PathVariable Long id) {
        return userService.findById(id);
    }

    // POST /api/v1/users
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserDTO create(@Valid @RequestBody CreateUserRequest request) {
        return userService.create(request);
    }

    // PUT /api/v1/users/42
    @PutMapping("/{id}")
    public UserDTO update(@PathVariable Long id, @Valid @RequestBody UpdateUserRequest request) {
        return userService.update(id, request);
    }

    // PATCH /api/v1/users/42
    @PatchMapping("/{id}")
    public UserDTO partialUpdate(@PathVariable Long id, @RequestBody Map<String, Object> updates) {
        return userService.partialUpdate(id, updates);
    }

    // DELETE /api/v1/users/42
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### 11.2 ResponseEntity — Full Control Over Response
```java
@RestController
@RequestMapping("/api/v1/users")
public class UserRestController {

    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getById(@PathVariable Long id) {
        UserDTO user = userService.findById(id);
        if (user == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(user);
    }

    @PostMapping
    public ResponseEntity<UserDTO> create(@Valid @RequestBody CreateUserRequest request) {
        UserDTO created = userService.create(request);
        URI location = URI.create("/api/v1/users/" + created.getId());
        return ResponseEntity.created(location).body(created);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }

    // Custom headers
    @GetMapping("/export")
    public ResponseEntity<byte[]> exportCsv() {
        byte[] csvData = userService.exportCsv();
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=users.csv")
            .contentType(MediaType.parseMediaType("text/csv"))
            .body(csvData);
    }
}
```

### ResponseEntity Builder Methods
| Method | HTTP Status |
|--------|-------------|
| `ResponseEntity.ok(body)` | 200 OK |
| `ResponseEntity.created(uri).body(body)` | 201 Created |
| `ResponseEntity.accepted().build()` | 202 Accepted |
| `ResponseEntity.noContent().build()` | 204 No Content |
| `ResponseEntity.badRequest().body(errors)` | 400 Bad Request |
| `ResponseEntity.notFound().build()` | 404 Not Found |
| `ResponseEntity.status(HttpStatus.CONFLICT).body(msg)` | 409 Conflict |
| `ResponseEntity.internalServerError().body(msg)` | 500 ISE |

### 11.3 Jackson JSON Configuration

```java
// DTO with Jackson annotations
public class UserDTO {

    @JsonProperty("user_id")        // custom JSON field name
    private Long id;

    private String name;

    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate birthDate;

    @JsonIgnore                     // exclude from JSON
    private String password;

    @JsonInclude(JsonInclude.Include.NON_NULL)   // skip if null
    private String middleName;

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)   // only in response
    private LocalDateTime createdAt;

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)  // only in request
    private String secret;
}
```

```java
// Global Jackson configuration
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return mapper;
    }
}
```

```properties
# Or via application.properties
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=UTC
spring.jackson.serialization.write-dates-as-timestamps=false
spring.jackson.default-property-inclusion=non_null
spring.jackson.deserialization.fail-on-unknown-properties=false
```

### 11.4 Content Negotiation
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(true)           // ?format=json
            .parameterName("format")
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML);
    }
}
```

```xml
<!-- For XML support, add Jackson XML module -->
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

### 11.5 Pagination & Sorting (with Spring Data)
```java
@GetMapping
public Page<UserDTO> getUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size,
    @RequestParam(defaultValue = "id") String sortBy,
    @RequestParam(defaultValue = "asc") String direction
) {
    Sort sort = direction.equalsIgnoreCase("desc")
        ? Sort.by(sortBy).descending()
        : Sort.by(sortBy).ascending();
    Pageable pageable = PageRequest.of(page, size, sort);
    return userService.findAll(pageable);
}

// Or use Spring Data's Pageable directly
@GetMapping
public Page<UserDTO> getUsers(Pageable pageable) {
    return userService.findAll(pageable);
}
// GET /api/users?page=0&size=10&sort=name,asc
```

### 11.6 HATEOAS (Hypermedia)
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public EntityModel<UserDTO> getUser(@PathVariable Long id) {
        UserDTO user = userService.findById(id);
        return EntityModel.of(user,
            linkTo(methodOn(UserController.class).getUser(id)).withSelfRel(),
            linkTo(methodOn(UserController.class).getAll()).withRel("users"),
            linkTo(methodOn(OrderController.class).getOrdersByUser(id)).withRel("orders")
        );
    }
}
// Response:
// {
//   "id": 1,
//   "name": "John",
//   "_links": {
//     "self": { "href": "/api/users/1" },
//     "users": { "href": "/api/users" },
//     "orders": { "href": "/api/users/1/orders" }
//   }
// }
```

---

## 12. INTERCEPTORS

### 12.1 HandlerInterceptor Interface
```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(LoggingInterceptor.class);

    // Runs BEFORE controller method
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        log.info("Request: {} {} from {}",
            request.getMethod(), request.getRequestURI(), request.getRemoteAddr());
        request.setAttribute("startTime", System.currentTimeMillis());
        return true;   // true = continue, false = stop (don't call controller)
    }

    // Runs AFTER controller method (before view rendering)
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) throws Exception {
        if (modelAndView != null) {
            modelAndView.addObject("serverTime", LocalDateTime.now());
        }
    }

    // Runs AFTER everything (including view rendering) — always called (even on error)
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
        long startTime = (Long) request.getAttribute("startTime");
        long duration = System.currentTimeMillis() - startTime;
        log.info("Response: {} {} → {} ({} ms)",
            request.getMethod(), request.getRequestURI(), response.getStatus(), duration);
    }
}
```

### 12.2 Register Interceptors
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private LoggingInterceptor loggingInterceptor;

    @Autowired
    private AuthInterceptor authInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // Apply to all paths
        registry.addInterceptor(loggingInterceptor)
                .addPathPatterns("/**");

        // Apply to specific paths, exclude some
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/admin/**", "/api/**")
                .excludePathPatterns("/api/public/**", "/login", "/register");

        // Order matters — lower number = runs first
        registry.addInterceptor(loggingInterceptor).order(1);
        registry.addInterceptor(authInterceptor).order(2);
    }
}
```

### 12.3 Common Interceptor Use Cases

#### Authentication Interceptor
```java
@Component
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        HttpSession session = request.getSession(false);
        if (session == null || session.getAttribute("user") == null) {
            response.sendRedirect("/login");
            return false;
        }
        return true;
    }
}
```

#### Rate Limiting Interceptor
```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {

    private final Map<String, AtomicInteger> requestCounts = new ConcurrentHashMap<>();

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String ip = request.getRemoteAddr();
        AtomicInteger count = requestCounts.computeIfAbsent(ip, k -> new AtomicInteger(0));
        if (count.incrementAndGet() > 100) {   // max 100 requests
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.getWriter().write("Rate limit exceeded");
            return false;
        }
        return true;
    }
}
```

### 12.4 Interceptor vs Filter
| Feature | Filter (Servlet) | Interceptor (Spring MVC) |
|---------|------------------|--------------------------|
| Level | Servlet container | Spring MVC |
| Access to Spring beans | No (unless WebApplicationContextUtils) | Yes (can @Autowire) |
| Access to handler info | No | Yes (handler parameter) |
| Access to ModelAndView | No | Yes (postHandle) |
| `preHandle` return | N/A | Can stop execution |
| Runs for | All requests (including static) | Only Spring MVC handler requests |
| Order | web.xml / @Order | InterceptorRegistry order |
| Use case | Request/response wrapping, encoding | Auth, logging, model enrichment |

---

## 13. FILTERS

### 13.1 Servlet Filters in Spring
```java
@Component
@Order(1)    // lower = runs first
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        // Before
        long start = System.currentTimeMillis();
        log.info("Incoming: {} {}", request.getMethod(), request.getRequestURI());

        // Continue filter chain
        filterChain.doFilter(request, response);

        // After
        long duration = System.currentTimeMillis() - start;
        log.info("Outgoing: {} {} → {} ({} ms)",
            request.getMethod(), request.getRequestURI(), response.getStatus(), duration);
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        return request.getRequestURI().startsWith("/static/");
    }
}
```

### 13.2 Register Filter with FilterRegistrationBean
```java
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<CorsFilter> corsFilter() {
        FilterRegistrationBean<CorsFilter> registration = new FilterRegistrationBean<>();
        registration.setFilter(new CorsFilter());
        registration.addUrlPatterns("/api/*");
        registration.setOrder(1);
        return registration;
    }
}
```

---

## 14. CORS (Cross-Origin Resource Sharing)

### 14.1 @CrossOrigin (Per Controller/Method)
```java
@RestController
@RequestMapping("/api/users")
@CrossOrigin(origins = "http://localhost:3000")   // Allow specific origin
public class UserController {

    @GetMapping
    @CrossOrigin(origins = "*", maxAge = 3600)    // override at method level
    public List<User> getAll() { ... }
}
```

### 14.2 Global CORS Configuration
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:3000", "https://myapp.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
                .allowedHeaders("*")
                .exposedHeaders("Authorization", "X-Total-Count")
                .allowCredentials(true)
                .maxAge(3600);

        // Different CORS for public API
        registry.addMapping("/public/**")
                .allowedOrigins("*")
                .allowedMethods("GET");
    }
}
```

### 14.3 CORS Filter (Lowest Level)
```java
@Bean
public CorsFilter corsFilter() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("http://localhost:3000"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return new CorsFilter(source);
}
```

---

## 15. STATIC RESOURCES

### 15.1 Default Locations (Spring Boot)
```
src/main/resources/
├── static/           ← default static resource folder
│   ├── css/
│   │   └── style.css         → /css/style.css
│   ├── js/
│   │   └── app.js            → /js/app.js
│   └── images/
│       └── logo.png          → /images/logo.png
├── public/            ← also served
├── resources/         ← also served
└── META-INF/resources/← also served
```

Priority: `META-INF/resources` > `resources` > `static` > `public`

### 15.2 Custom Static Resource Configuration
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // /assets/** → classpath:/custom-static/
        registry.addResourceHandler("/assets/**")
                .addResourceLocations("classpath:/custom-static/")
                .setCachePeriod(3600)
                .resourceChain(true)
                .addResolver(new VersionResourceResolver()
                    .addContentVersionStrategy("/**"));

        // External folder
        registry.addResourceHandler("/uploads/**")
                .addResourceLocations("file:///C:/uploads/");
    }
}
```

```properties
# application.properties
spring.web.resources.static-locations=classpath:/static/,classpath:/public/,file:./uploads/
spring.web.resources.cache.period=3600
spring.web.resources.chain.strategy.content.enabled=true
spring.web.resources.chain.strategy.content.paths=/**
```

---

## 16. FILE UPLOAD & DOWNLOAD

### 16.1 File Upload

```properties
# application.properties
spring.servlet.multipart.enabled=true
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=50MB
spring.servlet.multipart.file-size-threshold=2KB
```

```java
@Controller
@RequestMapping("/files")
public class FileController {

    private final Path uploadDir = Paths.get("uploads");

    @PostConstruct
    public void init() throws IOException {
        Files.createDirectories(uploadDir);
    }

    // Single file upload
    @PostMapping("/upload")
    public String uploadFile(@RequestParam("file") MultipartFile file,
                             RedirectAttributes redirectAttrs) throws IOException {
        if (file.isEmpty()) {
            redirectAttrs.addFlashAttribute("error", "Please select a file");
            return "redirect:/files";
        }

        // Validate file type
        String contentType = file.getContentType();
        if (!contentType.startsWith("image/")) {
            redirectAttrs.addFlashAttribute("error", "Only images allowed");
            return "redirect:/files";
        }

        // Save file
        String filename = UUID.randomUUID() + "_" + StringUtils.cleanPath(file.getOriginalFilename());
        Path targetPath = uploadDir.resolve(filename);
        Files.copy(file.getInputStream(), targetPath, StandardCopyOption.REPLACE_EXISTING);

        redirectAttrs.addFlashAttribute("success", "File uploaded: " + filename);
        return "redirect:/files";
    }

    // Multiple file upload
    @PostMapping("/upload-multiple")
    public String uploadMultiple(@RequestParam("files") MultipartFile[] files) throws IOException {
        for (MultipartFile file : files) {
            if (!file.isEmpty()) {
                String filename = UUID.randomUUID() + "_" + file.getOriginalFilename();
                Files.copy(file.getInputStream(), uploadDir.resolve(filename));
            }
        }
        return "redirect:/files";
    }
}
```

```html
<!-- Thymeleaf upload form -->
<form th:action="@{/files/upload}" method="post" enctype="multipart/form-data">
    <input type="file" name="file" accept="image/*"/>
    <button type="submit">Upload</button>
</form>

<!-- Multiple files -->
<form th:action="@{/files/upload-multiple}" method="post" enctype="multipart/form-data">
    <input type="file" name="files" multiple/>
    <button type="submit">Upload All</button>
</form>
```

### 16.2 File Upload via REST API
```java
@RestController
@RequestMapping("/api/files")
public class FileRestController {

    @PostMapping("/upload")
    public ResponseEntity<Map<String, String>> upload(@RequestParam("file") MultipartFile file) throws IOException {
        String filename = fileService.store(file);
        Map<String, String> response = Map.of(
            "filename", filename,
            "size", String.valueOf(file.getSize()),
            "type", file.getContentType()
        );
        return ResponseEntity.ok(response);
    }
}
```

### 16.3 File Download
```java
@GetMapping("/download/{filename}")
public ResponseEntity<Resource> downloadFile(@PathVariable String filename) throws IOException {
    Path filePath = uploadDir.resolve(filename).normalize();

    // Security: prevent path traversal
    if (!filePath.startsWith(uploadDir)) {
        return ResponseEntity.badRequest().build();
    }

    Resource resource = new UrlResource(filePath.toUri());
    if (!resource.exists()) {
        return ResponseEntity.notFound().build();
    }

    String contentType = Files.probeContentType(filePath);
    if (contentType == null) contentType = "application/octet-stream";

    return ResponseEntity.ok()
        .contentType(MediaType.parseMediaType(contentType))
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + resource.getFilename() + "\"")
        .body(resource);
}

// Inline display (in browser, e.g., images)
@GetMapping("/view/{filename}")
public ResponseEntity<Resource> viewFile(@PathVariable String filename) throws IOException {
    // Same as above but:
    .header(HttpHeaders.CONTENT_DISPOSITION, "inline; filename=\"" + resource.getFilename() + "\"")
}
```

---

## 17. SESSION MANAGEMENT

### 17.1 HttpSession
```java
@Controller
public class SessionController {

    // Store in session
    @PostMapping("/login")
    public String login(@ModelAttribute LoginForm form, HttpSession session) {
        User user = authService.authenticate(form);
        session.setAttribute("currentUser", user);
        session.setAttribute("loginTime", LocalDateTime.now());
        session.setMaxInactiveInterval(30 * 60);   // 30 minutes
        return "redirect:/dashboard";
    }

    // Read from session
    @GetMapping("/dashboard")
    public String dashboard(HttpSession session, Model model) {
        User user = (User) session.getAttribute("currentUser");
        if (user == null) return "redirect:/login";
        model.addAttribute("user", user);
        return "dashboard";
    }

    // Remove from session
    @PostMapping("/logout")
    public String logout(HttpSession session) {
        session.invalidate();
        return "redirect:/login";
    }
}
```

### 17.2 @SessionAttributes (Scoped to Controller)
```java
@Controller
@SessionAttributes("wizard")   // keeps "wizard" in session across requests
public class WizardController {

    @ModelAttribute("wizard")
    public WizardForm setupWizard() {
        return new WizardForm();   // initial setup
    }

    @GetMapping("/wizard/step1")
    public String step1(@ModelAttribute("wizard") WizardForm wizard) {
        return "wizard/step1";
    }

    @PostMapping("/wizard/step1")
    public String processStep1(@ModelAttribute("wizard") WizardForm wizard) {
        return "redirect:/wizard/step2";
    }

    @GetMapping("/wizard/step2")
    public String step2(@ModelAttribute("wizard") WizardForm wizard) {
        return "wizard/step2";
    }

    @PostMapping("/wizard/step2")
    public String processStep2(@ModelAttribute("wizard") WizardForm wizard, SessionStatus status) {
        wizardService.complete(wizard);
        status.setComplete();   // clears session attributes
        return "redirect:/wizard/done";
    }
}
```

### 17.3 @SessionAttribute (Read-Only, from Session)
```java
@GetMapping("/profile")
public String profile(@SessionAttribute("currentUser") User user, Model model) {
    // Reads "currentUser" from session (throws if missing)
    model.addAttribute("user", user);
    return "profile";
}

@GetMapping("/profile")
public String profile(@SessionAttribute(name = "currentUser", required = false) User user) {
    if (user == null) return "redirect:/login";
    ...
}
```

### 17.4 Session Properties (Spring Boot)
```properties
server.servlet.session.timeout=30m
server.servlet.session.cookie.name=MYSESSIONID
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.secure=true
server.servlet.session.cookie.same-site=strict
server.servlet.session.tracking-modes=cookie
```

---

## 18. INTERNATIONALIZATION (i18n)

### 18.1 Message Source Configuration
```java
@Configuration
public class I18nConfig {

    @Bean
    public MessageSource messageSource() {
        ResourceBundleMessageSource source = new ResourceBundleMessageSource();
        source.setBasename("messages");         // messages.properties
        source.setDefaultEncoding("UTF-8");
        source.setUseCodeAsDefaultMessage(true);
        return source;
    }

    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver = new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.US);
        return resolver;
    }

    @Bean
    public LocaleChangeInterceptor localeInterceptor() {
        LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
        interceptor.setParamName("lang");    // ?lang=fr
        return interceptor;
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private LocaleChangeInterceptor localeInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeInterceptor);
    }
}
```

### 18.2 Message Property Files
```properties
# messages.properties (default — English)
greeting=Hello
user.name=Name
user.email=Email
user.save=Save User
error.required=This field is required

# messages_fr.properties (French)
greeting=Bonjour
user.name=Nom
user.email=E-mail
user.save=Enregistrer l'utilisateur
error.required=Ce champ est obligatoire

# messages_es.properties (Spanish)
greeting=Hola
user.name=Nombre
user.email=Correo electrónico
user.save=Guardar usuario
error.required=Este campo es obligatorio
```

### 18.3 Using Messages

```html
<!-- Thymeleaf -->
<h1 th:text="#{greeting}">Hello</h1>
<label th:text="#{user.name}">Name</label>

<!-- With parameters -->
<!-- messages.properties: welcome=Welcome, {0}! You have {1} messages. -->
<p th:text="#{welcome(${user.name}, ${messageCount})}">Welcome!</p>
```

```java
// In controller
@Autowired
private MessageSource messageSource;

@GetMapping("/hello")
public String hello(Locale locale, Model model) {
    String greeting = messageSource.getMessage("greeting", null, locale);
    model.addAttribute("greeting", greeting);
    return "hello";
}
```

---

## 19. ASYNC REQUEST PROCESSING

### 19.1 Enable Async Support
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

### 19.2 Callable (Async Controller)
```java
@GetMapping("/async/report")
public Callable<String> generateReport() {
    // Releases servlet thread immediately
    return () -> {
        // Long-running task runs in separate thread
        Thread.sleep(3000);
        return "report";   // view name
    };
}
```

### 19.3 DeferredResult
```java
@GetMapping("/async/notification")
public DeferredResult<ResponseEntity<String>> waitForNotification() {
    DeferredResult<ResponseEntity<String>> result = new DeferredResult<>(30000L);   // 30s timeout

    result.onTimeout(() ->
        result.setErrorResult(ResponseEntity.status(HttpStatus.REQUEST_TIMEOUT).body("Timeout"))
    );

    // Some event/message listener will set the result later
    notificationService.addListener(notification ->
        result.setResult(ResponseEntity.ok(notification))
    );

    return result;
}
```

### 19.4 StreamingResponseBody (Large File Streaming)
```java
@GetMapping("/download/large")
public ResponseEntity<StreamingResponseBody> downloadLargeFile() {
    StreamingResponseBody stream = outputStream -> {
        // Write directly to response output stream — no memory buffering
        for (int i = 0; i < 1000; i++) {
            String data = "Line " + i + "\n";
            outputStream.write(data.getBytes());
            outputStream.flush();
        }
    };

    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=large.txt")
        .body(stream);
}
```

### 19.5 SSE (Server-Sent Events)
```java
@GetMapping(value = "/stream/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamEvents() {
    SseEmitter emitter = new SseEmitter(60_000L);   // 60s timeout

    CompletableFuture.runAsync(() -> {
        try {
            for (int i = 0; i < 10; i++) {
                emitter.send(SseEmitter.event()
                    .id(String.valueOf(i))
                    .name("update")
                    .data("Event " + i));
                Thread.sleep(1000);
            }
            emitter.complete();
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    });

    return emitter;
}
```

---

## 20. WEBMVCCONFIGURER — FULL CUSTOMIZATION

### All Override Methods
```java
@Configuration
@EnableWebMvc    // for non-Spring-Boot, or to override Boot's auto-config
public class WebConfig implements WebMvcConfigurer {

    // View Resolvers
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.jsp("/WEB-INF/views/", ".jsp");
    }

    // Static Resources
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/");
    }

    // Default Servlet (serves static files from servlet container)
    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }

    // Interceptors
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor());
    }

    // CORS
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**").allowedOrigins("*");
    }

    // Converters & Formatters
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToEnumConverter());
    }

    // Message Converters (JSON, XML, etc.)
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new MappingJackson2HttpMessageConverter());
    }

    // Extend (don't replace) message converters
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        // Add without clearing defaults
    }

    // Content Negotiation
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer.defaultContentType(MediaType.APPLICATION_JSON);
    }

    // Argument Resolvers (custom method parameters)
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new CurrentUserArgumentResolver());
    }

    // Return Value Handlers
    @Override
    public void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers) {
        handlers.add(new CsvReturnValueHandler());
    }

    // View Controllers (no logic, just view name)
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("home");
        registry.addViewController("/about").setViewName("about");
        registry.addViewController("/login").setViewName("login");
        registry.addRedirectViewController("/old-path", "/new-path");
        registry.addStatusController("/health", HttpStatus.OK);
    }

    // Path Matching
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.setUseTrailingSlashMatch(false);    // /users != /users/
    }

    // Async Support
    @Override
    public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
        configurer.setDefaultTimeout(30_000);
        configurer.setTaskExecutor(new ConcurrentTaskExecutor());
    }
}
```

---

## 21. CUSTOM ARGUMENT RESOLVERS

### 21.1 Creating a Custom Argument Resolver
```java
// Custom annotation
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface CurrentUser {}

// Resolver implementation
@Component
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CurrentUser.class)
            && parameter.getParameterType().equals(User.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
        HttpServletRequest request = (HttpServletRequest) webRequest.getNativeRequest();
        HttpSession session = request.getSession(false);
        if (session != null) {
            return session.getAttribute("currentUser");
        }
        return null;
    }
}

// Register
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new CurrentUserArgumentResolver());
    }
}

// Usage in controller
@GetMapping("/profile")
public String profile(@CurrentUser User user, Model model) {
    model.addAttribute("user", user);
    return "profile";
}
```

---

## 22. TESTING SPRING MVC

### 22.1 MockMvc — Integration Testing

```java
@WebMvcTest(UserController.class)   // loads only MVC layer
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean   // mock the service dependency
    private UserService userService;

    @Test
    void shouldReturnUserList() throws Exception {
        // Given
        List<User> users = List.of(new User(1L, "John"), new User(2L, "Jane"));
        when(userService.findAll()).thenReturn(users);

        // When & Then
        mockMvc.perform(get("/users"))
            .andExpect(status().isOk())
            .andExpect(view().name("user/list"))
            .andExpect(model().attribute("users", hasSize(2)))
            .andExpect(model().attribute("users", hasItem(
                hasProperty("name", is("John"))
            )));
    }

    @Test
    void shouldReturnUserById() throws Exception {
        when(userService.findById(1L)).thenReturn(new User(1L, "John"));

        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(view().name("user/detail"))
            .andExpect(model().attributeExists("user"));
    }

    @Test
    void shouldCreateUser() throws Exception {
        mockMvc.perform(post("/users")
                .param("name", "John")
                .param("email", "john@test.com")
                .with(csrf()))
            .andExpect(status().is3xxRedirection())
            .andExpect(redirectedUrl("/users"));
    }

    @Test
    void shouldReturnValidationErrors() throws Exception {
        mockMvc.perform(post("/users")
                .param("name", "")    // blank → validation error
                .param("email", "invalid")
                .with(csrf()))
            .andExpect(status().isOk())
            .andExpect(view().name("user/form"))
            .andExpect(model().attributeHasFieldErrors("userForm", "name", "email"));
    }
}
```

### 22.2 REST API Testing with MockMvc
```java
@WebMvcTest(UserRestController.class)
class UserRestControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUserJson() throws Exception {
        UserDTO user = new UserDTO(1L, "John", "john@test.com");
        when(userService.findById(1L)).thenReturn(user);

        mockMvc.perform(get("/api/v1/users/1")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$.name").value("John"))
            .andExpect(jsonPath("$.email").value("john@test.com"));
    }

    @Test
    void shouldCreateUser() throws Exception {
        CreateUserRequest request = new CreateUserRequest("John", "john@test.com");
        UserDTO created = new UserDTO(1L, "John", "john@test.com");
        when(userService.create(any())).thenReturn(created);

        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("John"));
    }

    @Test
    void shouldReturn404WhenNotFound() throws Exception {
        when(userService.findById(99L)).thenReturn(null);

        mockMvc.perform(get("/api/v1/users/99"))
            .andExpect(status().isNotFound());
    }

    @Test
    void shouldReturn400ForInvalidInput() throws Exception {
        CreateUserRequest request = new CreateUserRequest("", "");   // invalid

        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.fieldErrors.name").exists())
            .andExpect(jsonPath("$.fieldErrors.email").exists());
    }
}
```

### 22.3 Full Integration Test
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCRUDUser() {
        // Create
        CreateUserRequest request = new CreateUserRequest("John", "john@test.com");
        ResponseEntity<UserDTO> createResponse = restTemplate.postForEntity(
            "/api/v1/users", request, UserDTO.class);
        assertEquals(HttpStatus.CREATED, createResponse.getStatusCode());
        Long id = createResponse.getBody().getId();

        // Read
        ResponseEntity<UserDTO> getResponse = restTemplate.getForEntity(
            "/api/v1/users/" + id, UserDTO.class);
        assertEquals("John", getResponse.getBody().getName());

        // Update
        UpdateUserRequest update = new UpdateUserRequest("Jane", "jane@test.com");
        restTemplate.put("/api/v1/users/" + id, update);

        // Delete
        restTemplate.delete("/api/v1/users/" + id);

        // Verify deleted
        ResponseEntity<UserDTO> after = restTemplate.getForEntity(
            "/api/v1/users/" + id, UserDTO.class);
        assertEquals(HttpStatus.NOT_FOUND, after.getStatusCode());
    }
}
```

### 22.4 WebTestClient (WebFlux compatible)
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserWebTestClientTest {

    @Autowired
    private WebTestClient webClient;

    @Test
    void shouldGetUsers() {
        webClient.get().uri("/api/v1/users")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk()
            .expectHeader().contentType(MediaType.APPLICATION_JSON)
            .expectBodyList(UserDTO.class).hasSize(5);
    }
}
```

---

## 23. SECURITY BASICS (Spring Security Integration)

### 23.1 Add Spring Security
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### 23.2 Basic Security Configuration
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/home", "/public/**", "/css/**", "/js/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout")
                .permitAll()
            )
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/api/**")   // disable CSRF for REST API
            );

        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.withDefaultPasswordEncoder()
            .username("user")
            .password("password")
            .roles("USER")
            .build();
        UserDetails admin = User.withDefaultPasswordEncoder()
            .username("admin")
            .password("admin")
            .roles("ADMIN")
            .build();
        return new InMemoryUserDetailsManager(user, admin);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 23.3 Access Current User in Controller
```java
@GetMapping("/profile")
public String profile(
    @AuthenticationPrincipal UserDetails userDetails,   // Spring Security user
    Principal principal,                                 // Java security principal
    Model model
) {
    model.addAttribute("username", userDetails.getUsername());
    model.addAttribute("roles", userDetails.getAuthorities());
    return "profile";
}

// Or programmatically
SecurityContextHolder.getContext().getAuthentication().getName();
```

---

## 24. COMMON PATTERNS & BEST PRACTICES

### 24.1 Layered Architecture
```
Controller  ←→  Service  ←→  Repository  ←→  Database
    │               │              │
    ▼               ▼              ▼
  DTO            Entity        Entity
(UserDTO)       (User)         (User)
```

```java
// Controller — HTTP concern only
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    private final UserService userService;
    // Constructor injection (no @Autowired needed for single constructor)
    public UserController(UserService userService) {
        this.userService = userService;
    }
    @GetMapping
    public List<UserDTO> getAll() { return userService.findAll(); }
}

// Service — Business logic
@Service
@Transactional
public class UserService {
    private final UserRepository userRepository;
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    public List<UserDTO> findAll() {
        return userRepository.findAll().stream()
            .map(this::toDTO)
            .toList();
    }
}

// Repository — Data access
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
}
```

### 24.2 DTO Pattern
```java
// Request DTO
public record CreateUserRequest(
    @NotBlank String name,
    @Email @NotBlank String email,
    @Min(18) int age
) {}

// Response DTO
public record UserResponse(
    Long id,
    String name,
    String email,
    LocalDateTime createdAt
) {}

// Mapping (manual)
private UserResponse toResponse(User user) {
    return new UserResponse(user.getId(), user.getName(), user.getEmail(), user.getCreatedAt());
}

// Or use MapStruct
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserResponse toResponse(User user);
    User toEntity(CreateUserRequest request);
}
```

### 24.3 API Versioning Strategies
```java
// 1. URI versioning (most common)
@RequestMapping("/api/v1/users")
@RequestMapping("/api/v2/users")

// 2. Header versioning
@GetMapping(value = "/api/users", headers = "X-API-VERSION=1")
@GetMapping(value = "/api/users", headers = "X-API-VERSION=2")

// 3. Parameter versioning
@GetMapping(value = "/api/users", params = "version=1")
@GetMapping(value = "/api/users", params = "version=2")

// 4. Media type versioning (content negotiation)
@GetMapping(value = "/api/users", produces = "application/vnd.myapp.v1+json")
@GetMapping(value = "/api/users", produces = "application/vnd.myapp.v2+json")
```

### 24.4 Response Wrapper Pattern
```java
public record ApiResponse<T>(
    boolean success,
    String message,
    T data,
    LocalDateTime timestamp
) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, "Success", data, LocalDateTime.now());
    }
    public static <T> ApiResponse<T> ok(String message, T data) {
        return new ApiResponse<>(true, message, data, LocalDateTime.now());
    }
    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, message, null, LocalDateTime.now());
    }
}

@GetMapping("/{id}")
public ResponseEntity<ApiResponse<UserDTO>> getUser(@PathVariable Long id) {
    UserDTO user = userService.findById(id);
    return ResponseEntity.ok(ApiResponse.ok(user));
}
// Response: {"success":true,"message":"Success","data":{"id":1,"name":"John"},"timestamp":"..."}
```

---

## 25. IMPORTANT ANNOTATIONS REFERENCE

### Controller Annotations
| Annotation | Purpose |
|-----------|---------|
| `@Controller` | Marks class as MVC controller (returns views) |
| `@RestController` | = @Controller + @ResponseBody (returns data) |
| `@RequestMapping` | Maps URL to class/method |
| `@GetMapping` | Shortcut for GET requests |
| `@PostMapping` | Shortcut for POST requests |
| `@PutMapping` | Shortcut for PUT requests |
| `@DeleteMapping` | Shortcut for DELETE requests |
| `@PatchMapping` | Shortcut for PATCH requests |

### Parameter Annotations
| Annotation | Source |
|-----------|--------|
| `@PathVariable` | URL path: /users/{id} |
| `@RequestParam` | Query string: ?key=value |
| `@RequestBody` | JSON/XML request body |
| `@ModelAttribute` | Form data / model object |
| `@RequestHeader` | HTTP headers |
| `@CookieValue` | Cookies |
| `@SessionAttribute` | Session attributes |
| `@RequestAttribute` | Request attributes |
| `@MatrixVariable` | URL matrix variables: /path;key=value |
| `@RequestPart` | Multipart part (JSON/file) |

### Response Annotations
| Annotation | Purpose |
|-----------|---------|
| `@ResponseBody` | Return value → response body (JSON) |
| `@ResponseStatus` | Set HTTP status code |

### Configuration Annotations
| Annotation | Purpose |
|-----------|---------|
| `@EnableWebMvc` | Enable Spring MVC config |
| `@ControllerAdvice` | Global controller concern |
| `@RestControllerAdvice` | Global REST exception handler |
| `@ExceptionHandler` | Handle specific exceptions |
| `@InitBinder` | Customize data binding |
| `@CrossOrigin` | Enable CORS |
| `@SessionAttributes` | Store model attributes in session |

### Validation Annotations
| Annotation | Purpose |
|-----------|---------|
| `@Valid` | Trigger validation |
| `@Validated` | Trigger validation with groups |
| `@NotNull` | Not null |
| `@NotBlank` | Not blank string |
| `@NotEmpty` | Not empty |
| `@Size` | Size constraint |
| `@Min` / `@Max` | Numeric range |
| `@Email` | Email format |
| `@Pattern` | Regex match |
| `@Past` / `@Future` | Date constraints |

---

## 26. COMMON CONFIGURATION PROPERTIES

```properties
# ═══════════════════════════════════════════════
# SERVER
# ═══════════════════════════════════════════════
server.port=8080
server.servlet.context-path=/myapp
server.error.whitelabel.enabled=false
server.compression.enabled=true
server.compression.mime-types=application/json,text/html

# ═══════════════════════════════════════════════
# SPRING MVC
# ═══════════════════════════════════════════════
spring.mvc.view.prefix=/WEB-INF/views/
spring.mvc.view.suffix=.jsp
spring.mvc.static-path-pattern=/static/**
spring.mvc.format.date=yyyy-MM-dd
spring.mvc.format.date-time=yyyy-MM-dd HH:mm:ss
spring.mvc.format.time=HH:mm:ss
spring.mvc.throw-exception-if-no-handler-found=true
spring.mvc.log-resolved-exception=true

# ═══════════════════════════════════════════════
# THYMELEAF
# ═══════════════════════════════════════════════
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html
spring.thymeleaf.mode=HTML
spring.thymeleaf.cache=false
spring.thymeleaf.encoding=UTF-8

# ═══════════════════════════════════════════════
# JACKSON (JSON)
# ═══════════════════════════════════════════════
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=UTC
spring.jackson.serialization.indent-output=true
spring.jackson.serialization.write-dates-as-timestamps=false
spring.jackson.deserialization.fail-on-unknown-properties=false
spring.jackson.default-property-inclusion=non_null

# ═══════════════════════════════════════════════
# FILE UPLOAD
# ═══════════════════════════════════════════════
spring.servlet.multipart.enabled=true
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=50MB

# ═══════════════════════════════════════════════
# SESSION
# ═══════════════════════════════════════════════
server.servlet.session.timeout=30m
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.secure=true

# ═══════════════════════════════════════════════
# LOGGING
# ═══════════════════════════════════════════════
logging.level.org.springframework.web=DEBUG
logging.level.org.springframework.web.servlet.mvc=TRACE
```

---

## 27. SPRING MVC vs SPRING WEBFLUX

| Feature | Spring MVC | Spring WebFlux |
|---------|-----------|----------------|
| Model | Synchronous, blocking | Asynchronous, non-blocking |
| Servlet API | Yes (requires Servlet container) | No (runs on Netty, Undertow, etc.) |
| Thread Model | Thread-per-request | Event loop |
| Return Types | Object, ModelAndView, String | Mono, Flux, Object |
| Use Case | Traditional web apps, CRUD APIs | High-concurrency, streaming, reactive |
| Annotation | @Controller, @RestController | Same (shared annotations) |
| Server | Tomcat, Jetty, Undertow | Netty, Tomcat, Jetty, Undertow |
| Spring Boot Starter | `spring-boot-starter-web` | `spring-boot-starter-webflux` |

---

## 28. QUICK REFERENCE — COMPLETE CRUD EXAMPLE

### Entity
```java
@Entity
@Table(name = "products")
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Column(nullable = false)
    private BigDecimal price;

    private String category;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @PrePersist
    void onCreate() { this.createdAt = LocalDateTime.now(); }

    // constructors, getters, setters
}
```

### Repository
```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    List<Product> findByCategory(String category);
    List<Product> findByNameContainingIgnoreCase(String keyword);
    List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);
}
```

### Service
```java
@Service
@Transactional
public class ProductService {

    private final ProductRepository repo;

    public ProductService(ProductRepository repo) { this.repo = repo; }

    public List<Product> findAll() { return repo.findAll(); }
    public Product findById(Long id) {
        return repo.findById(id).orElseThrow(() -> new ResourceNotFoundException("Product not found: " + id));
    }
    public Product save(Product product) { return repo.save(product); }
    public void delete(Long id) { repo.deleteById(id); }
    public List<Product> search(String keyword) { return repo.findByNameContainingIgnoreCase(keyword); }
}
```

### REST Controller
```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductRestController {

    private final ProductService service;

    public ProductRestController(ProductService service) { this.service = service; }

    @GetMapping
    public List<Product> getAll() { return service.findAll(); }

    @GetMapping("/{id}")
    public Product getById(@PathVariable Long id) { return service.findById(id); }

    @GetMapping("/search")
    public List<Product> search(@RequestParam String q) { return service.search(q); }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Product create(@Valid @RequestBody Product product) { return service.save(product); }

    @PutMapping("/{id}")
    public Product update(@PathVariable Long id, @Valid @RequestBody Product product) {
        product.setId(id);
        return service.save(product);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) { service.delete(id); }
}
```

### Web Controller (Thymeleaf)
```java
@Controller
@RequestMapping("/products")
public class ProductWebController {

    private final ProductService service;

    public ProductWebController(ProductService service) { this.service = service; }

    @GetMapping
    public String list(Model model) {
        model.addAttribute("products", service.findAll());
        return "product/list";
    }

    @GetMapping("/new")
    public String showForm(Model model) {
        model.addAttribute("product", new Product());
        return "product/form";
    }

    @PostMapping
    public String save(@Valid @ModelAttribute Product product, BindingResult result) {
        if (result.hasErrors()) return "product/form";
        service.save(product);
        return "redirect:/products";
    }

    @GetMapping("/{id}/edit")
    public String editForm(@PathVariable Long id, Model model) {
        model.addAttribute("product", service.findById(id));
        return "product/form";
    }

    @GetMapping("/{id}/delete")
    public String delete(@PathVariable Long id) {
        service.delete(id);
        return "redirect:/products";
    }
}
```

### Exception Handler
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> handleNotFound(ResourceNotFoundException ex) {
        return Map.of("error", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, String> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        return errors;
    }
}
```

---

## 29. WEB SCOPES

### 29.1 Built-in Spring Web Scopes
| Scope | Annotation | Lifetime |
|-------|-----------|----------|
| Singleton | (default) | Application lifetime — single shared instance |
| Prototype | `@Scope("prototype")` | New instance per injection point |
| Request | `@RequestScope` | One HTTP request |
| Session | `@SessionScope` | One HTTP session |
| Application | `@ApplicationScope` | `ServletContext` lifetime |

### 29.2 @RequestScope
```java
// New bean instance created for EACH HTTP request
@Component
@RequestScope
public class RequestContext {
    private final String requestId = UUID.randomUUID().toString();
    private String currentUser;
    // getters + setters
}

// Inject like any other bean — Spring uses a proxy
@Controller
public class MyController {

    @Autowired
    private RequestContext requestContext;   // new instance per request

    @GetMapping("/info")
    public String info(Model model) {
        model.addAttribute("reqId", requestContext.getRequestId());
        return "info";
    }
}
```

### 29.3 @SessionScope
```java
// Same instance reused across all requests in one HTTP session
@Component
@SessionScope
public class ShoppingCart {
    private final List<CartItem> items = new ArrayList<>();

    public void addItem(CartItem item) { items.add(item); }
    public void removeItem(Long itemId) { items.removeIf(i -> i.getId().equals(itemId)); }
    public List<CartItem> getItems() { return Collections.unmodifiableList(items); }
    public void clear() { items.clear(); }
    public int getCount() { return items.size(); }
}

@Controller
public class CartController {

    @Autowired
    private ShoppingCart cart;   // same instance for the same session

    @PostMapping("/cart/add")
    public String addToCart(@RequestParam Long productId) {
        cart.addItem(productService.getItem(productId));
        return "redirect:/cart";
    }

    @GetMapping("/cart")
    public String viewCart(Model model) {
        model.addAttribute("cart", cart);
        return "cart";
    }

    @PostMapping("/cart/clear")
    public String clearCart() {
        cart.clear();
        return "redirect:/cart";
    }
}
```

### 29.4 @ApplicationScope
```java
// Shared across ALL sessions and requests — backed by ServletContext
@Component
@ApplicationScope
public class SiteVisitCounter {
    private final AtomicLong count = new AtomicLong(0);

    public long increment() { return count.incrementAndGet(); }
    public long getCount() { return count.get(); }
}

@Controller
public class HomeController {
    @Autowired
    private SiteVisitCounter counter;

    @GetMapping("/")
    public String home(Model model) {
        model.addAttribute("visits", counter.increment());
        return "home";
    }
}
```

> **Note:** Spring creates a **scoped proxy** for request/session-scoped beans injected into singleton beans. The proxy delegates each method call to the correct instance for the current request/session.

---

## 30. CUSTOM HTTPMESSAGECONVERTER

### 30.1 Built-in Message Converters
| Converter | Handles Media Type |
|-----------|--------------------|
| `StringHttpMessageConverter` | `text/plain`, `text/*` |
| `MappingJackson2HttpMessageConverter` | `application/json` |
| `MappingJackson2XmlHttpMessageConverter` | `application/xml` |
| `FormHttpMessageConverter` | `application/x-www-form-urlencoded`, `multipart/form-data` |
| `ByteArrayHttpMessageConverter` | `application/octet-stream` |
| `ResourceHttpMessageConverter` | `*/*` |
| `Jaxb2RootElementHttpMessageConverter` | `application/xml` (JAXB) |

### 30.2 Creating a Custom Converter (CSV example)
```java
public class CsvHttpMessageConverter extends AbstractHttpMessageConverter<List<?>> {

    public CsvHttpMessageConverter() {
        super(new MediaType("text", "csv"));
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        return List.class.isAssignableFrom(clazz);
    }

    @Override
    protected List<?> readInternal(Class<? extends List<?>> clazz,
                                   HttpInputMessage inputMessage) {
        throw new UnsupportedOperationException("CSV reading not supported");
    }

    @Override
    protected void writeInternal(List<?> items,
                                 HttpOutputMessage outputMessage) throws IOException {
        try (PrintWriter writer = new PrintWriter(outputMessage.getBody())) {
            for (Object item : items) {
                writer.println(item.toString());   // simplified — use a CSV library in production
            }
        }
    }
}
```

### 30.3 Registering the Converter
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    // extendMessageConverters — adds WITHOUT clearing Spring's defaults
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new CsvHttpMessageConverter());
    }

    // configureMessageConverters — REPLACES all defaults (use carefully)
    // @Override
    // public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    //     converters.add(new MappingJackson2HttpMessageConverter());
    //     converters.add(new CsvHttpMessageConverter());
    // }
}
```

### 30.4 Using the Custom Converter
```java
// Controller produces text/csv — Jackson is skipped, CsvHttpMessageConverter is used
@GetMapping(value = "/api/users/export", produces = "text/csv")
public List<UserDTO> exportUsersCsv() {
    return userService.findAll();
}
// curl -H "Accept: text/csv" http://localhost:8080/api/users/export
```

---

## 31. HTTP CLIENTS — RESTTEMPLATE & WEBCLIENT

### 31.1 RestTemplate (Classic, Synchronous)
```xml
<!-- Included with spring-boot-starter-web -->
```

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(10))
            .build();
    }
}
```

```java
@Service
public class ExternalUserService {

    private final RestTemplate restTemplate;
    private static final String BASE = "https://api.example.com";

    public ExternalUserService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    // GET — returns body directly
    public UserDTO getUser(Long id) {
        return restTemplate.getForObject(BASE + "/users/{id}", UserDTO.class, id);
    }

    // GET — with full ResponseEntity (status + headers + body)
    public ResponseEntity<UserDTO> getUserEntity(Long id) {
        return restTemplate.getForEntity(BASE + "/users/{id}", UserDTO.class, id);
    }

    // GET list (ParameterizedTypeReference handles generic types)
    public List<UserDTO> getAllUsers() {
        ResponseEntity<List<UserDTO>> response = restTemplate.exchange(
            BASE + "/users",
            HttpMethod.GET,
            null,
            new ParameterizedTypeReference<List<UserDTO>>() {}
        );
        return response.getBody();
    }

    // POST
    public UserDTO createUser(CreateUserRequest request) {
        return restTemplate.postForObject(BASE + "/users", request, UserDTO.class);
    }

    // PUT
    public void updateUser(Long id, UpdateUserRequest request) {
        restTemplate.put(BASE + "/users/{id}", request, id);
    }

    // DELETE
    public void deleteUser(Long id) {
        restTemplate.delete(BASE + "/users/{id}", id);
    }

    // Custom request headers
    public UserDTO getWithAuth(Long id, String token) {
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(token);
        HttpEntity<Void> entity = new HttpEntity<>(headers);
        return restTemplate.exchange(
            BASE + "/users/{id}", HttpMethod.GET, entity, UserDTO.class, id
        ).getBody();
    }
}
```

### 31.2 WebClient (Reactive, Non-blocking — recommended for new code)
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .codecs(c -> c.defaultCodecs().maxInMemorySize(16 * 1024 * 1024))
            .build();
    }
}
```

```java
@Service
public class ExternalService {

    private final WebClient webClient;

    public ExternalService(WebClient webClient) {
        this.webClient = webClient;
    }

    // GET — blocking call (safe in non-reactive Spring MVC)
    public UserDTO getUser(Long id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(UserDTO.class)
            .block();   // blocks current thread
    }

    // POST
    public UserDTO createUser(CreateUserRequest request) {
        return webClient.post()
            .uri("/users")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(UserDTO.class)
            .block();
    }

    // GET list
    public List<UserDTO> getAllUsers() {
        return webClient.get()
            .uri("/users")
            .retrieve()
            .bodyToFlux(UserDTO.class)
            .collectList()
            .block();
    }

    // Error handling
    public UserDTO getUserSafe(Long id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError,
                res -> Mono.error(new ResourceNotFoundException("User not found: " + id)))
            .onStatus(HttpStatusCode::is5xxServerError,
                res -> Mono.error(new RuntimeException("External server error")))
            .bodyToMono(UserDTO.class)
            .block();
    }
}
```

### RestTemplate vs WebClient
| Feature | RestTemplate | WebClient |
|---------|-------------|----------|
| Model | Synchronous, blocking | Async, non-blocking (Reactor) |
| Active development | Maintenance mode | Actively developed |
| Reactive streams | No | Yes (`Mono` / `Flux`) |
| Works with | Spring MVC | Spring MVC & WebFlux |
| Recommended for new code | No | Yes |

---

## 32. PROBLEM DETAILS — RFC 9457 (Spring 6+)

### 32.1 Overview
Spring Framework 6.0+ has built-in support for **RFC 9457 "Problem Details for HTTP APIs"** — a standardised JSON/XML error response format.

```json
{
  "type": "https://myapi.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User with ID 42 was not found",
  "instance": "/api/users/42"
}
```

### 32.2 Enable Problem Details (Spring Boot)
```properties
# application.properties
spring.mvc.problemdetails.enabled=true
# Spring Boot auto-handles standard MVC exceptions (404, 405, 400, etc.)
# with Problem Details format when this flag is on
```

### 32.3 Using ProblemDetail in Exception Handlers
```java
@RestControllerAdvice
public class ProblemDetailsExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex,
                                        HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail
            .forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setType(URI.create("https://myapi.com/errors/not-found"));
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("timestamp", LocalDateTime.now());   // custom extension field
        return problem;
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ProblemDetail handleBadRequest(IllegalArgumentException ex,
                                          HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail
            .forStatusAndDetail(HttpStatus.BAD_REQUEST, ex.getMessage());
        problem.setTitle("Bad Request");
        problem.setInstance(URI.create(request.getRequestURI()));
        return problem;
    }
}
```

### 32.4 ErrorResponseException (Inline)
```java
// Throw Problem-Detail-backed exceptions directly from anywhere without @ExceptionHandler
@GetMapping("/users/{id}")
public UserDTO getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow(() -> {
        ProblemDetail pd = ProblemDetail
            .forStatusAndDetail(HttpStatus.NOT_FOUND, "User not found: " + id);
        pd.setTitle("User Not Found");
        return new ErrorResponseException(HttpStatus.NOT_FOUND, pd, null);
    });
}
```

### 32.5 ProblemDetail Fields
| Field | Type | Description |
|-------|------|-------------|
| `type` | URI | URI identifying the error type |
| `title` | String | Short human-readable summary |
| `status` | int | HTTP status code |
| `detail` | String | Human-readable explanation specific to this occurrence |
| `instance` | URI | URI of the specific occurrence (usually request URI) |
| custom properties | any | Add via `setProperty("key", value)` |

---

## 33. OPENAPI & SWAGGER

### 33.1 Setup (springdoc-openapi)
```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

```
Swagger UI  → http://localhost:8080/swagger-ui.html
OpenAPI JSON → http://localhost:8080/v3/api-docs
```

### 33.2 application.properties
```properties
springdoc.api-docs.path=/v3/api-docs
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.swagger-ui.enabled=true
springdoc.api-docs.enabled=true
springdoc.packages-to-scan=com.example.controller
springdoc.swagger-ui.try-it-out-enabled=true
springdoc.swagger-ui.operations-sorter=method
```

### 33.3 Global API Info
```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("My REST API")
                .description("Full API documentation for My Application")
                .version("v1.0")
                .contact(new Contact()
                    .name("Dev Team")
                    .email("dev@example.com")
                    .url("https://example.com"))
                .license(new License()
                    .name("Apache 2.0")
                    .url("https://www.apache.org/licenses/LICENSE-2.0")))
            .externalDocs(new ExternalDocumentation()
                .description("Project Wiki")
                .url("https://github.com/example/wiki"));
    }
}
```

### 33.4 Annotating Controllers
```java
@RestController
@RequestMapping("/api/v1/users")
@Tag(name = "User Management", description = "APIs for managing users")
public class UserController {

    @Operation(
        summary = "Get user by ID",
        description = "Retrieves a single user by their unique ID",
        responses = {
            @ApiResponse(responseCode = "200", description = "User found",
                content = @Content(schema = @Schema(implementation = UserDTO.class))),
            @ApiResponse(responseCode = "404", description = "User not found",
                content = @Content(schema = @Schema(implementation = ProblemDetail.class)))
        }
    )
    @GetMapping("/{id}")
    public UserDTO getById(
        @Parameter(description = "User ID", example = "42") @PathVariable Long id) {
        return userService.findById(id);
    }

    @Operation(summary = "Create a new user")
    @ApiResponse(responseCode = "201", description = "User created")
    @ApiResponse(responseCode = "400", description = "Validation failed")
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserDTO create(
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            description = "User data", required = true,
            content = @Content(schema = @Schema(implementation = CreateUserRequest.class)))
        @org.springframework.web.bind.annotation.RequestBody @Valid CreateUserRequest request) {
        return userService.create(request);
    }

    @Operation(summary = "Delete a user")
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(
        @Parameter(description = "User ID") @PathVariable Long id) {
        userService.delete(id);
    }
}
```

### 33.5 Annotating DTOs
```java
@Schema(description = "Request body for creating a user")
public class CreateUserRequest {

    @Schema(description = "Full name", example = "John Doe", requiredMode = Schema.RequiredMode.REQUIRED)
    @NotBlank
    private String name;

    @Schema(description = "Email address", example = "john@example.com")
    @Email @NotBlank
    private String email;

    @Schema(description = "Age (must be 18 or older)", example = "25", minimum = "18")
    @Min(18)
    private int age;
}
```

### 33.6 Securing Swagger UI in Production
```java
// Disable in production via profile or property
@Configuration
@ConditionalOnProperty(name = "springdoc.swagger-ui.enabled", havingValue = "true", matchIfMissing = false)
public class SwaggerConfig { ... }
```

```properties
# application-prod.properties — hide docs in production
springdoc.swagger-ui.enabled=false
springdoc.api-docs.enabled=false
```

---

*End of Spring MVC Notes*
