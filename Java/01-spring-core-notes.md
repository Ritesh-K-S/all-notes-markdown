# Spring Core — Complete Notes

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Inversion of Control (IoC)](#2-inversion-of-control-ioc)
3. [Dependency Injection (DI)](#3-dependency-injection-di)
4. [Spring Beans](#4-spring-beans)
5. [Bean Scopes](#5-bean-scopes)
6. [Bean Lifecycle](#6-bean-lifecycle)
7. [@Autowired & Dependency Resolution](#7-autowired--dependency-resolution)
8. [Configuration Approaches](#8-configuration-approaches)
9. [Properties & Externalized Configuration](#9-properties--externalized-configuration)
10. [Profiles](#10-profiles)
11. [Spring Expression Language (SpEL)](#11-spring-expression-language-spel)
12. [Aspect-Oriented Programming (AOP)](#12-aspect-oriented-programming-aop)
13. [Event Handling](#13-event-handling)
14. [Resource Handling](#14-resource-handling)
15. [Task Scheduling & Async](#15-task-scheduling--async)
16. [Transaction Management](#16-transaction-management)
17. [Spring Boot Basics](#17-spring-boot-basics)
18. [Testing](#18-testing)
19. [Common Annotations Reference](#19-common-annotations-reference)
20. [Design Principles & Best Practices](#20-design-principles--best-practices)
21. [Quick Reference Cheat Sheet](#21-quick-reference-cheat-sheet)

---

## 1. INTRODUCTION

### What is Spring Framework?
- **Lightweight**, **open-source** Java application framework.
- Created by **Rod Johnson** (2003), maintained by **VMware/Broadcom**.
- Core philosophy: **Inversion of Control (IoC)** and **Dependency Injection (DI)**.
- Simplifies enterprise Java development — reduces boilerplate.
- Modular — use only what you need.

### Spring Ecosystem
| Module | Purpose |
|--------|---------|
| **Spring Core** | IoC container, DI, beans |
| **Spring MVC** | Web framework (REST, MVC) |
| **Spring Boot** | Auto-configuration, standalone apps |
| **Spring Data** | Database access (JPA, MongoDB, Redis) |
| **Spring Security** | Authentication & authorization |
| **Spring Cloud** | Microservices (config, discovery, gateway) |
| **Spring AOP** | Aspect-Oriented Programming |
| **Spring Batch** | Batch processing |
| **Spring Integration** | Enterprise integration patterns |
| **Spring WebFlux** | Reactive web framework |

### Spring Core Modules
```
spring-core          → Core utilities, IoC foundation
spring-beans         → Bean factory, bean definitions
spring-context       → ApplicationContext, events, i18n
spring-expression    → SpEL (Spring Expression Language)
spring-aop           → Aspect-Oriented Programming
```

### Maven Dependencies
```xml
<!-- Spring Core only -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.1.0</version>
</dependency>

<!-- Spring Boot Starter (includes core + auto-config) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <version>3.2.0</version>
</dependency>
```

---

## 2. INVERSION OF CONTROL (IoC)

### What is IoC?
- **Inversion of Control** — the framework controls object creation, not you.
- Traditional: You create objects → `new MyService()`
- IoC: Framework creates and manages objects → you just declare what you need.
- The **IoC Container** manages the lifecycle of objects (called **beans**).

### Without IoC (Tight Coupling)
```java
public class OrderService {
    // Directly creates dependency — tightly coupled
    private PaymentService paymentService = new PaymentService();
    private EmailService emailService = new EmailService();

    public void placeOrder(Order order) {
        paymentService.processPayment(order);
        emailService.sendConfirmation(order);
    }
}
// Problems:
// - Can't swap implementations easily
// - Hard to test (can't mock dependencies)
// - OrderService controls PaymentService lifecycle
```

### With IoC (Loose Coupling)
```java
public class OrderService {
    // Dependencies injected — loosely coupled
    private final PaymentService paymentService;
    private final EmailService emailService;

    public OrderService(PaymentService paymentService, EmailService emailService) {
        this.paymentService = paymentService;
        this.emailService = emailService;
    }

    public void placeOrder(Order order) {
        paymentService.processPayment(order);
        emailService.sendConfirmation(order);
    }
}
// Benefits:
// - Easy to swap implementations
// - Easy to test with mocks
// - Container manages lifecycle
```

### IoC Container Types
| Container | Interface | Description |
|-----------|-----------|-------------|
| **BeanFactory** | `BeanFactory` | Basic container, lazy initialization |
| **ApplicationContext** | `ApplicationContext` | Advanced container (recommended), eager initialization, events, i18n, AOP |

```
ApplicationContext extends BeanFactory
├── ClassPathXmlApplicationContext    — XML config from classpath
├── FileSystemXmlApplicationContext   — XML config from file system
├── AnnotationConfigApplicationContext — Java-based config
├── GenericWebApplicationContext      — Web applications
└── SpringApplication.run()           — Spring Boot
```

---

## 3. DEPENDENCY INJECTION (DI)

### What is DI?
- **Dependency Injection** — the process of providing dependencies to an object.
- A specific form of IoC.
- Spring injects required objects automatically.

### Three Types of DI

#### 1. Constructor Injection (Recommended)
```java
@Component
public class OrderService {
    private final PaymentService paymentService;
    private final EmailService emailService;

    // @Autowired is optional for single constructor (Spring 4.3+)
    public OrderService(PaymentService paymentService, EmailService emailService) {
        this.paymentService = paymentService;
        this.emailService = emailService;
    }
}
```
**Best practice** — fields are `final`, immutable, required dependencies guaranteed.

#### 2. Setter Injection
```java
@Component
public class OrderService {
    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```
Use for **optional** dependencies or when you need to change them later.

#### 3. Field Injection (Avoid)
```java
@Component
public class OrderService {
    @Autowired
    private PaymentService paymentService;

    @Autowired
    private EmailService emailService;
}
```
**Not recommended** — can't make fields `final`, hard to test without Spring, hides dependencies.

### Why Constructor Injection is Best
```
1. Immutability — fields can be final
2. Required dependencies — guaranteed to be set
3. Testability — easy to pass mocks via constructor
4. No reflection needed — works without Spring
5. Fail-fast — missing dependencies caught at startup
6. Clear API — constructor shows all dependencies
```

---

## 4. SPRING BEANS

### What is a Bean?
- An **object** managed by the Spring IoC container.
- Spring creates, configures, and manages its lifecycle.
- Registered through annotations, XML, or Java config.

### Declaring Beans — Stereotype Annotations
```java
@Component          // Generic Spring-managed bean
@Service            // Business logic / service layer
@Repository         // Data access layer (adds exception translation)
@Controller         // Spring MVC controller (returns views)
@RestController     // REST API controller (@Controller + @ResponseBody)
@Configuration      // Declares @Bean methods
```

```java
@Component
public class MyComponent { }

@Service
public class UserService { }

@Repository
public class UserRepository { }

@Controller
public class HomeController { }

@RestController
public class UserApiController { }
```

All stereotype annotations are specializations of `@Component` — Spring scans and registers them.

### @Bean — Manual Bean Declaration
```java
@Configuration
public class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new StripePaymentService();
    }

    @Bean
    public EmailService emailService() {
        return new SmtpEmailService("smtp.example.com", 587);
    }

    // Bean with dependency
    @Bean
    public OrderService orderService(PaymentService paymentService, EmailService emailService) {
        return new OrderService(paymentService, emailService);
    }

    // Custom bean name
    @Bean("myCustomName")
    public CacheService cacheService() {
        return new RedisCacheService();
    }

    // Init and destroy methods
    @Bean(initMethod = "init", destroyMethod = "cleanup")
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}
```

### @Component vs @Bean
| Feature | @Component | @Bean |
|---------|-----------|------|
| Declared on | Class | Method (inside @Configuration) |
| Auto-detected | Yes (component scanning) | No (explicit declaration) |
| Use when | You own the class | Third-party class or complex creation |
| Custom logic | No | Yes (any initialization logic) |

### Component Scanning
```java
@Configuration
@ComponentScan("com.myapp")                    // scan this package
@ComponentScan(basePackages = {"com.myapp", "com.utils"})  // multiple packages
@ComponentScan(basePackageClasses = {UserService.class})   // type-safe
@ComponentScan(
    basePackages = "com.myapp",
    excludeFilters = @ComponentScan.Filter(type = FilterType.REGEX, pattern = ".*Test.*")
)
public class AppConfig { }

// Spring Boot: @SpringBootApplication includes @ComponentScan for its package + sub-packages
```

---

## 5. BEAN SCOPES

### Available Scopes
| Scope | Description | Default |
|-------|-------------|---------|
| `singleton` | One instance per IoC container | ✅ Yes |
| `prototype` | New instance every time requested | |
| `request` | One per HTTP request (web only) | |
| `session` | One per HTTP session (web only) | |
| `application` | One per ServletContext (web only) | |
| `websocket` | One per WebSocket session | |

### Declaring Scope
```java
@Component
@Scope("singleton")       // default — one shared instance
public class SingletonService { }

@Component
@Scope("prototype")       // new instance every injection
public class PrototypeService { }

// Web scopes
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestScopedBean { }

@Component
@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class SessionScopedBean { }

// Using constants
@Scope(ConfigurableBeanFactory.SCOPE_SINGLETON)
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@Scope(WebApplicationContext.SCOPE_REQUEST)
@Scope(WebApplicationContext.SCOPE_SESSION)
```

### Singleton vs Prototype
```java
@Component
@Scope("singleton")
public class SingletonBean {
    // Created ONCE when ApplicationContext starts
    // Same instance returned every time
    // Default scope — don't even need @Scope annotation
}

@Component
@Scope("prototype")
public class PrototypeBean {
    // Created fresh EVERY TIME requested
    // Spring does NOT manage full lifecycle (no destroy callback)
}

// Injecting prototype into singleton (PROBLEM!)
@Component
public class SingletonService {
    @Autowired
    private PrototypeBean prototype;  // SAME instance always! (Bug)
}

// Solution 1: ObjectFactory / ObjectProvider
@Component
public class SingletonService {
    @Autowired
    private ObjectProvider<PrototypeBean> prototypeProvider;

    public void doWork() {
        PrototypeBean fresh = prototypeProvider.getObject();  // new each time
    }
}

// Solution 2: @Lookup
@Component
public abstract class SingletonService {
    @Lookup
    public abstract PrototypeBean getPrototype();  // Spring overrides this

    public void doWork() {
        PrototypeBean fresh = getPrototype();  // new each time
    }
}

// Solution 3: Proxy
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class PrototypeBean { }
```

---

## 6. BEAN LIFECYCLE

### Lifecycle Phases
```
1. Instantiation        → constructor called
2. Populate Properties  → DI (setters/fields)
3. BeanNameAware        → setBeanName()
4. BeanFactoryAware     → setBeanFactory()
5. ApplicationContextAware → setApplicationContext()
6. BeanPostProcessor    → postProcessBeforeInitialization()
7. @PostConstruct       → custom init method
8. InitializingBean     → afterPropertiesSet()
9. Custom init-method   → @Bean(initMethod = "init")
10. BeanPostProcessor   → postProcessAfterInitialization()
    ════ Bean Ready to Use ════
11. @PreDestroy         → custom cleanup method
12. DisposableBean      → destroy()
13. Custom destroy-method → @Bean(destroyMethod = "cleanup")
```

### Lifecycle Callbacks
```java
@Component
public class MyService {

    // 1. @PostConstruct — runs after DI complete (recommended)
    @PostConstruct
    public void init() {
        System.out.println("Bean initialized — load cache, open connections...");
    }

    // 2. @PreDestroy — runs before bean destruction (recommended)
    @PreDestroy
    public void cleanup() {
        System.out.println("Bean destroying — close connections, cleanup...");
    }
}
```

### InitializingBean & DisposableBean (Interface-based)
```java
@Component
public class MyService implements InitializingBean, DisposableBean {

    @Override
    public void afterPropertiesSet() throws Exception {
        // runs after properties set (like @PostConstruct)
    }

    @Override
    public void destroy() throws Exception {
        // runs before destruction (like @PreDestroy)
    }
}
```

### Custom init/destroy with @Bean
```java
@Configuration
public class AppConfig {
    @Bean(initMethod = "initialize", destroyMethod = "shutdown")
    public ConnectionPool connectionPool() {
        return new ConnectionPool();
    }
}

public class ConnectionPool {
    public void initialize() { /* open connections */ }
    public void shutdown() { /* close connections */ }
}
```

### BeanPostProcessor (Global Hook)
```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("Before init: " + beanName);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("After init: " + beanName);
        return bean;  // can return proxy/wrapper
    }
}
// Runs for EVERY bean in the container
```

### Aware Interfaces
```java
@Component
public class MyBean implements ApplicationContextAware, BeanNameAware, EnvironmentAware {

    @Override
    public void setBeanName(String name) {
        // called with bean's name in container
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        // access to ApplicationContext
    }

    @Override
    public void setEnvironment(Environment env) {
        // access to Environment
    }
}

// Common Aware interfaces:
// BeanNameAware             → bean name
// BeanFactoryAware          → BeanFactory reference
// ApplicationContextAware   → ApplicationContext reference
// EnvironmentAware          → Environment (properties, profiles)
// ResourceLoaderAware       → ResourceLoader
// ApplicationEventPublisherAware → event publisher
// MessageSourceAware        → i18n message source
```

---

## 7. @AUTOWIRED & DEPENDENCY RESOLUTION

### @Autowired — Automatic Wiring
```java
@Component
public class OrderService {

    // Constructor (recommended) — @Autowired optional for single constructor
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    // Setter
    @Autowired
    public void setNotifier(NotificationService notifier) {
        this.notifier = notifier;
    }

    // Field (not recommended)
    @Autowired
    private LogService logService;

    // Optional dependency
    @Autowired(required = false)
    private CacheService cacheService;   // null if not available

    // Using Optional
    @Autowired
    private Optional<CacheService> cacheService;  // Optional.empty() if missing

    // Method injection
    @Autowired
    public void configure(DataSource ds, TransactionManager tx) {
        // both injected
    }
}
```

### @Qualifier — Resolve Ambiguity
```java
// When multiple beans of same type exist
public interface PaymentService { }

@Service("stripe")
public class StripePaymentService implements PaymentService { }

@Service("paypal")
public class PayPalPaymentService implements PaymentService { }

// Inject specific one
@Component
public class OrderService {

    @Autowired
    @Qualifier("stripe")                // specify which bean
    private PaymentService paymentService;

    // Or in constructor
    public OrderService(@Qualifier("paypal") PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

### @Primary — Default Bean
```java
@Service
@Primary                               // used when no @Qualifier specified
public class StripePaymentService implements PaymentService { }

@Service
public class PayPalPaymentService implements PaymentService { }

@Component
public class OrderService {
    @Autowired
    private PaymentService payment;    // gets StripePaymentService (@Primary)
}
```

### Custom Qualifier Annotation
```java
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface PaymentType {
    String value();
}

@Service
@PaymentType("stripe")
public class StripePaymentService implements PaymentService { }

@Component
public class OrderService {
    @Autowired
    @PaymentType("stripe")
    private PaymentService paymentService;
}
```

### Injecting Collections
```java
// All beans of a type
@Autowired
private List<PaymentService> allPaymentServices;     // all implementations

@Autowired
private Set<PaymentService> paymentServices;

@Autowired
private Map<String, PaymentService> paymentServiceMap;  // bean name → instance

// Control order
@Service
@Order(1)
public class StripePaymentService implements PaymentService { }

@Service
@Order(2)
public class PayPalPaymentService implements PaymentService { }

@Autowired
private List<PaymentService> services;  // Stripe first, then PayPal
```

### ObjectProvider (Lazy & Safe)
```java
@Component
public class MyService {

    private final ObjectProvider<ExpensiveBean> provider;

    public MyService(ObjectProvider<ExpensiveBean> provider) {
        this.provider = provider;
    }

    public void doWork() {
        ExpensiveBean bean = provider.getIfAvailable();       // null if missing
        ExpensiveBean bean = provider.getIfAvailable(ExpensiveBean::new);  // default
        ExpensiveBean bean = provider.getIfUnique();          // null if ambiguous
        provider.ifAvailable(b -> b.process());               // consume if present
        provider.orderedStream().forEach(...);                 // all, ordered
    }
}
```

---

## 8. CONFIGURATION APPROACHES

### 1. Java-Based Configuration (Recommended)
```java
@Configuration
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        ds.setUsername("root");
        ds.setPassword("password");
        return ds;
    }

    @Bean
    public UserRepository userRepository(DataSource dataSource) {
        return new JdbcUserRepository(dataSource);
    }

    @Bean
    public UserService userService(UserRepository userRepository) {
        return new UserServiceImpl(userRepository);
    }
}

// Bootstrap
public class App {
    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
        UserService service = ctx.getBean(UserService.class);
        service.createUser("Ritesh");
    }
}
```

### 2. Annotation-Based Configuration
```java
@Configuration
@ComponentScan("com.myapp")    // auto-detect @Component, @Service, etc.
public class AppConfig { }

// Beans auto-registered via annotations
@Service
public class UserService {
    private final UserRepository repo;

    public UserService(UserRepository repo) {
        this.repo = repo;
    }
}

@Repository
public class JdbcUserRepository implements UserRepository { }
```

### 3. XML Configuration (Legacy)
```xml
<!-- applicationContext.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- Enable annotation processing -->
    <context:annotation-config/>

    <!-- Component scanning -->
    <context:component-scan base-package="com.myapp"/>

    <!-- Manual bean declaration -->
    <bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource">
        <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/mydb"/>
        <property name="username" value="root"/>
        <property name="password" value="password"/>
    </bean>

    <!-- Constructor injection -->
    <bean id="userService" class="com.myapp.UserServiceImpl">
        <constructor-arg ref="userRepository"/>
    </bean>

    <!-- Setter injection -->
    <bean id="mailService" class="com.myapp.MailService">
        <property name="host" value="smtp.example.com"/>
        <property name="port" value="587"/>
    </bean>

</beans>

<!-- Bootstrap -->
ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
```

### @Import — Compose Configurations
```java
@Configuration
@Import({DatabaseConfig.class, SecurityConfig.class, CacheConfig.class})
public class AppConfig { }

// @ImportResource for XML
@Configuration
@ImportResource("classpath:legacy-config.xml")
public class AppConfig { }
```

### @Conditional — Conditional Bean Registration
```java
@Configuration
public class AppConfig {

    @Bean
    @Conditional(WindowsCondition.class)
    public FileService windowsFileService() {
        return new WindowsFileService();
    }

    @Bean
    @Conditional(LinuxCondition.class)
    public FileService linuxFileService() {
        return new LinuxFileService();
    }
}

public class WindowsCondition implements Condition {
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata metadata) {
        return ctx.getEnvironment().getProperty("os.name").contains("Windows");
    }
}

// Spring Boot shorthand annotations:
@ConditionalOnProperty(name = "cache.enabled", havingValue = "true")
@ConditionalOnMissingBean(CacheService.class)
@ConditionalOnBean(DataSource.class)
@ConditionalOnClass(name = "com.redis.Client")
@ConditionalOnMissingClass("com.redis.Client")
@ConditionalOnWebApplication
@ConditionalOnNotWebApplication
@ConditionalOnExpression("${feature.enabled:false}")
```

---

## 9. PROPERTIES & EXTERNALIZED CONFIGURATION

### @Value — Inject Property Values
```java
@Component
public class AppSettings {

    @Value("${app.name}")                        // from properties file
    private String appName;

    @Value("${app.port:8080}")                   // with default value
    private int port;

    @Value("${APP_SECRET}")                      // environment variable
    private String secret;

    @Value("#{systemProperties['java.home']}")   // SpEL expression
    private String javaHome;

    @Value("${app.features:feature1,feature2}")  // comma-separated → array
    private String[] features;

    @Value("#{'${app.features}'.split(',')}")     // SpEL split → List
    private List<String> featureList;

    @Value("${app.enabled:true}")                // boolean
    private boolean enabled;

    @Value("classpath:data/init.sql")            // Resource
    private Resource initScript;

    @Value("#{T(java.lang.Math).random()}")      // SpEL method call
    private double randomValue;
}
```

### Property Sources
```java
@Configuration
@PropertySource("classpath:application.properties")
@PropertySource("classpath:database.properties")
@PropertySource(value = "classpath:optional.properties", ignoreResourceNotFound = true)
public class AppConfig { }

// application.properties
app.name=My Application
app.port=8080
app.enabled=true
db.url=jdbc:mysql://localhost:3306/mydb
db.username=root
db.password=secret
app.features=auth,logging,caching
```

### @ConfigurationProperties (Type-safe — Spring Boot)
```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private int port;
    private boolean enabled;
    private List<String> features;
    private Database database = new Database();

    // getters and setters

    public static class Database {
        private String url;
        private String username;
        private String password;
        // getters and setters
    }
}

// Enable
@Configuration
@EnableConfigurationProperties(AppProperties.class)
public class AppConfig { }

// Or use on component
@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties { ... }

// application.yml
app:
  name: My Application
  port: 8080
  enabled: true
  features:
    - auth
    - logging
  database:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: secret

// Usage
@Service
public class MyService {
    private final AppProperties props;

    public MyService(AppProperties props) {
        this.props = props;
    }

    public void work() {
        String name = props.getName();
        String dbUrl = props.getDatabase().getUrl();
    }
}
```

### @ConfigurationProperties with Records (Java 16+)
```java
@ConfigurationProperties(prefix = "app")
public record AppProperties(
    String name,
    int port,
    boolean enabled,
    List<String> features,
    Database database
) {
    public record Database(String url, String username, String password) {}
}
```

### Environment
```java
@Component
public class MyService {

    @Autowired
    private Environment env;

    public void doWork() {
        String name = env.getProperty("app.name");
        int port = env.getProperty("app.port", Integer.class, 8080);
        boolean exists = env.containsProperty("app.name");
        String[] profiles = env.getActiveProfiles();
    }
}
```

### Property Resolution Order (Spring Boot — highest to lowest)
```
1. Command line args:           --app.name=MyApp
2. JVM system properties:      -Dapp.name=MyApp
3. OS environment variables:   APP_NAME=MyApp
4. application-{profile}.yml
5. application.yml
6. @PropertySource files
7. Default properties
```

---

## 10. PROFILES

### What are Profiles?
- Activate different **beans and configuration** for different environments.
- Common: `dev`, `test`, `staging`, `prod`.

### Defining Profile-Specific Beans
```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        // H2 in-memory database for development
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://prod-server:3306/mydb");
        return ds;
    }
}

// On bean methods
@Configuration
public class AppConfig {
    @Bean
    @Profile("dev")
    public CacheService devCache() { return new InMemoryCache(); }

    @Bean
    @Profile("prod")
    public CacheService prodCache() { return new RedisCache(); }
}

// Negate profiles
@Profile("!prod")          // active in all profiles except prod

// Multiple profiles
@Profile({"dev", "test"})  // active in dev OR test
```

### Activating Profiles
```bash
# application.properties
spring.profiles.active=dev

# application.yml
spring:
  profiles:
    active: dev, logging

# Command line
java -jar app.jar --spring.profiles.active=prod
java -Dspring.profiles.active=prod -jar app.jar

# Environment variable
export SPRING_PROFILES_ACTIVE=prod

# Programmatically
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("dev");
ctx.register(AppConfig.class);
ctx.refresh();

// @ActiveProfiles in tests
@SpringBootTest
@ActiveProfiles("test")
public class MyServiceTest { }
```

### Profile-Specific Property Files (Spring Boot)
```
application.properties           ← always loaded
application-dev.properties        ← loaded when dev profile active
application-prod.properties       ← loaded when prod profile active
application-test.properties       ← loaded when test profile active

# application-dev.properties
server.port=8080
logging.level.root=DEBUG

# application-prod.properties
server.port=80
logging.level.root=WARN
```

### Default Profile
```java
@Configuration
@Profile("default")    // active when NO profile is explicitly set
public class DefaultConfig { }

// application.properties
spring.profiles.default=dev    // change default profile
```

---

## 11. SPRING EXPRESSION LANGUAGE (SpEL)

### Basic Syntax
```java
// SpEL uses #{...} syntax
@Value("#{2 + 3}")                              // 5
@Value("#{T(java.lang.Math).PI}")               // 3.14159...
@Value("#{T(java.lang.Math).random()}")         // random number
@Value("#{systemProperties['java.home']}")      // system property
@Value("#{systemEnvironment['PATH']}")          // env variable
```

### Literals
```java
@Value("#{42}")              // int
@Value("#{3.14}")            // double
@Value("#{'Hello'}")         // string
@Value("#{true}")            // boolean
@Value("#{null}")            // null
```

### Operators
```java
// Arithmetic
@Value("#{10 + 5}")          // 15
@Value("#{10 - 5}")          // 5
@Value("#{10 * 5}")          // 50
@Value("#{10 / 3}")          // 3
@Value("#{10 % 3}")          // 1
@Value("#{2 ^ 10}")          // 1024 (power)

// Comparison
@Value("#{10 > 5}")          // true
@Value("#{10 == 10}")        // true
@Value("#{10 != 5}")         // true
@Value("#{10 ge 5}")         // true (>=)
@Value("#{10 le 5}")         // false (<=)
@Value("#{10 gt 5}")         // true (>)
@Value("#{10 lt 5}")         // false (<)

// Logical
@Value("#{true and false}")  // false
@Value("#{true or false}")   // true
@Value("#{!true}")           // false
@Value("#{not true}")        // false

// Ternary
@Value("#{${app.port} > 0 ? ${app.port} : 8080}")

// Elvis (null-safe default)
@Value("#{${app.name} ?: 'DefaultApp'}")

// Safe navigation (?.) — null-safe
@Value("#{user?.address?.city}")    // null if any part is null
```

### Bean References
```java
@Value("#{myBean.name}")                    // bean property
@Value("#{myBean.getName()}")               // bean method
@Value("#{myBean.doWork('arg')}")           // method with argument
@Value("#{@myBean}")                        // bean reference
@Value("#{@userService.findAll().size()}")  // chain calls
```

### Collections
```java
// List
@Value("#{{'a', 'b', 'c'}}")                     // inline list
@Value("#{{1, 2, 3, 4, 5}}")

// Map
@Value("#{{'key1': 'val1', 'key2': 'val2'}}")

// Selection (filter)
@Value("#{users.?[age > 18]}")                    // all matching
@Value("#{users.^[age > 18]}")                    // first matching
@Value("#{users.$[age > 18]}")                    // last matching

// Projection (map/transform)
@Value("#{users.![name]}")                        // extract names → List<String>
@Value("#{users.![name.toUpperCase()]}")
```

### Type References
```java
@Value("#{T(java.lang.Math).PI}")
@Value("#{T(java.lang.Math).random()}")
@Value("#{T(java.lang.Integer).MAX_VALUE}")
@Value("#{T(java.time.LocalDate).now()}")

// instanceof
@Value("#{someBean instanceof T(String)}")
```

### SpEL in @Conditional and @Cacheable
```java
@Bean
@ConditionalOnExpression("${cache.enabled:true} and '${cache.type}' == 'redis'")
public CacheService cacheService() { ... }

@Cacheable(value = "users", condition = "#id > 10")
public User findById(Long id) { ... }
```

---

## 12. ASPECT-ORIENTED PROGRAMMING (AOP)

### What is AOP?
- **Cross-cutting concerns** — logging, security, transactions, caching.
- Without AOP: same code repeated across many classes.
- With AOP: define behavior once, apply everywhere declaratively.

### AOP Terminology
| Term | Description |
|------|-------------|
| **Aspect** | Class containing cross-cutting logic |
| **Advice** | Action taken (before, after, around) |
| **Join Point** | Point in code (method execution) |
| **Pointcut** | Expression that matches join points |
| **Target** | Object being advised |
| **Proxy** | Object created by AOP (wraps target) |
| **Weaving** | Linking aspects to objects (Spring = runtime) |

### Enable AOP
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```
```java
@Configuration
@EnableAspectJAutoProxy
public class AopConfig { }
// Spring Boot auto-enables if spring-boot-starter-aop is on classpath
```

### Advice Types
```java
@Aspect
@Component
public class LoggingAspect {

    // ===== BEFORE — runs before method =====
    @Before("execution(* com.myapp.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        String method = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        System.out.println("Before: " + method + " args: " + Arrays.toString(args));
    }

    // ===== AFTER RETURNING — runs after successful return =====
    @AfterReturning(pointcut = "execution(* com.myapp.service.*.*(..))", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        System.out.println("Returned: " + result);
    }

    // ===== AFTER THROWING — runs after exception =====
    @AfterThrowing(pointcut = "execution(* com.myapp.service.*.*(..))", throwing = "ex")
    public void logAfterThrowing(JoinPoint joinPoint, Exception ex) {
        System.out.println("Exception: " + ex.getMessage());
    }

    // ===== AFTER (FINALLY) — runs after method (always) =====
    @After("execution(* com.myapp.service.*.*(..))")
    public void logAfter(JoinPoint joinPoint) {
        System.out.println("After: " + joinPoint.getSignature().getName());
    }

    // ===== AROUND — wraps method (most powerful) =====
    @Around("execution(* com.myapp.service.*.*(..))")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            Object result = joinPoint.proceed();    // call actual method
            return result;
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            System.out.println(joinPoint.getSignature().getName() + " took " + elapsed + "ms");
        }
    }
}
```

### Pointcut Expressions
```java
// Method execution pattern:
// execution(modifiers? return-type declaring-type? method-name(params) throws?)

// All methods in service package
@Pointcut("execution(* com.myapp.service.*.*(..))")

// All public methods
@Pointcut("execution(public * *(..))")

// Methods returning void
@Pointcut("execution(void com.myapp.service.*.*(..))")

// Specific method
@Pointcut("execution(* com.myapp.service.UserService.findById(Long))")

// Any method with specific name
@Pointcut("execution(* find*(..))")

// Methods with specific parameter types
@Pointcut("execution(* *..service.*.*(String, ..))")   // first param String

// Within specific class
@Pointcut("within(com.myapp.service.UserService)")

// Within package
@Pointcut("within(com.myapp.service..*)")              // includes sub-packages

// Bean name pattern
@Pointcut("bean(*Service)")
@Pointcut("bean(userService)")

// Target type (class/interface)
@Pointcut("target(com.myapp.service.UserService)")

// Annotation-based
@Pointcut("@annotation(com.myapp.annotation.Loggable)")         // method has annotation
@Pointcut("@within(org.springframework.stereotype.Service)")     // class has annotation
@Pointcut("@target(org.springframework.stereotype.Repository)")  // target class annotation

// Combine pointcuts
@Pointcut("execution(* com.myapp.service.*.*(..)) && !execution(* com.myapp.service.*.get*(..))")
@Pointcut("serviceLayer() || repositoryLayer()")
@Pointcut("serviceLayer() && @annotation(Transactional)")
```

### Reusable Pointcuts
```java
@Aspect
@Component
public class Pointcuts {
    @Pointcut("within(com.myapp.service..*)")
    public void serviceLayer() {}

    @Pointcut("within(com.myapp.repository..*)")
    public void repositoryLayer() {}

    @Pointcut("@annotation(com.myapp.annotation.Auditable)")
    public void auditable() {}
}

// Use in another aspect
@Aspect
@Component
public class AuditAspect {
    @Before("com.myapp.aop.Pointcuts.serviceLayer()")
    public void audit(JoinPoint jp) { ... }
}
```

### Custom Annotation + AOP
```java
// Define annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogExecutionTime { }

// Aspect
@Aspect
@Component
public class PerformanceAspect {
    @Around("@annotation(LogExecutionTime)")
    public Object logTime(ProceedingJoinPoint jp) throws Throwable {
        long start = System.nanoTime();
        Object result = jp.proceed();
        long elapsed = (System.nanoTime() - start) / 1_000_000;
        System.out.println(jp.getSignature() + " executed in " + elapsed + "ms");
        return result;
    }
}

// Usage — just add annotation
@Service
public class UserService {
    @LogExecutionTime
    public List<User> findAll() { ... }
}
```

### AOP Ordering
```java
@Aspect
@Order(1)                    // lower number = higher priority (runs first)
public class SecurityAspect { }

@Aspect
@Order(2)
public class LoggingAspect { }

// Execution order for @Before: Order 1 → Order 2
// Execution order for @After:  Order 2 → Order 1 (reversed)
```

---

## 13. EVENT HANDLING

### Application Events (Observer Pattern)

#### Define Custom Event
```java
public class UserCreatedEvent extends ApplicationEvent {
    private final User user;

    public UserCreatedEvent(Object source, User user) {
        super(source);
        this.user = user;
    }

    public User getUser() { return user; }
}

// Or without extending ApplicationEvent (Spring 4.2+)
public record UserCreatedEvent(User user) { }
```

#### Publish Event
```java
@Service
public class UserService {

    private final ApplicationEventPublisher publisher;

    public UserService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public User createUser(String name) {
        User user = new User(name);
        // save user...
        publisher.publishEvent(new UserCreatedEvent(user));
        return user;
    }
}
```

#### Listen for Event
```java
// Method 1: @EventListener (recommended)
@Component
public class UserEventListener {

    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        System.out.println("User created: " + event.user().name());
        // send welcome email, audit log, etc.
    }

    // Conditional
    @EventListener(condition = "#event.user().role() == 'ADMIN'")
    public void handleAdminCreated(UserCreatedEvent event) { }

    // Async
    @Async
    @EventListener
    public void handleAsync(UserCreatedEvent event) { }

    // Ordered
    @EventListener
    @Order(1)
    public void firstHandler(UserCreatedEvent event) { }

    // Return event to chain
    @EventListener
    public AuditEvent handleAndChain(UserCreatedEvent event) {
        return new AuditEvent("user_created", event.user().id());
    }
}

// Method 2: ApplicationListener interface
@Component
public class UserListener implements ApplicationListener<UserCreatedEvent> {
    @Override
    public void onApplicationEvent(UserCreatedEvent event) { }
}
```

#### Built-in Events
```java
@EventListener
public void onStartup(ContextRefreshedEvent event) {
    // ApplicationContext initialized or refreshed
}

@EventListener
public void onShutdown(ContextClosedEvent event) {
    // ApplicationContext closed
}

@EventListener
public void onStart(ContextStartedEvent event) { }

@EventListener
public void onStop(ContextStoppedEvent event) { }

// Spring Boot events
@EventListener
public void onReady(ApplicationReadyEvent event) {
    // App fully started
}

@EventListener
public void onStarted(ApplicationStartedEvent event) { }
```

---

## 14. RESOURCE HANDLING

### Loading Resources
```java
@Component
public class MyService {

    // Inject as Resource
    @Value("classpath:data/config.json")
    private Resource configFile;

    @Value("file:/etc/app/config.json")
    private Resource externalFile;

    @Value("classpath:templates/*.html")
    private Resource[] templates;

    // ResourceLoader
    @Autowired
    private ResourceLoader resourceLoader;

    public void readFile() throws IOException {
        // From classpath
        Resource resource = resourceLoader.getResource("classpath:data/init.sql");

        // From file system
        Resource resource = resourceLoader.getResource("file:/path/to/file");

        // From URL
        Resource resource = resourceLoader.getResource("https://example.com/data.json");

        // Check & read
        if (resource.exists() && resource.isReadable()) {
            String content = new String(resource.getInputStream().readAllBytes());
            // or use StreamUtils
            String content = StreamUtils.copyToString(resource.getInputStream(), StandardCharsets.UTF_8);
        }

        resource.getFile();           // File object (if file-based)
        resource.getURI();            // URI
        resource.getFilename();       // filename
        resource.contentLength();     // size
        resource.lastModified();      // timestamp
    }
}
```

### Resource Prefixes
| Prefix | Example | Description |
|--------|---------|-------------|
| `classpath:` | `classpath:data/file.txt` | From classpath (src/main/resources) |
| `classpath*:` | `classpath*:META-INF/*.xml` | All matching from all JARs |
| `file:` | `file:/etc/config.txt` | File system |
| `https:` | `https://example.com/data` | URL |
| (none) | `data/file.txt` | Depends on ApplicationContext type |

---

## 15. TASK SCHEDULING & ASYNC

### @Scheduled — Run Tasks Periodically
```java
@Configuration
@EnableScheduling
public class SchedulingConfig { }

@Component
public class ScheduledTasks {

    // Fixed rate — every 5 seconds (from start of previous)
    @Scheduled(fixedRate = 5000)
    public void reportEvery5Seconds() {
        System.out.println("Time: " + LocalTime.now());
    }

    // Fixed delay — 5 seconds after previous completes
    @Scheduled(fixedDelay = 5000)
    public void runAfterPrevious() { }

    // Initial delay + fixed rate
    @Scheduled(initialDelay = 10000, fixedRate = 5000)
    public void startAfter10Seconds() { }

    // Cron expression
    @Scheduled(cron = "0 0 9 * * MON-FRI")    // 9 AM weekdays
    public void morningReport() { }

    @Scheduled(cron = "0 */15 * * * *")        // every 15 minutes
    public void every15Minutes() { }

    @Scheduled(cron = "0 0 0 1 * *")           // first day of month at midnight
    public void monthlyCleanup() { }

    // From properties
    @Scheduled(fixedRateString = "${task.rate:5000}")
    public void configurable() { }

    @Scheduled(cron = "${task.cron:0 0 * * * *}")
    public void configurableCron() { }
}
```

### Cron Format
```
┌──────── second (0-59)
│ ┌────── minute (0-59)
│ │ ┌──── hour (0-23)
│ │ │ ┌── day of month (1-31)
│ │ │ │ ┌ month (1-12 or JAN-DEC)
│ │ │ │ │ ┌ day of week (0-7 or MON-SUN, 0&7=Sunday)
│ │ │ │ │ │
* * * * * *

Special characters:
*     = every
?     = no specific value (day-of-month or day-of-week)
-     = range (MON-FRI)
,     = list (MON,WED,FRI)
/     = increment (0/15 = every 15)
```

### @Async — Asynchronous Execution
```java
@Configuration
@EnableAsync
public class AsyncConfig { }

// Or with custom executor
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.setThreadNamePrefix("Async-");
        executor.initialize();
        return executor;
    }
}

@Service
public class EmailService {

    @Async
    public void sendEmail(String to, String body) {
        // runs in separate thread
        // no return value (fire-and-forget)
    }

    @Async
    public CompletableFuture<String> sendEmailAsync(String to) {
        // returns future for caller to check result
        String result = doSend(to);
        return CompletableFuture.completedFuture(result);
    }

    @Async("customExecutor")    // use specific executor bean
    public void processInBackground() { }
}

// Caller
@Service
public class OrderService {
    @Autowired
    private EmailService emailService;

    public void placeOrder(Order order) {
        // send email asynchronously
        emailService.sendEmail(order.getEmail(), "Order confirmed");

        // wait for async result
        CompletableFuture<String> future = emailService.sendEmailAsync(order.getEmail());
        String result = future.get();  // blocks until done
    }
}
```

**Note:** `@Async` only works when called from **another bean** (not from same class — proxy limitation).

---

## 16. TRANSACTION MANAGEMENT

### @Transactional
```java
// Enable transactions
@Configuration
@EnableTransactionManagement
public class TxConfig { }
// Spring Boot auto-enables if DataSource is available

@Service
public class UserService {

    @Transactional    // method runs in a transaction
    public void createUser(User user) {
        userRepo.save(user);
        auditRepo.log("user_created", user.getId());
        // if any exception → rollback BOTH operations
    }

    // Read-only (optimization — no write locks)
    @Transactional(readOnly = true)
    public User findById(Long id) {
        return userRepo.findById(id);
    }

    // Rollback rules
    @Transactional(rollbackFor = Exception.class)          // rollback on checked exceptions too
    @Transactional(noRollbackFor = EmailException.class)   // don't rollback for this
    @Transactional(rollbackFor = {IOException.class, SQLException.class})

    // Timeout
    @Transactional(timeout = 30)    // seconds

    // Isolation level
    @Transactional(isolation = Isolation.READ_COMMITTED)
    @Transactional(isolation = Isolation.SERIALIZABLE)
}
```

### Propagation Levels
| Propagation | Description |
|-------------|-------------|
| `REQUIRED` (default) | Join existing tx, or create new |
| `REQUIRES_NEW` | Always create new tx (suspend existing) |
| `SUPPORTS` | Join existing tx, or run without tx |
| `NOT_SUPPORTED` | Run without tx (suspend existing) |
| `MANDATORY` | Must have existing tx (throw if none) |
| `NEVER` | Must NOT have existing tx (throw if exists) |
| `NESTED` | Nested tx with savepoints |

```java
@Transactional(propagation = Propagation.REQUIRED)       // default
@Transactional(propagation = Propagation.REQUIRES_NEW)   // independent tx
@Transactional(propagation = Propagation.MANDATORY)      // must be called within tx
```

### Isolation Levels
| Level | Description |
|-------|-------------|
| `DEFAULT` | DB default |
| `READ_UNCOMMITTED` | Dirty reads possible |
| `READ_COMMITTED` | No dirty reads |
| `REPEATABLE_READ` | No dirty + non-repeatable reads |
| `SERIALIZABLE` | Full isolation (slowest) |

### Transactional Gotchas
```
1. Only works on PUBLIC methods (proxy-based)
2. Self-invocation doesn't work (calling @Transactional from same class)
3. Only unchecked exceptions (RuntimeException) trigger rollback by default
4. Use rollbackFor = Exception.class to include checked exceptions
5. @Transactional on class → applies to ALL public methods
6. Method-level overrides class-level settings
```

---

## 17. SPRING BOOT BASICS

### What is Spring Boot?
- **Opinionated** Spring setup — auto-configures everything.
- Embedded server (Tomcat/Jetty/Undertow) — no WAR deployment needed.
- Starter dependencies — one dependency pulls everything you need.
- Production-ready features (health checks, metrics, externalized config).

### Create Spring Boot Project
```bash
# Spring Initializr (web)
https://start.spring.io/

# Spring CLI
spring init --dependencies=web,data-jpa,mysql my-project

# Maven archetype
mvn archetype:generate -DarchetypeGroupId=org.springframework.boot \
    -DarchetypeArtifactId=spring-boot-sample-archetype
```

### Main Application Class
```java
@SpringBootApplication    // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

### Common Starters
| Starter | Includes |
|---------|----------|
| `spring-boot-starter` | Core, logging, auto-config |
| `spring-boot-starter-web` | MVC, Tomcat, Jackson |
| `spring-boot-starter-data-jpa` | JPA, Hibernate, HikariCP |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-test` | JUnit, Mockito, AssertJ |
| `spring-boot-starter-validation` | Bean validation (Hibernate Validator) |
| `spring-boot-starter-actuator` | Health, metrics, monitoring |
| `spring-boot-starter-cache` | Caching abstraction |
| `spring-boot-starter-mail` | Email |
| `spring-boot-starter-amqp` | RabbitMQ |
| `spring-boot-starter-data-redis` | Redis |
| `spring-boot-starter-data-mongodb` | MongoDB |
| `spring-boot-starter-webflux` | Reactive web |
| `spring-boot-starter-oauth2-client` | OAuth2 login |

### application.properties / application.yml
```yaml
# application.yml
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  application:
    name: my-app
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: secret
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update     # none | validate | update | create | create-drop
    show-sql: true
    properties:
      hibernate:
        format_sql: true
  profiles:
    active: dev

logging:
  level:
    root: INFO
    com.myapp: DEBUG
    org.springframework: WARN
  file:
    name: app.log
```

### Auto-Configuration
```java
// Spring Boot auto-configures beans based on:
// 1. Classpath dependencies (JAR on classpath → auto-configure)
// 2. Properties (configure behavior)
// 3. Existing beans (don't override user-defined beans)

// Disable specific auto-configuration
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})

// See what was auto-configured
// application.properties:
debug=true    // prints auto-configuration report on startup
```

### CommandLineRunner / ApplicationRunner
```java
// Run code at startup
@Component
public class StartupRunner implements CommandLineRunner {
    @Override
    public void run(String... args) {
        System.out.println("App started with args: " + Arrays.toString(args));
    }
}

@Bean
@Order(1)
CommandLineRunner initDatabase(UserRepository repo) {
    return args -> {
        repo.save(new User("admin"));
        repo.save(new User("user"));
    };
}

// ApplicationRunner — parsed args
@Component
public class AppRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        if (args.containsOption("debug")) { ... }
    }
}
```

---

## 18. TESTING

### Test Dependencies
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<!-- Includes: JUnit 5, Mockito, AssertJ, Spring Test, Hamcrest -->
```

### Unit Test (No Spring Context)
```java
class UserServiceTest {

    private UserService userService;
    private UserRepository mockRepo;

    @BeforeEach
    void setUp() {
        mockRepo = Mockito.mock(UserRepository.class);
        userService = new UserService(mockRepo);
    }

    @Test
    void shouldFindUser() {
        when(mockRepo.findById(1L)).thenReturn(Optional.of(new User("Ritesh")));

        User user = userService.findById(1L);

        assertEquals("Ritesh", user.getName());
        verify(mockRepo).findById(1L);
    }
}
```

### Unit Test with Mockito Annotations
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepo;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService userService;      // injects mocks via constructor

    @Test
    void shouldCreateUser() {
        User user = new User("Ritesh");
        when(userRepo.save(any(User.class))).thenReturn(user);

        User created = userService.createUser("Ritesh");

        assertNotNull(created);
        assertEquals("Ritesh", created.getName());
        verify(userRepo).save(any(User.class));
        verify(emailService).sendWelcomeEmail("Ritesh");
    }

    @Test
    void shouldThrowWhenNotFound() {
        when(userRepo.findById(99L)).thenReturn(Optional.empty());

        assertThrows(UserNotFoundException.class, () -> userService.findById(99L));
    }
}
```

### Integration Test (Full Spring Context)
```java
@SpringBootTest
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;

    @Test
    void shouldCreateAndFindUser() {
        User user = userService.createUser("Ritesh");
        User found = userService.findById(user.getId());
        assertEquals("Ritesh", found.getName());
    }
}
```

### Test with Specific Context
```java
// Load specific configuration
@SpringBootTest(classes = {UserService.class, TestConfig.class})

// With active profile
@SpringBootTest
@ActiveProfiles("test")

// With properties
@SpringBootTest(properties = {"app.cache.enabled=false"})

// Random port
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)

// No web environment
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
```

### @MockBean (Replace bean in context)
```java
@SpringBootTest
class OrderServiceTest {

    @Autowired
    private OrderService orderService;

    @MockBean                           // replaces real bean in Spring context
    private PaymentService paymentService;

    @Test
    void shouldProcessOrder() {
        when(paymentService.charge(anyDouble())).thenReturn(true);
        orderService.placeOrder(new Order(100.0));
        verify(paymentService).charge(100.0);
    }
}
```

### @SpyBean (Partial Mock)
```java
@SpringBootTest
class UserServiceTest {

    @SpyBean                            // real bean, but can verify/stub
    private UserRepository userRepo;

    @Test
    void shouldCallRepo() {
        userService.findAll();
        verify(userRepo).findAll();     // verify call on real bean
    }
}
```

### Testing Properties
```java
@SpringBootTest
@TestPropertySource(properties = {
    "app.name=TestApp",
    "app.debug=true"
})
class ConfigTest { }

@SpringBootTest
@TestPropertySource(locations = "classpath:test.properties")
class ConfigTest { }
```

---

## 19. COMMON ANNOTATIONS REFERENCE

### Core Annotations
| Annotation | Purpose |
|------------|---------|
| `@Component` | Generic Spring bean |
| `@Service` | Business logic bean |
| `@Repository` | Data access bean (exception translation) |
| `@Controller` | MVC controller |
| `@RestController` | REST controller (@Controller + @ResponseBody) |
| `@Configuration` | Java config class (declares @Bean methods) |
| `@Bean` | Declare bean via factory method |
| `@ComponentScan` | Auto-detect component classes |

### Dependency Injection
| Annotation | Purpose |
|------------|---------|
| `@Autowired` | Auto-wire dependency |
| `@Qualifier("name")` | Select specific bean |
| `@Primary` | Default bean when ambiguous |
| `@Value("${prop}")` | Inject property value |
| `@Lazy` | Lazy initialization |
| `@Lookup` | Method injection for prototype in singleton |

### Lifecycle
| Annotation | Purpose |
|------------|---------|
| `@PostConstruct` | Run after DI complete |
| `@PreDestroy` | Run before bean destroyed |
| `@Scope("prototype")` | Bean scope |
| `@Order(1)` | Execution order |
| `@DependsOn("otherBean")` | Ensure other bean created first |

### Configuration
| Annotation | Purpose |
|------------|---------|
| `@PropertySource` | Load properties file |
| `@ConfigurationProperties` | Type-safe property binding |
| `@Profile("dev")` | Conditional on active profile |
| `@Conditional` | Conditional bean registration |
| `@Import` | Import other config classes |
| `@ImportResource` | Import XML config |
| `@EnableScheduling` | Enable @Scheduled |
| `@EnableAsync` | Enable @Async |
| `@EnableTransactionManagement` | Enable @Transactional |
| `@EnableAspectJAutoProxy` | Enable AOP proxies |
| `@EnableCaching` | Enable @Cacheable |

### Spring Boot
| Annotation | Purpose |
|------------|---------|
| `@SpringBootApplication` | Main application class |
| `@EnableAutoConfiguration` | Enable auto-config |
| `@ConditionalOnProperty` | Bean if property exists |
| `@ConditionalOnMissingBean` | Bean if no other exists |
| `@ConditionalOnClass` | Bean if class on classpath |
| `@ConditionalOnBean` | Bean if another bean exists |

### AOP
| Annotation | Purpose |
|------------|---------|
| `@Aspect` | Declare aspect class |
| `@Before` | Run before method |
| `@After` | Run after method (always) |
| `@AfterReturning` | Run after successful return |
| `@AfterThrowing` | Run after exception |
| `@Around` | Wrap method execution |
| `@Pointcut` | Reusable pointcut expression |

### Testing
| Annotation | Purpose |
|------------|---------|
| `@SpringBootTest` | Full integration test |
| `@MockBean` | Replace bean with mock |
| `@SpyBean` | Partial mock of real bean |
| `@ActiveProfiles` | Set test profiles |
| `@TestPropertySource` | Override properties |
| `@DataJpaTest` | JPA layer test only |
| `@WebMvcTest` | MVC layer test only |
| `@JsonTest` | JSON serialization test |

---

## 20. DESIGN PRINCIPLES & BEST PRACTICES

### SOLID with Spring
```
S — Single Responsibility
    Each @Service / @Repository handles ONE concern

O — Open/Closed
    Use interfaces + DI to extend without modifying

L — Liskov Substitution
    Program to interfaces, swap implementations via @Qualifier

I — Interface Segregation
    Small, focused interfaces (not one giant interface)

D — Dependency Inversion
    Depend on abstractions (interfaces), Spring injects implementations
```

### Best Practices
```
1. ALWAYS use constructor injection (not field injection)
2. Program to interfaces — inject interface, not implementation
3. Use @Service/@Repository/@Controller correctly (not @Component for everything)
4. Keep @Configuration classes focused (separate concerns)
5. Use profiles for environment-specific beans
6. Externalize config — never hardcode values
7. Use @ConfigurationProperties over @Value for groups of properties
8. Let Spring manage bean lifecycle — don't call new on Spring beans
9. Use @Transactional on service layer (not repository/controller)
10. Write unit tests without Spring context when possible
11. Use @MockBean sparingly — prefer constructor mocks
12. Don't catch exceptions just to rethrow — let Spring handle
13. Use @Valid for input validation
14. Keep beans stateless (singleton-safe)
15. Use @Lazy only when truly needed (large startup cost)
```

### Common Patterns
```
Controller → Service → Repository
   ↑           ↑           ↑
 @RestController  @Service   @Repository
 HTTP handling   Business    Data access
 Validation      logic       Database
 Response        Tx mgmt     Queries
```

### Anti-Patterns to Avoid
```
1. Field injection (@Autowired on fields)
2. God classes (one service doing everything)
3. Business logic in controllers
4. Catching exceptions silently
5. Using new instead of DI for Spring beans
6. Injecting ApplicationContext when a specific bean suffices
7. @Transactional on private methods (doesn't work)
8. Self-invocation of @Transactional / @Async / @Cacheable (proxy bypass)
9. Mutable beans in singleton scope (thread-safety issues)
10. Circular dependencies (redesign to eliminate)
```

---

## 21. QUICK REFERENCE CHEAT SHEET

### Application Bootstrap
```java
// Spring Boot
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}

// Pure Spring
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
MyService service = ctx.getBean(MyService.class);
```

### Bean Declaration
```java
// Auto-detected
@Component / @Service / @Repository / @Controller

// Manual
@Configuration
class Config {
    @Bean
    MyService myService() { return new MyService(); }
}
```

### Injection Pattern
```java
@Service
public class MyService {
    private final MyRepo repo;                    // 1. final field

    public MyService(MyRepo repo) {               // 2. constructor
        this.repo = repo;
    }
}
```

### Property Injection
```java
@Value("${app.name}")           // simple property
@Value("${app.port:8080}")      // with default
@ConfigurationProperties("app") // type-safe binding
```

### Lifecycle
```java
@PostConstruct → init
@PreDestroy    → cleanup
```

### Scopes
```
singleton  → one instance (default)
prototype  → new instance each time
request    → per HTTP request
session    → per HTTP session
```

### AOP Pattern
```java
@Aspect @Component
public class MyAspect {
    @Around("@annotation(MyAnnotation)")
    public Object wrap(ProceedingJoinPoint jp) throws Throwable {
        // before
        Object result = jp.proceed();
        // after
        return result;
    }
}
```

### Testing Pattern
```java
// Unit test (no Spring)
@ExtendWith(MockitoExtension.class)
class ServiceTest {
    @Mock Repository repo;
    @InjectMocks Service service;
}

// Integration test (with Spring)
@SpringBootTest
@ActiveProfiles("test")
class IntegrationTest {
    @Autowired Service service;
    @MockBean ExternalApi api;
}
```

---

*Complete Spring Core Reference — All 21 Topics*
*Last updated: April 2026*
