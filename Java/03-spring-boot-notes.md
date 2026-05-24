# Spring Boot — Complete Notes (Beginner to Advanced)

---

## Table of Contents

1. [Introduction to Spring Boot](#1-introduction-to-spring-boot)
2. [Spring Boot Project Setup](#2-spring-boot-project-setup)
3. [Spring Boot Starters](#3-spring-boot-starters)
4. [Auto-Configuration](#4-auto-configuration)
5. [Application Properties & YAML Configuration](#5-application-properties--yaml-configuration)
6. [Profiles](#6-profiles)
7. [Embedded Servers](#7-embedded-servers)
8. [Dependency Injection Recap in Boot Context](#8-dependency-injection-recap-in-boot-context)
9. [Spring Boot DevTools](#9-spring-boot-devtools)
10. [Logging](#10-logging)
11. [Lombok Integration](#11-lombok-integration)
12. [Global Exception Handling](#12-global-exception-handling)
13. [Request Validation](#13-request-validation)
14. [Spring Boot Data Access — Spring Data JPA](#14-spring-boot-data-access--spring-data-jpa)
15. [Database Migrations — Flyway & Liquibase](#15-database-migrations--flyway--liquibase)
16. [Connection Pooling — HikariCP](#16-connection-pooling--hikaricp)
17. [Transaction Management](#17-transaction-management)
18. [Spring Boot Caching](#18-spring-boot-caching)
19. [Spring Boot Security](#19-spring-boot-security)
20. [JWT Authentication](#20-jwt-authentication)
21. [Spring Boot Actuator](#21-spring-boot-actuator)
22. [Custom Health Indicators & Metrics](#22-custom-health-indicators--metrics)
23. [Spring Boot Testing](#23-spring-boot-testing)
24. [OpenAPI / Swagger Documentation](#24-openapi--swagger-documentation)
25. [Configuration Properties Binding — @ConfigurationProperties](#25-configuration-properties-binding--configurationproperties)
26. [Externalized Configuration & Config Server](#26-externalized-configuration--config-server)
27. [Event Handling](#27-event-handling)
28. [Scheduling](#28-scheduling)
29. [Async Processing](#29-async-processing)
30. [HTTP Clients — RestTemplate, WebClient & RestClient](#30-http-clients--resttemplate-webclient--restclient)
31. [Email Sending](#31-email-sending)
32. [File Upload & Download](#32-file-upload--download)
33. [Spring Boot with Docker](#33-spring-boot-with-docker)
34. [Building Fat/Uber JARs & Deployment](#34-building-fatuber-jars--deployment)
35. [Spring Boot CLI](#35-spring-boot-cli)
36. [Custom Auto-Configuration](#36-custom-auto-configuration)
37. [Spring Boot with Kafka](#37-spring-boot-with-kafka)
38. [Spring Boot with RabbitMQ](#38-spring-boot-with-rabbitmq)
39. [Spring Boot with Redis](#39-spring-boot-with-redis)
40. [Spring Boot with GraphQL](#40-spring-boot-with-graphql)
41. [Resilience — Retry & Circuit Breaker](#41-resilience--retry--circuit-breaker-resilience4j)
42. [AOP — Aspect-Oriented Programming](#42-aop--aspect-oriented-programming)
43. [Spring Boot Observability — Micrometer & Tracing](#43-spring-boot-observability--micrometer--tracing)
44. [GraalVM Native Images](#44-graalvm-native-images)
45. [Best Practices & Production Checklist](#45-best-practices--production-checklist)
46. [Important Annotations Reference](#46-important-annotations-reference)
47. [Common Configuration Properties Reference](#47-common-configuration-properties-reference)

---

## 1. INTRODUCTION TO SPRING BOOT

### What is Spring Boot?
- **Spring Boot** = an opinionated framework built **on top of Spring Framework** that simplifies bootstrapping & developing Spring applications.
- Created by **Pivotal (now VMware/Broadcom)** — first release **2014**.
- Current major version: **Spring Boot 3.x** (requires **Java 17+**, built on **Spring Framework 6.x**).

### Why Spring Boot?
| Problem (Plain Spring) | Spring Boot Solution |
|------------------------|----------------------|
| Tons of XML / Java config | **Auto-configuration** — sensible defaults |
| Manual dependency management | **Starters** — curated dependency sets |
| External server setup (Tomcat/Jetty) | **Embedded server** — run as `java -jar` |
| No standard project structure | **Convention over configuration** |
| Painful environment config | **Profiles** + externalized config |
| No production-ready features | **Actuator** — health, metrics, info |

### Spring vs Spring Boot
| Aspect | Spring Framework | Spring Boot |
|--------|-----------------|-------------|
| Configuration | Manual (XML / Java) | Auto-configured |
| Server | External (WAR deployment) | Embedded (JAR) |
| Startup | `ApplicationContext` manually | `SpringApplication.run()` |
| Dependencies | Cherry-pick each lib | Starters bundle them |
| Production features | DIY | Actuator out of the box |
| Opinionated? | No | Yes (with escape hatches) |

> **Key Insight**: Spring Boot is NOT a replacement for Spring — it's a layer on top that reduces boilerplate.

### Spring Boot Architecture
```
┌─────────────────────────────────────────────┐
│               Your Application              │
├─────────────────────────────────────────────┤
│             Spring Boot Layer               │
│  (Auto-config, Starters, Actuator, CLI)     │
├─────────────────────────────────────────────┤
│            Spring Framework                 │
│  (Core, MVC, Data, Security, AOP, etc.)     │
├─────────────────────────────────────────────┤
│         Embedded Server (Tomcat/Jetty)      │
├─────────────────────────────────────────────┤
│                   JVM                       │
└─────────────────────────────────────────────┘
```

---

## 2. SPRING BOOT PROJECT SETUP

### 2.1 Using Spring Initializr
- **URL**: [https://start.spring.io](https://start.spring.io)
- Select: Project type (Maven/Gradle), Language (Java/Kotlin/Groovy), Spring Boot version, metadata, dependencies.
- Download ZIP → extract → open in IDE.

### 2.2 Using IntelliJ / STS
- IntelliJ: `File → New → Project → Spring Initializr`
- STS (Spring Tool Suite): `File → New → Spring Starter Project`

### 2.3 Using Spring Boot CLI
```bash
# Install via SDKMAN
sdk install springboot

# Create project
spring init --dependencies=web,data-jpa,h2 --build=maven my-app
```

### 2.4 Minimal Project Structure
```
my-app/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/myapp/
│   │   │       ├── MyAppApplication.java          ← Main class
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       ├── model/                          ← Entity / DTO
│   │   │       ├── config/
│   │   │       └── exception/
│   │   └── resources/
│   │       ├── application.properties              ← Config
│   │       ├── application.yml                     ← Alt config
│   │       ├── static/                             ← CSS, JS, images
│   │       └── templates/                          ← Thymeleaf templates
│   └── test/
│       └── java/
│           └── com/example/myapp/
│               └── MyAppApplicationTests.java
├── pom.xml                                         ← Maven build file
└── mvnw / mvnw.cmd                                ← Maven wrapper
```

### 2.5 Main Application Class
```java
package com.example.myapp;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication   // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MyAppApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyAppApplication.class, args);
    }
}
```

### 2.6 Breaking Down @SpringBootApplication
| Annotation | Purpose |
|------------|---------|
| `@Configuration` | Marks class as a source of bean definitions |
| `@EnableAutoConfiguration` | Enables Spring Boot auto-configuration |
| `@ComponentScan` | Scans current package + sub-packages for `@Component`, `@Service`, `@Repository`, `@Controller` |

### 2.7 Minimal pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>my-app</name>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### 2.8 Running the Application
```bash
# Maven
./mvnw spring-boot:run

# Or build JAR and run
./mvnw clean package
java -jar target/my-app-0.0.1-SNAPSHOT.jar

# Gradle
./gradlew bootRun
```

---

## 3. SPRING BOOT STARTERS

### What are Starters?
- **Convenience dependency descriptors** — a single dependency that pulls in everything you need for a feature.
- Follow naming convention: `spring-boot-starter-*`

### Common Starters
| Starter | Includes |
|---------|----------|
| `spring-boot-starter-web` | Spring MVC, Tomcat, Jackson, Validation |
| `spring-boot-starter-data-jpa` | Spring Data JPA, Hibernate, HikariCP |
| `spring-boot-starter-security` | Spring Security, auto-config |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, MockMvc |
| `spring-boot-starter-validation` | Hibernate Validator (JSR-380) |
| `spring-boot-starter-actuator` | Health, Metrics, Info endpoints |
| `spring-boot-starter-cache` | Spring Cache abstraction |
| `spring-boot-starter-mail` | JavaMail, Spring mail integration |
| `spring-boot-starter-thymeleaf` | Thymeleaf template engine |
| `spring-boot-starter-data-redis` | Redis client + Spring Data Redis |
| `spring-boot-starter-amqp` | RabbitMQ + Spring AMQP |
| `spring-boot-starter-websocket` | WebSocket support |
| `spring-boot-starter-oauth2-client` | OAuth2 login / client |
| `spring-boot-starter-oauth2-resource-server` | JWT resource server |
| `spring-boot-starter-webflux` | Reactive web (Netty) |
| `spring-boot-starter-data-mongodb` | MongoDB support |
| `spring-boot-starter-batch` | Spring Batch |
| `spring-boot-starter-quartz` | Quartz scheduler |
| `spring-boot-starter-graphql` | Spring for GraphQL |

### Custom Starter (Advanced)
```
my-custom-starter/
├── my-custom-starter/              ← Just dependency POM
│   └── pom.xml
└── my-custom-starter-autoconfigure/
    ├── src/main/java/.../
    │   ├── MyAutoConfiguration.java
    │   └── MyProperties.java
    └── src/main/resources/
        └── META-INF/spring/
            └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

---

## 4. AUTO-CONFIGURATION

### How It Works
1. `@EnableAutoConfiguration` triggers auto-config.
2. Spring Boot scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 3.x) or `META-INF/spring.factories` (Boot 2.x).
3. Each auto-config class has **conditional annotations** — only activates if conditions are met.

### Key Conditional Annotations
| Annotation | Condition |
|------------|-----------|
| `@ConditionalOnClass` | Class is on classpath |
| `@ConditionalOnMissingClass` | Class NOT on classpath |
| `@ConditionalOnBean` | Bean already exists in context |
| `@ConditionalOnMissingBean` | Bean does NOT exist (most common) |
| `@ConditionalOnProperty` | Property has specific value |
| `@ConditionalOnResource` | Resource exists on classpath |
| `@ConditionalOnWebApplication` | It's a web application |
| `@ConditionalOnExpression` | SpEL expression evaluates to true |

### Example: How DataSource Auto-Config Works
```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)                // Only if JDBC on classpath
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                        // Only if user didn't define one
    public DataSource dataSource(DataSourceProperties props) {
        return DataSourceBuilder.create()
                .url(props.getUrl())
                .username(props.getUsername())
                .password(props.getPassword())
                .build();
    }
}
```

### Debugging Auto-Configuration
```bash
# See auto-config report
java -jar app.jar --debug

# Or in application.properties
debug=true
```
Output shows:
- **Positive matches** — auto-configs that were applied
- **Negative matches** — auto-configs that were skipped (and why)

### Excluding Auto-Configuration
```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})
public class MyApp { }
```
Or via properties:
```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## 5. APPLICATION PROPERTIES & YAML CONFIGURATION

### 5.1 Property Sources (Priority Order — highest first)
1. Command-line arguments (`--server.port=9090`)
2. `SPRING_APPLICATION_JSON` (inline JSON)
3. Servlet init parameters
4. JNDI attributes
5. Java System properties (`-Dserver.port=9090`)
6. OS environment variables (`SERVER_PORT=9090`)
7. `application-{profile}.properties` / `.yml`
8. `application.properties` / `.yml`
9. `@PropertySource` on `@Configuration` class
10. Default properties (`SpringApplication.setDefaultProperties()`)

### 5.2 application.properties Format
```properties
# Server
server.port=8081
server.servlet.context-path=/api

# DataSource
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=secret
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.database-platform=org.hibernate.dialect.MySQLDialect

# Logging
logging.level.root=INFO
logging.level.com.example=DEBUG
logging.file.name=app.log
```

### 5.3 application.yml Format (Equivalent)
```yaml
server:
  port: 8081
  servlet:
    context-path: /api

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: secret
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
    database-platform: org.hibernate.dialect.MySQLDialect

logging:
  level:
    root: INFO
    com.example: DEBUG
  file:
    name: app.log
```

### 5.4 Injecting Properties
```java
// Single value
@Value("${server.port}")
private int port;

// With default
@Value("${app.name:MyApp}")
private String appName;

// SpEL expression
@Value("#{${app.timeout:30} * 1000}")
private long timeoutMs;
```

### 5.5 Custom Properties
```properties
app.name=MyApp
app.version=1.0
app.upload-dir=/uploads
app.max-file-size=10MB
app.allowed-origins=http://localhost:3000,http://example.com
```
```java
@Value("${app.upload-dir}")
private String uploadDir;

// List injection
@Value("${app.allowed-origins}")
private List<String> allowedOrigins;
```

### 5.6 Property Placeholder in Properties
```properties
app.base-url=http://localhost:${server.port}
app.api-url=${app.base-url}/api/v1
```

### 5.7 Random Values
```properties
app.secret=${random.value}
app.number=${random.int}
app.range=${random.int(1,100)}
app.uuid=${random.uuid}
```

---

## 6. PROFILES

### What are Profiles?
- Profiles let you have **different configurations** for different environments (dev, test, staging, prod).
- Any `@Component`, `@Configuration`, or `@Bean` can be **profile-specific**.

### 6.1 Profile-Specific Property Files
```
application.properties            ← Default (always loaded)
application-dev.properties        ← Loaded when "dev" profile is active
application-prod.properties       ← Loaded when "prod" profile is active
application-test.properties       ← Loaded when "test" profile is active
```

### 6.2 Activating Profiles
```properties
# In application.properties
spring.profiles.active=dev
```
```bash
# Command line
java -jar app.jar --spring.profiles.active=prod

# Environment variable
SPRING_PROFILES_ACTIVE=prod

# JVM argument
java -Dspring.profiles.active=prod -jar app.jar
```

### 6.3 Profile-Specific Beans
```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        // H2 in-memory database
        return new EmbeddedDatabaseBuilder()
                .setType(EmbeddedDatabaseType.H2)
                .build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://prod-server:3306/mydb");
        ds.setUsername("prod_user");
        ds.setPassword("prod_pass");
        return ds;
    }
}
```

### 6.4 Profile Groups (Boot 2.4+)
```properties
# application.properties
spring.profiles.group.production=proddb,prodmq,prodmetrics
spring.profiles.group.development=devdb,devmq
```
Now `--spring.profiles.active=production` activates all three sub-profiles.

### 6.5 Default Profile
```properties
# If no profile is set, "default" profile is active
spring.profiles.default=dev
```

### 6.6 Multi-Document YAML (Profile Sections)
```yaml
# Default
server:
  port: 8080

---
spring:
  config:
    activate:
      on-profile: dev
server:
  port: 8081

---
spring:
  config:
    activate:
      on-profile: prod
server:
  port: 80
```

### 6.7 Programmatic Profile Check
```java
@Autowired
private Environment env;

public void someMethod() {
    if (env.acceptsProfiles(Profiles.of("dev"))) {
        // dev-specific logic
    }
}
```

---

## 7. EMBEDDED SERVERS

### Supported Embedded Servers
| Server | Starter Dependency | Default? |
|--------|-------------------|----------|
| **Tomcat** | `spring-boot-starter-web` | Yes |
| **Jetty** | `spring-boot-starter-jetty` | No |
| **Undertow** | `spring-boot-starter-undertow` | No |
| **Netty** | `spring-boot-starter-webflux` | Yes (reactive) |

### Switching to Jetty
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

### Common Server Properties
```properties
# Port
server.port=8080
server.port=0                           # Random available port

# SSL
server.ssl.enabled=true
server.ssl.key-store=classpath:keystore.p12
server.ssl.key-store-password=changeit
server.ssl.key-store-type=PKCS12

# Compression
server.compression.enabled=true
server.compression.mime-types=text/html,application/json
server.compression.min-response-size=1024

# Tomcat specific
server.tomcat.max-threads=200
server.tomcat.max-connections=10000
server.tomcat.accept-count=100
server.tomcat.connection-timeout=20000

# Access log
server.tomcat.accesslog.enabled=true
server.tomcat.accesslog.directory=logs
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b

# Session
server.servlet.session.timeout=30m
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.secure=true
```

### Programmatic Server Customization
```java
@Component
public class ServerConfig implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {
    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setPort(9090);
        factory.addConnectorCustomizers(connector -> {
            connector.setMaxPostSize(10 * 1024 * 1024); // 10MB
        });
    }
}
```

---

## 8. DEPENDENCY INJECTION RECAP IN BOOT CONTEXT

### Stereotype Annotations
| Annotation | Layer | Purpose |
|------------|-------|---------|
| `@Component` | Generic | General-purpose Spring bean |
| `@Service` | Service | Business logic (no extra behavior, just semantic) |
| `@Repository` | DAO | Data access (adds exception translation) |
| `@Controller` | Web | MVC controller (returns views) |
| `@RestController` | Web | `@Controller` + `@ResponseBody` |
| `@Configuration` | Config | Defines `@Bean` methods |

### Injection Types in Boot
```java
// 1. Constructor injection (RECOMMENDED)
@Service
public class UserService {
    private final UserRepository repo;

    public UserService(UserRepository repo) {  // @Autowired optional if single constructor
        this.repo = repo;
    }
}

// 2. Field injection (NOT recommended — hard to test)
@Service
public class UserService {
    @Autowired
    private UserRepository repo;
}

// 3. Setter injection
@Service
public class UserService {
    private UserRepository repo;

    @Autowired
    public void setRepo(UserRepository repo) {
        this.repo = repo;
    }
}
```

### Qualifier & Primary
```java
public interface NotificationService { void send(String msg); }

@Service("email")
public class EmailService implements NotificationService { ... }

@Service("sms")
public class SmsService implements NotificationService { ... }

// Usage
@Service
public class AlertService {
    public AlertService(@Qualifier("email") NotificationService ns) { ... }
}

// Alternative: mark one as primary
@Primary
@Service
public class EmailService implements NotificationService { ... }
```

---

## 9. SPRING BOOT DEVTOOLS

### What is DevTools?
- **Development-time** features: automatic restart, live reload, relaxed configuration.
- **Auto-disabled in production** (when running from a packaged JAR).

### Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

### Features
| Feature | Description |
|---------|-------------|
| **Automatic Restart** | Restarts app when classpath files change |
| **LiveReload** | Refreshes browser automatically |
| **Property Defaults** | Sets `spring.thymeleaf.cache=false`, etc. |
| **H2 Console** | Auto-enables `/h2-console` |
| **Remote DevTools** | Restart tunneling over HTTP (dev only!) |

### Configuration
```properties
# Disable restart
spring.devtools.restart.enabled=false

# Additional paths to watch
spring.devtools.restart.additional-paths=src/main/resources

# Exclude paths from triggering restart
spring.devtools.restart.exclude=static/**,templates/**

# Trigger file (restart only when this file changes)
spring.devtools.restart.trigger-file=.reloadtrigger

# LiveReload
spring.devtools.livereload.enabled=true
```

---

## 10. LOGGING

### Default Logging
- Spring Boot uses **SLF4J** + **Logback** by default.
- Starters include `spring-boot-starter-logging` (Logback + SLF4J + JUL bridge + Log4j bridge).

### Using Logger
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class UserService {
    private static final Logger log = LoggerFactory.getLogger(UserService.class);
    // Or with Lombok: @Slf4j on the class

    public User findById(Long id) {
        log.info("Finding user by id: {}", id);
        log.debug("Debug message with params: {} and {}", param1, param2);
        log.warn("Warning: user not found for id={}", id);
        log.error("Error occurred", exception);
        return repo.findById(id).orElse(null);
    }
}
```

### Log Levels (Low → High)
```
TRACE → DEBUG → INFO → WARN → ERROR → FATAL → OFF
```

### Configuration
```properties
# Root level
logging.level.root=INFO

# Package level
logging.level.com.example=DEBUG
logging.level.org.springframework.web=WARN
logging.level.org.hibernate.SQL=DEBUG

# Log file
logging.file.name=app.log
logging.file.path=/var/log/myapp

# File rotation
logging.logback.rollingpolicy.max-file-size=10MB
logging.logback.rollingpolicy.max-history=30
logging.logback.rollingpolicy.total-size-cap=1GB

# Pattern
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} %-5level [%thread] %logger{36} - %msg%n
logging.pattern.file=%d{yyyy-MM-dd HH:mm:ss} %-5level [%thread] %logger{36} - %msg%n
```

### Custom Logback Configuration
Create `logback-spring.xml` in `src/main/resources/`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %highlight(%-5level) [%thread] %cyan(%logger{36}) - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/app.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>10MB</maxFileSize>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- Profile-specific logging -->
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>
</configuration>
```

### Switching to Log4j2
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

---

## 11. LOMBOK INTEGRATION

### What is Lombok?
- **Project Lombok** = a Java library that auto-generates boilerplate code (getters, setters, constructors, `toString`, `equals`, `hashCode`, builders, loggers) at **compile time** via annotations.
- Used in **90%+ of real-world Spring Boot projects**.

### Setup
```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```
> **IDE Plugin Required**: Install Lombok plugin in IntelliJ / Eclipse for IDE support.

### Common Annotations
| Annotation | Generates |
|------------|-----------|
| `@Getter` / `@Setter` | Getters / Setters for all fields |
| `@ToString` | `toString()` method |
| `@EqualsAndHashCode` | `equals()` and `hashCode()` |
| `@NoArgsConstructor` | No-argument constructor |
| `@AllArgsConstructor` | All-fields constructor |
| `@RequiredArgsConstructor` | Constructor for `final` / `@NonNull` fields |
| `@Data` | `@Getter` + `@Setter` + `@ToString` + `@EqualsAndHashCode` + `@RequiredArgsConstructor` |
| `@Builder` | Builder pattern |
| `@Slf4j` / `@Log4j2` | Logger field (`log`) |
| `@Value` (Lombok) | Immutable class (all fields `private final`, no setters) |

### Real-World Usage Examples
```java
// Entity with Lombok
@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
}

// DTO with Builder
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserDTO {
    private String name;
    private String email;
    private String role;
}

// Creating objects
User user = User.builder()
        .name("John")
        .email("john@example.com")
        .build();

// Service with Slf4j
@Slf4j
@Service
@RequiredArgsConstructor   // Generates constructor for final fields → auto DI
public class UserService {
    private final UserRepository userRepo;    // Injected via generated constructor
    private final PasswordEncoder encoder;

    public User create(UserDTO dto) {
        log.info("Creating user: {}", dto.getEmail());   // log field auto-generated
        return userRepo.save(User.builder()
                .name(dto.getName())
                .email(dto.getEmail())
                .build());
    }
}
```

### ⚠️ Lombok Pitfalls
```java
// ❌ Don't use @Data on JPA entities with relationships (breaks equals/hashCode)
// ✅ Use @Getter @Setter @ToString(exclude = "orders") on entities

@Entity
@Getter @Setter
@NoArgsConstructor
@ToString(exclude = "orders")
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class User {
    @Id
    @EqualsAndHashCode.Include
    private Long id;

    private String name;

    @OneToMany(mappedBy = "user")
    private List<Order> orders;   // Excluded from toString/equals to avoid infinite loops
}
```

### Exclude from Maven Build (Don't ship Lombok in JAR)
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <excludes>
            <exclude>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
            </exclude>
        </excludes>
    </configuration>
</plugin>
```

---

## 12. GLOBAL EXCEPTION HANDLING

### Why?
- **Every real Spring Boot REST API** needs centralized exception handling.
- Without it, Spring returns generic error pages / 500 errors with stack traces.
- `@RestControllerAdvice` catches exceptions across **all controllers** in one place.

### 12.1 Custom Exception Classes
```java
// Base exception
public class AppException extends RuntimeException {
    private final HttpStatus status;

    public AppException(String message, HttpStatus status) {
        super(message);
        this.status = status;
    }

    public HttpStatus getStatus() { return status; }
}

// Specific exceptions
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String resource, Long id) {
        super(resource + " not found with id: " + id, HttpStatus.NOT_FOUND);
    }
}

public class DuplicateResourceException extends AppException {
    public DuplicateResourceException(String message) {
        super(message, HttpStatus.CONFLICT);
    }
}

public class BadRequestException extends AppException {
    public BadRequestException(String message) {
        super(message, HttpStatus.BAD_REQUEST);
    }
}
```

### 12.2 Standard Error Response DTO
```java
@Data
@Builder
@AllArgsConstructor
public class ErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private Map<String, String> fieldErrors;   // For validation errors
}
```

### 12.3 Global Exception Handler
```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // Handle custom app exceptions
    @ExceptionHandler(AppException.class)
    public ResponseEntity<ErrorResponse> handleAppException(AppException ex,
                                                             HttpServletRequest request) {
        log.error("AppException: {}", ex.getMessage());
        ErrorResponse error = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(ex.getStatus().value())
                .error(ex.getStatus().getReasonPhrase())
                .message(ex.getMessage())
                .path(request.getRequestURI())
                .build();
        return new ResponseEntity<>(error, ex.getStatus());
    }

    // Handle validation errors (@Valid)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex,
                                                           HttpServletRequest request) {
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
                .forEach(fe -> fieldErrors.put(fe.getField(), fe.getDefaultMessage()));

        ErrorResponse error = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.BAD_REQUEST.value())
                .error("Validation Failed")
                .message("Invalid request data")
                .path(request.getRequestURI())
                .fieldErrors(fieldErrors)
                .build();
        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }

    // Handle missing request parameters
    @ExceptionHandler(MissingServletRequestParameterException.class)
    public ResponseEntity<ErrorResponse> handleMissingParam(
            MissingServletRequestParameterException ex, HttpServletRequest request) {
        ErrorResponse error = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.BAD_REQUEST.value())
                .error("Bad Request")
                .message("Missing parameter: " + ex.getParameterName())
                .path(request.getRequestURI())
                .build();
        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }

    // Handle access denied
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex,
                                                             HttpServletRequest request) {
        ErrorResponse error = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.FORBIDDEN.value())
                .error("Forbidden")
                .message("Access denied")
                .path(request.getRequestURI())
                .build();
        return new ResponseEntity<>(error, HttpStatus.FORBIDDEN);
    }

    // Catch-all for unexpected errors
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex,
                                                        HttpServletRequest request) {
        log.error("Unexpected error: ", ex);
        ErrorResponse error = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
                .error("Internal Server Error")
                .message("An unexpected error occurred")
                .path(request.getRequestURI())
                .build();
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

### 12.4 Using Exceptions in Service Layer
```java
@Service
public class UserService {

    public User findById(Long id) {
        return userRepo.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }

    public User create(UserDTO dto) {
        if (userRepo.existsByEmail(dto.getEmail())) {
            throw new DuplicateResourceException("Email already exists: " + dto.getEmail());
        }
        return userRepo.save(mapper.toEntity(dto));
    }
}
```

### 12.5 Problem Details — RFC 9457 (Spring Boot 3.x / Spring 6+)
```java
// Enable RFC 9457 Problem Details format
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
                HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Resource Not Found");
        pd.setProperty("timestamp", Instant.now());
        return pd;
    }
}
```
```properties
# Enable Problem Details globally
spring.mvc.problemdetails.enabled=true
```

---

## 13. REQUEST VALIDATION

### Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### 13.1 Validating Request Body
```java
// DTO with validation constraints
@Data
public class CreateUserRequest {

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be 2-100 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d).*$",
             message = "Password must contain uppercase, lowercase, and digit")
    private String password;

    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 150, message = "Age must be at most 150")
    private Integer age;

    @NotNull(message = "Role is required")
    private Role role;

    @NotEmpty(message = "At least one phone number required")
    private List<@Pattern(regexp = "^\\+?[0-9]{10,15}$") String> phoneNumbers;
}

// Controller — @Valid triggers validation
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<User> create(@Valid @RequestBody CreateUserRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(userService.create(request));
    }

    // @Validated on path/query params
    @GetMapping("/{id}")
    public User getById(@PathVariable @Min(1) Long id) {
        return userService.findById(id);
    }
}
```

### 13.2 Validation Annotations Reference
| Annotation | Applies To | Checks |
|------------|-----------|--------|
| `@NotNull` | Any type | Not null |
| `@NotBlank` | String | Not null, not empty, not whitespace-only |
| `@NotEmpty` | String/Collection/Map | Not null, not empty |
| `@Size(min, max)` | String/Collection | Length/size within bounds |
| `@Min(value)` | Number | >= value |
| `@Max(value)` | Number | <= value |
| `@Email` | String | Valid email format |
| `@Pattern(regexp)` | String | Matches regex |
| `@Positive` / `@PositiveOrZero` | Number | > 0 / >= 0 |
| `@Negative` / `@NegativeOrZero` | Number | < 0 / <= 0 |
| `@Past` / `@PastOrPresent` | Date/Time | In the past |
| `@Future` / `@FutureOrPresent` | Date/Time | In the future |
| `@DecimalMin` / `@DecimalMax` | Number | Decimal bounds |
| `@Digits(integer, fraction)` | Number | Digit constraints |

### 13.3 Custom Validator
```java
// 1. Define annotation
@Documented
@Constraint(validatedBy = UniqueEmailValidator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// 2. Implement validator
@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    @Autowired
    private UserRepository userRepo;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext ctx) {
        if (email == null) return true;   // @NotBlank handles null
        return !userRepo.existsByEmail(email);
    }
}

// 3. Use on DTO
public class CreateUserRequest {
    @UniqueEmail
    @Email
    private String email;
}
```

### 13.4 Validation Groups
```java
// Define groups
public interface OnCreate {}
public interface OnUpdate {}

// DTO
public class UserRequest {
    @Null(groups = OnCreate.class)         // ID must be null on create
    @NotNull(groups = OnUpdate.class)      // ID required on update
    private Long id;

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    private String name;
}

// Controller — use @Validated (not @Valid) for groups
@PostMapping
public User create(@Validated(OnCreate.class) @RequestBody UserRequest req) { ... }

@PutMapping("/{id}")
public User update(@Validated(OnUpdate.class) @RequestBody UserRequest req) { ... }
```

### 13.5 Nested Object Validation
```java
public class OrderRequest {
    @Valid                           // Must add @Valid to trigger nested validation
    @NotNull
    private AddressDTO shippingAddress;

    @Valid
    @NotEmpty
    private List<OrderItemDTO> items;
}

public class AddressDTO {
    @NotBlank
    private String street;
    @NotBlank
    private String city;
    @Pattern(regexp = "^[0-9]{5,6}$")
    private String zipCode;
}
```

---

## 14. SPRING BOOT DATA ACCESS — SPRING DATA JPA

### 14.1 Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
<!-- Or H2 for dev -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

### 14.2 Entity
```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Enumerated(EnumType.STRING)
    private Role role;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
    }

    // Constructors, getters, setters (or use Lombok @Data)
}

public enum Role { ADMIN, USER, MODERATOR }
```

### 14.3 Repository
```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Derived query methods (auto-implemented by Spring Data)
    Optional<User> findByEmail(String email);
    List<User> findByRole(Role role);
    List<User> findByNameContainingIgnoreCase(String name);
    List<User> findByCreatedAtAfter(LocalDateTime date);
    boolean existsByEmail(String email);
    long countByRole(Role role);
    List<User> findTop5ByOrderByCreatedAtDesc();

    // Custom JPQL
    @Query("SELECT u FROM User u WHERE u.email LIKE %:domain")
    List<User> findByEmailDomain(@Param("domain") String domain);

    // Native SQL
    @Query(value = "SELECT * FROM users WHERE role = :role", nativeQuery = true)
    List<User> findByRoleNative(@Param("role") String role);

    // Modifying queries
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.role = :role WHERE u.id = :id")
    int updateUserRole(@Param("id") Long id, @Param("role") Role role);

    // Pagination
    Page<User> findByRole(Role role, Pageable pageable);

    // Sorting
    List<User> findByRole(Role role, Sort sort);
}
```

### 14.4 Query Method Keywords
| Keyword | Example | JPQL Equivalent |
|---------|---------|-----------------|
| `findBy` | `findByName(String)` | `WHERE name = ?` |
| `And` | `findByNameAndEmail(...)` | `WHERE name = ? AND email = ?` |
| `Or` | `findByNameOrEmail(...)` | `WHERE name = ? OR email = ?` |
| `Between` | `findByAgeBetween(int, int)` | `WHERE age BETWEEN ? AND ?` |
| `LessThan` | `findByAgeLessThan(int)` | `WHERE age < ?` |
| `GreaterThanEqual` | `findByAgeGreaterThanEqual` | `WHERE age >= ?` |
| `Like` | `findByNameLike(String)` | `WHERE name LIKE ?` |
| `Containing` | `findByNameContaining(String)` | `WHERE name LIKE %?%` |
| `StartingWith` | `findByNameStartingWith` | `WHERE name LIKE ?%` |
| `In` | `findByRoleIn(List)` | `WHERE role IN (?)` |
| `OrderBy` | `findByRoleOrderByNameAsc` | `ORDER BY name ASC` |
| `Not` | `findByRoleNot(Role)` | `WHERE role <> ?` |
| `IsNull` | `findByEmailIsNull()` | `WHERE email IS NULL` |
| `True/False` | `findByActiveTrue()` | `WHERE active = true` |
| `Top/First` | `findTop10ByOrderByIdDesc` | `LIMIT 10` |

### 14.5 Pagination & Sorting
```java
@Service
public class UserService {
    @Autowired
    private UserRepository repo;

    public Page<User> getUsers(int page, int size, String sortBy) {
        Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy).descending());
        return repo.findAll(pageable);
    }
}

// Controller
@GetMapping("/users")
public Page<User> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(defaultValue = "id") String sortBy) {
    return userService.getUsers(page, size, sortBy);
}
```

### 14.6 Specifications (Dynamic Queries)
```java
public class UserSpecifications {

    public static Specification<User> hasRole(Role role) {
        return (root, query, cb) -> cb.equal(root.get("role"), role);
    }

    public static Specification<User> nameContains(String name) {
        return (root, query, cb) -> cb.like(
            cb.lower(root.get("name")), "%" + name.toLowerCase() + "%"
        );
    }

    public static Specification<User> createdAfter(LocalDateTime date) {
        return (root, query, cb) -> cb.greaterThan(root.get("createdAt"), date);
    }
}

// Repository must extend JpaSpecificationExecutor
public interface UserRepository extends JpaRepository<User, Long>,
                                         JpaSpecificationExecutor<User> { }

// Usage
Specification<User> spec = Specification
    .where(UserSpecifications.hasRole(Role.ADMIN))
    .and(UserSpecifications.nameContains("john"));

List<User> users = userRepo.findAll(spec);
```

### 14.7 Projections
```java
// Interface-based projection
public interface UserSummary {
    String getName();
    String getEmail();
}

// In repository
List<UserSummary> findByRole(Role role);

// Class-based projection (DTO)
public record UserDTO(String name, String email) { }

@Query("SELECT new com.example.dto.UserDTO(u.name, u.email) FROM User u WHERE u.role = :role")
List<UserDTO> findUserDTOsByRole(@Param("role") Role role);
```

### 14.8 Auditing
```java
@Configuration
@EnableJpaAuditing
public class JpaConfig { }

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {
    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}

// Implement AuditorAware for createdBy/updatedBy
@Component
public class AuditorAwareImpl implements AuditorAware<String> {
    @Override
    public Optional<String> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext())
                .map(ctx -> ctx.getAuthentication())
                .map(auth -> auth.getName());
    }
}

// Entity extends Auditable
@Entity
public class User extends Auditable { ... }
```

### 14.9 ddl-auto Options
| Value | Behavior |
|-------|----------|
| `none` | No schema management |
| `validate` | Validate schema matches entities (prod recommended) |
| `update` | Update schema (add columns, don't drop) |
| `create` | Drop & create on startup |
| `create-drop` | Drop & create on startup, drop on shutdown |

---

## 15. DATABASE MIGRATIONS — FLYWAY & LIQUIBASE

### 15.1 Flyway
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

Migration files in `src/main/resources/db/migration/`:
```
V1__Create_users_table.sql
V2__Add_email_column.sql
V3__Create_orders_table.sql
```

```sql
-- V1__Create_users_table.sql
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    role VARCHAR(20) DEFAULT 'USER',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

```properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true
```

### 15.2 Liquibase
```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

`src/main/resources/db/changelog/db.changelog-master.yaml`:
```yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: dev
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: name
                  type: VARCHAR(100)
                  constraints:
                    nullable: false
```

---

## 16. CONNECTION POOLING — HIKARICP

### Default Pool (HikariCP)
Spring Boot uses **HikariCP** by default — the fastest JDBC connection pool.

```properties
# Pool settings
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.idle-timeout=300000
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.pool-name=MyHikariPool
spring.datasource.hikari.leak-detection-threshold=60000
```

### Multiple DataSources
```java
@Configuration
public class DataSourceConfig {

    @Primary
    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

---

## 17. TRANSACTION MANAGEMENT

### @Transactional
```java
@Service
public class OrderService {

    @Transactional  // Method runs in a DB transaction
    public void placeOrder(OrderRequest request) {
        Order order = orderRepo.save(new Order(request));
        inventoryService.reduceStock(request.getProductId(), request.getQuantity());
        paymentService.processPayment(request.getPaymentDetails());
        // If any step throws → entire transaction rolls back
    }

    @Transactional(readOnly = true)  // Optimization hint
    public List<Order> getAllOrders() {
        return orderRepo.findAll();
    }

    @Transactional(
        propagation = Propagation.REQUIRED,      // Default
        isolation = Isolation.READ_COMMITTED,
        timeout = 30,
        rollbackFor = Exception.class,
        noRollbackFor = MailException.class
    )
    public void complexOperation() { ... }
}
```

### Propagation Types
| Type | Behavior |
|------|----------|
| `REQUIRED` | Join existing TX or create new (default) |
| `REQUIRES_NEW` | Always create new TX (suspend existing) |
| `SUPPORTS` | Join existing TX or run without TX |
| `NOT_SUPPORTED` | Run without TX (suspend existing) |
| `MANDATORY` | Must have existing TX or throw exception |
| `NEVER` | Must NOT have TX or throw exception |
| `NESTED` | Nested TX (savepoint) within existing |

### Isolation Levels
| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|-------------------|-------------|
| `READ_UNCOMMITTED` | Yes | Yes | Yes |
| `READ_COMMITTED` | No | Yes | Yes |
| `REPEATABLE_READ` | No | No | Yes |
| `SERIALIZABLE` | No | No | No |

### Common Pitfall
```java
// ❌ WRONG — @Transactional on private/self-call won't work (no proxy)
@Service
public class UserService {
    public void methodA() {
        this.methodB();  // Self-call bypasses proxy!
    }

    @Transactional
    public void methodB() { ... }
}

// ✅ FIX — Inject self or extract to another bean
@Service
public class UserService {
    @Autowired
    private UserService self;  // Proxy injection

    public void methodA() {
        self.methodB();  // Goes through proxy
    }
}
```

---

## 18. SPRING BOOT CACHING

### Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

### Enable & Use
```java
@SpringBootApplication
@EnableCaching
public class MyApp { }

@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        log.info("Fetching from DB...");  // Only called on cache miss
        return productRepo.findById(id).orElseThrow();
    }

    @Cacheable(value = "products", key = "#name", condition = "#name.length() > 3")
    public List<Product> findByName(String name) { ... }

    @CachePut(value = "products", key = "#product.id")  // Always executes, updates cache
    public Product updateProduct(Product product) {
        return productRepo.save(product);
    }

    @CacheEvict(value = "products", key = "#id")   // Remove from cache
    public void deleteProduct(Long id) {
        productRepo.deleteById(id);
    }

    @CacheEvict(value = "products", allEntries = true)  // Clear entire cache
    public void clearCache() { }
}
```

### Cache Providers
| Provider | Dependency | Use Case |
|----------|-----------|----------|
| ConcurrentHashMap | Default (no extra dep) | Dev / simple apps |
| Caffeine | `com.github.ben-manes.caffeine:caffeine` | High-performance local |
| Redis | `spring-boot-starter-data-redis` | Distributed cache |
| EhCache | `org.ehcache:ehcache` | Feature-rich local |
| Hazelcast | `com.hazelcast:hazelcast` | Distributed |

### Caffeine Cache Config
```properties
spring.cache.type=caffeine
spring.cache.caffeine.spec=maximumSize=500,expireAfterWrite=10m
```
```java
@Configuration
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager("products", "users");
        manager.setCaffeine(Caffeine.newBuilder()
                .maximumSize(1000)
                .expireAfterWrite(Duration.ofMinutes(10))
                .recordStats());
        return manager;
    }
}
```

---

## 19. SPRING BOOT SECURITY

### 19.1 Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```
> Adding this dependency **immediately secures all endpoints** with HTTP Basic auth. Default user: `user`, password: printed in console.

### 19.2 Security Filter Chain (Spring Security 6.x / Boot 3.x)
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())  // Disable for REST APIs
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**", "/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())        // HTTP Basic
            .formLogin(form -> form                       // Form login
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout")
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)  // For REST APIs
            );
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 19.3 In-Memory Authentication (Dev/Testing)
```java
@Bean
public UserDetailsService userDetailsService(PasswordEncoder encoder) {
    UserDetails admin = User.builder()
            .username("admin")
            .password(encoder.encode("admin123"))
            .roles("ADMIN")
            .build();

    UserDetails user = User.builder()
            .username("user")
            .password(encoder.encode("user123"))
            .roles("USER")
            .build();

    return new InMemoryUserDetailsManager(admin, user);
}
```

### 19.4 Database Authentication
```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepo;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        com.example.model.User user = userRepo.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        return org.springframework.security.core.userdetails.User.builder()
                .username(user.getUsername())
                .password(user.getPassword())  // Must be BCrypt encoded
                .roles(user.getRole().name())
                .build();
    }
}
```

### 19.5 Method-Level Security
```java
@Configuration
@EnableMethodSecurity   // Replaces @EnableGlobalMethodSecurity in Boot 3
public class MethodSecurityConfig { }

@Service
public class AdminService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long id) { ... }

    @PreAuthorize("hasRole('ADMIN') or #username == authentication.name")
    public User getUser(String username) { ... }

    @PostAuthorize("returnObject.username == authentication.name")
    public User getProfile(Long id) { ... }

    @PreAuthorize("@customSecurity.canAccess(#id)")
    public Resource getResource(Long id) { ... }
}

// Custom security expression
@Component("customSecurity")
public class CustomSecurity {
    public boolean canAccess(Long resourceId) {
        // Custom logic
        return true;
    }
}
```

### 19.6 CORS Configuration with Security
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.cors(cors -> cors.configurationSource(corsConfigSource()));
    // ... rest of config
    return http.build();
}

@Bean
public CorsConfigurationSource corsConfigSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("http://localhost:3000"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

---

## 20. JWT AUTHENTICATION

### 20.1 Dependencies
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

### 20.2 JWT Utility
```java
@Component
public class JwtUtil {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration:86400000}")  // 24 hours default
    private long expiration;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }

    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList());

        return Jwts.builder()
                .claims(claims)
                .subject(userDetails.getUsername())
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(getSigningKey())
                .compact();
    }

    public String extractUsername(String token) {
        return extractClaims(token).getSubject();
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractClaims(token).getExpiration().before(new Date());
    }

    private Claims extractClaims(String token) {
        return Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

### 20.3 JWT Authentication Filter
```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Autowired private JwtUtil jwtUtil;
    @Autowired private UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7);
        String username = jwtUtil.extractUsername(token);

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            if (jwtUtil.isTokenValid(token, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        chain.doFilter(request, response);
    }
}
```

### 20.4 Security Config with JWT
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired private JwtAuthFilter jwtAuthFilter;
    @Autowired private UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
            throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 20.5 Auth Controller
```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired private AuthenticationManager authManager;
    @Autowired private JwtUtil jwtUtil;
    @Autowired private UserDetailsService userDetailsService;

    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request) {
        authManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.getUsername(), request.getPassword())
        );

        UserDetails userDetails = userDetailsService.loadUserByUsername(request.getUsername());
        String token = jwtUtil.generateToken(userDetails);

        return ResponseEntity.ok(Map.of("token", token));
    }
}
```

---

## 21. SPRING BOOT ACTUATOR

### Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Key Endpoints
| Endpoint | Description |
|----------|-------------|
| `/actuator/health` | App health status |
| `/actuator/info` | App info |
| `/actuator/metrics` | Metrics (JVM, HTTP, etc.) |
| `/actuator/metrics/{name}` | Specific metric |
| `/actuator/env` | Environment properties |
| `/actuator/beans` | All beans in context |
| `/actuator/mappings` | All request mappings |
| `/actuator/loggers` | Logger levels |
| `/actuator/loggers/{name}` | Get/set specific logger level |
| `/actuator/threaddump` | Thread dump |
| `/actuator/heapdump` | Heap dump (binary) |
| `/actuator/scheduledtasks` | Scheduled tasks |
| `/actuator/caches` | Cache info |
| `/actuator/configprops` | Configuration properties |
| `/actuator/flyway` | Flyway migration info |
| `/actuator/prometheus` | Prometheus metrics (if micrometer-prometheus added) |

### Configuration
```properties
# Expose endpoints
management.endpoints.web.exposure.include=health,info,metrics,loggers,env
management.endpoints.web.exposure.include=*           # Expose all (dev only!)
management.endpoints.web.exposure.exclude=heapdump

# Base path
management.endpoints.web.base-path=/actuator

# Health details
management.endpoint.health.show-details=always        # always | when_authorized | never

# Info
management.info.env.enabled=true
info.app.name=My App
info.app.version=1.0.0
info.app.description=Spring Boot Application

# Separate port for actuator
management.server.port=8081
management.server.address=127.0.0.1

# Security — actuator endpoints should be protected in production!
```

### Dynamically Change Log Level
```bash
# Get current level
curl http://localhost:8080/actuator/loggers/com.example

# Change level at runtime
curl -X POST http://localhost:8080/actuator/loggers/com.example \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'
```

---

## 22. CUSTOM HEALTH INDICATORS & METRICS

### Custom Health Indicator
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Autowired
    private DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(1)) {
                return Health.up()
                        .withDetail("database", "Reachable")
                        .withDetail("type", conn.getMetaData().getDatabaseProductName())
                        .build();
            }
        } catch (SQLException e) {
            return Health.down()
                    .withDetail("database", "Unreachable")
                    .withException(e)
                    .build();
        }
        return Health.down().build();
    }
}
```

### Custom Metrics with Micrometer
```java
@Service
public class OrderService {

    private final Counter orderCounter;
    private final Timer orderTimer;
    private final AtomicInteger activeOrders;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.placed.total")
                .description("Total orders placed")
                .tag("type", "all")
                .register(registry);

        this.orderTimer = Timer.builder("orders.processing.time")
                .description("Order processing time")
                .register(registry);

        this.activeOrders = registry.gauge("orders.active",
                new AtomicInteger(0));
    }

    public Order placeOrder(OrderRequest request) {
        return orderTimer.record(() -> {
            activeOrders.incrementAndGet();
            try {
                Order order = processOrder(request);
                orderCounter.increment();
                return order;
            } finally {
                activeOrders.decrementAndGet();
            }
        });
    }
}
```

---

## 23. SPRING BOOT TESTING

### 23.1 Test Dependencies (Included with starter-test)
| Library | Purpose |
|---------|---------|
| JUnit 5 | Testing framework |
| Mockito | Mocking |
| AssertJ | Fluent assertions |
| Hamcrest | Matchers |
| JSONassert | JSON assertions |
| JsonPath | JSON path expressions |
| MockMvc | MVC testing without server |

### 23.2 Test Slices
| Annotation | What It Loads |
|------------|---------------|
| `@SpringBootTest` | Full application context |
| `@WebMvcTest` | Only web layer (controllers, filters) |
| `@DataJpaTest` | Only JPA layer (repos, entities, H2) |
| `@RestClientTest` | RestTemplate/WebClient testing |
| `@JsonTest` | JSON serialization/deserialization |

### 23.3 Unit Test (Service Layer)
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepo;

    @InjectMocks
    private UserService userService;

    @Test
    void findById_existingUser_returnsUser() {
        User expected = new User(1L, "John", "john@example.com");
        when(userRepo.findById(1L)).thenReturn(Optional.of(expected));

        User result = userService.findById(1L);

        assertThat(result).isNotNull();
        assertThat(result.getName()).isEqualTo("John");
        verify(userRepo, times(1)).findById(1L);
    }

    @Test
    void findById_nonExistent_throwsException() {
        when(userRepo.findById(99L)).thenReturn(Optional.empty());

        assertThrows(UserNotFoundException.class, () -> userService.findById(99L));
    }
}
```

### 23.4 Controller Test (@WebMvcTest)
```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void getUser_returns200() throws Exception {
        User user = new User(1L, "John", "john@example.com");
        when(userService.findById(1L)).thenReturn(user);

        mockMvc.perform(get("/api/users/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("John"))
                .andExpect(jsonPath("$.email").value("john@example.com"));
    }

    @Test
    void createUser_returns201() throws Exception {
        User user = new User(null, "John", "john@example.com");
        User saved = new User(1L, "John", "john@example.com");
        when(userService.create(any(User.class))).thenReturn(saved);

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(user)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(1));
    }

    @Test
    void getUser_notFound_returns404() throws Exception {
        when(userService.findById(99L)).thenThrow(new UserNotFoundException(99L));

        mockMvc.perform(get("/api/users/99"))
                .andExpect(status().isNotFound());
    }
}
```

### 23.5 Repository Test (@DataJpaTest)
```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepo;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void findByEmail_existingEmail_returnsUser() {
        User user = new User(null, "John", "john@example.com");
        entityManager.persistAndFlush(user);

        Optional<User> found = userRepo.findByEmail("john@example.com");

        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John");
    }
}
```

### 23.6 Integration Test (@SpringBootTest)
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void createAndGetUser() {
        User user = new User(null, "John", "john@example.com");

        ResponseEntity<User> createResponse = restTemplate
                .postForEntity("/api/users", user, User.class);
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        Long id = createResponse.getBody().getId();

        ResponseEntity<User> getResponse = restTemplate
                .getForEntity("/api/users/" + id, User.class);
        assertThat(getResponse.getBody().getName()).isEqualTo("John");
    }
}
```

### 23.7 Testcontainers (Real DB in Tests)
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mysql</artifactId>
    <scope>test</scope>
</dependency>
```
```java
@SpringBootTest
@Testcontainers
class UserServiceIntegrationTest {

    @Container
    @ServiceConnection   // Boot 3.1+ auto-configures datasource
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

    @Autowired
    private UserService userService;

    @Test
    void crudOperations() {
        // Tests against real MySQL in Docker
    }
}
```

---

## 24. OPENAPI / SWAGGER DOCUMENTATION

### Why?
- **API documentation** is an industry standard for REST APIs.
- **OpenAPI 3.0** (formerly Swagger) generates interactive docs at `/swagger-ui.html`.
- Clients (frontend devs, mobile devs, QA) use it to understand and test endpoints.

### Setup (SpringDoc — Recommended for Boot 3.x)
```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```
> After adding, Swagger UI is available at: `http://localhost:8080/swagger-ui.html`
> OpenAPI JSON spec at: `http://localhost:8080/v3/api-docs`

### Configuration Properties
```properties
# Custom path
springdoc.swagger-ui.path=/swagger-ui.html
springdoc.api-docs.path=/v3/api-docs

# Show actuator endpoints
springdoc.show-actuator=false

# Group APIs
springdoc.group-configs[0].group=public
springdoc.group-configs[0].paths-to-match=/api/public/**
springdoc.group-configs[1].group=admin
springdoc.group-configs[1].paths-to-match=/api/admin/**

# Disable in production
springdoc.api-docs.enabled=true
springdoc.swagger-ui.enabled=true
```

### Global API Info
```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("My App API")
                        .version("1.0.0")
                        .description("REST API documentation")
                        .contact(new Contact()
                                .name("Dev Team")
                                .email("dev@example.com"))
                        .license(new License()
                                .name("Apache 2.0")))
                .addSecurityItem(new SecurityRequirement().addList("Bearer Auth"))
                .components(new Components()
                        .addSecuritySchemes("Bearer Auth",
                                new SecurityScheme()
                                        .type(SecurityScheme.Type.HTTP)
                                        .scheme("bearer")
                                        .bearerFormat("JWT")));
    }
}
```

### Annotating Controllers
```java
@RestController
@RequestMapping("/api/users")
@Tag(name = "User", description = "User management APIs")
public class UserController {

    @Operation(
        summary = "Get user by ID",
        description = "Returns a single user by their ID"
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "User found",
                content = @Content(schema = @Schema(implementation = UserDTO.class))),
        @ApiResponse(responseCode = "404", description = "User not found"),
        @ApiResponse(responseCode = "401", description = "Unauthorized")
    })
    @GetMapping("/{id}")
    public UserDTO getById(@Parameter(description = "User ID") @PathVariable Long id) {
        return userService.findById(id);
    }

    @Operation(summary = "Create a new user")
    @PostMapping
    public ResponseEntity<UserDTO> create(
            @io.swagger.v3.oas.annotations.parameters.RequestBody(
                description = "User to create",
                required = true,
                content = @Content(schema = @Schema(implementation = CreateUserRequest.class)))
            @Valid @RequestBody CreateUserRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(userService.create(request));
    }

    @Operation(summary = "List users with pagination")
    @GetMapping
    public Page<UserDTO> list(
            @Parameter(description = "Page number (0-based)") @RequestParam(defaultValue = "0") int page,
            @Parameter(description = "Page size") @RequestParam(defaultValue = "10") int size) {
        return userService.findAll(page, size);
    }
}
```

### Annotating DTOs
```java
@Data
@Schema(description = "User response")
public class UserDTO {
    @Schema(description = "User ID", example = "1")
    private Long id;

    @Schema(description = "Full name", example = "John Doe")
    private String name;

    @Schema(description = "Email address", example = "john@example.com")
    private String email;
}
```

### Hide Endpoints
```java
@Operation(hidden = true)   // Hide specific endpoint
@GetMapping("/internal")
public String internal() { ... }

// Or hide entire controller
@Hidden
@RestController
public class InternalController { ... }
```

### Secure Swagger in Production
```properties
# application-prod.properties
springdoc.api-docs.enabled=false
springdoc.swagger-ui.enabled=false
```

---

## 25. CONFIGURATION PROPERTIES BINDING — @ConfigurationProperties

### Type-Safe Configuration
```java
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {

    @NotBlank
    private String name;

    @Min(1) @Max(65535)
    private int port = 8080;

    private final Security security = new Security();
    private final Upload upload = new Upload();

    public static class Security {
        private String jwtSecret;
        private long jwtExpiration = 86400000;
        private List<String> allowedOrigins = new ArrayList<>();
        // getters & setters
    }

    public static class Upload {
        private String directory = "/uploads";
        private DataSize maxFileSize = DataSize.ofMegabytes(10);
        // getters & setters
    }

    // getters & setters
}
```

```properties
app.name=MyApp
app.port=9090
app.security.jwt-secret=mySecretKey123
app.security.jwt-expiration=86400000
app.security.allowed-origins=http://localhost:3000,http://example.com
app.upload.directory=/data/uploads
app.upload.max-file-size=25MB
```

### Enable
```java
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
// OR @ConfigurationPropertiesScan
public class MyApp { }
```

### Usage
```java
@Service
public class FileService {
    private final AppProperties appProps;

    public FileService(AppProperties appProps) {
        this.appProps = appProps;
    }

    public void upload(MultipartFile file) {
        if (file.getSize() > appProps.getUpload().getMaxFileSize().toBytes()) {
            throw new FileTooLargeException();
        }
        // save to appProps.getUpload().getDirectory()
    }
}
```

### Records (Immutable — Boot 3.x)
```java
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(
    String host,
    int port,
    @DefaultValue("noreply@example.com") String from,
    @DefaultValue("true") boolean auth
) { }
```

---

## 26. EXTERNALIZED CONFIGURATION & CONFIG SERVER

### Config File Locations (Priority Order)
1. `./config/application.properties` (next to JAR)
2. `./application.properties` (next to JAR)
3. `classpath:/config/application.properties`
4. `classpath:/application.properties`

### Spring Cloud Config Server
```xml
<!-- Config Server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```
```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApp { }
```
```yaml
# Config Server application.yml
server:
  port: 8888
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
```

### Config Client
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```
```properties
# application.properties (client)
spring.config.import=configserver:http://localhost:8888
spring.application.name=my-service
spring.profiles.active=dev
```

### @RefreshScope — Hot Reload Config
```java
@RestController
@RefreshScope
public class MessageController {
    @Value("${app.message}")
    private String message;  // Refreshes on POST /actuator/refresh
}
```

---

## 27. EVENT HANDLING

### Application Events
```java
// Custom event
public class UserRegisteredEvent extends ApplicationEvent {
    private final User user;

    public UserRegisteredEvent(Object source, User user) {
        super(source);
        this.user = user;
    }

    public User getUser() { return user; }
}

// Publishing
@Service
public class UserService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public User register(UserRequest req) {
        User user = userRepo.save(new User(req));
        publisher.publishEvent(new UserRegisteredEvent(this, user));
        return user;
    }
}

// Listening
@Component
public class UserEventListener {

    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        log.info("User registered: {}", event.getUser().getEmail());
        // Send welcome email, etc.
    }

    @EventListener
    @Async   // Non-blocking
    public void handleAsync(UserRegisteredEvent event) {
        // Long-running task
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void afterCommit(UserRegisteredEvent event) {
        // Only runs after TX is committed
    }
}
```

### Built-in Spring Boot Events
| Event | When |
|-------|------|
| `ApplicationStartingEvent` | Very beginning, before anything |
| `ApplicationEnvironmentPreparedEvent` | Environment ready, context not created |
| `ApplicationContextInitializedEvent` | Context created, beans not loaded |
| `ApplicationPreparedEvent` | Beans loaded, not refreshed |
| `ApplicationStartedEvent` | Context refreshed, runners not called |
| `ApplicationReadyEvent` | App fully started and ready |
| `ApplicationFailedEvent` | Startup failed |

### Startup Runners
```java
@Component
@Order(1)
public class DataInitializer implements CommandLineRunner {
    @Override
    public void run(String... args) {
        // Runs after app starts — seed data, etc.
    }
}

@Component
@Order(2)
public class CacheWarmer implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        // ApplicationArguments provides parsed args
    }
}
```

---

## 28. SCHEDULING

### Enable Scheduling
```java
@SpringBootApplication
@EnableScheduling
public class MyApp { }
```

### Scheduled Tasks
```java
@Component
public class ScheduledTasks {

    // Fixed rate: every 5 seconds (regardless of previous execution)
    @Scheduled(fixedRate = 5000)
    public void fixedRateTask() {
        log.info("Fixed rate task: {}", Instant.now());
    }

    // Fixed delay: 5 seconds AFTER previous execution completes
    @Scheduled(fixedDelay = 5000)
    public void fixedDelayTask() {
        log.info("Fixed delay task");
    }

    // Initial delay + fixed rate
    @Scheduled(initialDelay = 10000, fixedRate = 60000)
    public void delayedTask() { ... }

    // Cron expression: every day at 2 AM
    @Scheduled(cron = "0 0 2 * * *")
    public void cronTask() {
        log.info("Daily cleanup job");
    }

    // Cron with timezone
    @Scheduled(cron = "0 0 9 * * MON-FRI", zone = "Asia/Kolkata")
    public void weekdayMorningTask() { ... }

    // Using properties
    @Scheduled(fixedRateString = "${app.task.rate:60000}")
    public void configuredTask() { ... }
}
```

### Cron Expression Format
```
┌───────── second (0-59)
│ ┌───────── minute (0-59)
│ │ ┌───────── hour (0-23)
│ │ │ ┌───────── day of month (1-31)
│ │ │ │ ┌───────── month (1-12 or JAN-DEC)
│ │ │ │ │ ┌───────── day of week (0-7 or MON-SUN, 0 & 7 = Sunday)
│ │ │ │ │ │
* * * * * *
```

### Custom Thread Pool for Scheduling
```java
@Configuration
public class SchedulingConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        registrar.setScheduler(Executors.newScheduledThreadPool(5));
    }
}
```

---

## 29. ASYNC PROCESSING

### Enable Async
```java
@SpringBootApplication
@EnableAsync
public class MyApp { }
```

### Async Methods
```java
@Service
public class EmailService {

    @Async
    public void sendEmail(String to, String subject, String body) {
        // Runs in a separate thread
        log.info("Sending email on thread: {}", Thread.currentThread().getName());
        // ... send email logic
    }

    @Async
    public CompletableFuture<String> sendEmailWithResult(String to) {
        // ... send email
        return CompletableFuture.completedFuture("Email sent to " + to);
    }
}

// Usage
@RestController
public class NotificationController {
    @Autowired private EmailService emailService;

    @PostMapping("/notify")
    public ResponseEntity<String> notify(@RequestBody NotificationRequest req) {
        emailService.sendEmail(req.getTo(), req.getSubject(), req.getBody());
        return ResponseEntity.ok("Notification queued");
    }
}
```

### Custom Async Executor
```java
@Configuration
public class AsyncConfig {

    @Bean("taskExecutor")
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

@Async("taskExecutor")
public void specificExecutor() { ... }
```

### Async Exception Handling
```java
@Configuration
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (throwable, method, params) -> {
            log.error("Async error in method {}: {}", method.getName(), throwable.getMessage());
        };
    }
}
```

---

## 30. HTTP CLIENTS — RestTemplate, WebClient & RestClient

### 30.1 RestTemplate (Synchronous — Classic)
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

@Service
public class PaymentService {
    private final RestTemplate restTemplate;

    public PaymentService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    // GET
    public PaymentDTO getPayment(Long id) {
        return restTemplate.getForObject(
                "https://api.example.com/payments/{id}", PaymentDTO.class, id);
    }

    // GET with headers
    public PaymentDTO getPaymentWithAuth(Long id, String token) {
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(token);
        HttpEntity<Void> entity = new HttpEntity<>(headers);

        ResponseEntity<PaymentDTO> response = restTemplate.exchange(
                "https://api.example.com/payments/{id}",
                HttpMethod.GET, entity, PaymentDTO.class, id);
        return response.getBody();
    }

    // POST
    public PaymentDTO createPayment(PaymentRequest request) {
        return restTemplate.postForObject(
                "https://api.example.com/payments", request, PaymentDTO.class);
    }

    // GET list
    public List<PaymentDTO> getAllPayments() {
        ResponseEntity<List<PaymentDTO>> response = restTemplate.exchange(
                "https://api.example.com/payments",
                HttpMethod.GET, null,
                new ParameterizedTypeReference<List<PaymentDTO>>() {});
        return response.getBody();
    }
}
```

### 30.2 WebClient (Non-Blocking — Reactive)
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
    public WebClient webClient(WebClient.Builder builder) {
        return builder
                .baseUrl("https://api.example.com")
                .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                .filter(ExchangeFilterFunctions.basicAuthentication("user", "pass"))
                .build();
    }
}

@Service
public class PaymentService {
    private final WebClient webClient;

    // GET
    public Mono<PaymentDTO> getPayment(Long id) {
        return webClient.get()
                .uri("/payments/{id}", id)
                .retrieve()
                .bodyToMono(PaymentDTO.class);
    }

    // GET — blocking (for non-reactive apps)
    public PaymentDTO getPaymentBlocking(Long id) {
        return webClient.get()
                .uri("/payments/{id}", id)
                .retrieve()
                .bodyToMono(PaymentDTO.class)
                .block();
    }

    // POST
    public Mono<PaymentDTO> createPayment(PaymentRequest request) {
        return webClient.post()
                .uri("/payments")
                .bodyValue(request)
                .retrieve()
                .onStatus(HttpStatusCode::is4xxClientError, response ->
                        Mono.error(new BadRequestException("Invalid payment")))
                .bodyToMono(PaymentDTO.class);
    }

    // GET list
    public Flux<PaymentDTO> getAllPayments() {
        return webClient.get()
                .uri("/payments")
                .retrieve()
                .bodyToFlux(PaymentDTO.class);
    }
}
```

### 30.3 RestClient (Spring Boot 3.2+ — New Synchronous Client)
```java
@Configuration
public class RestClientConfig {
    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        return builder
                .baseUrl("https://api.example.com")
                .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                .build();
    }
}

@Service
public class PaymentService {
    private final RestClient restClient;

    // GET
    public PaymentDTO getPayment(Long id) {
        return restClient.get()
                .uri("/payments/{id}", id)
                .retrieve()
                .body(PaymentDTO.class);
    }

    // POST with error handling
    public PaymentDTO createPayment(PaymentRequest request) {
        return restClient.post()
                .uri("/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .body(request)
                .retrieve()
                .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
                    throw new BadRequestException("Invalid payment: " + res.getStatusCode());
                })
                .body(PaymentDTO.class);
    }
}
```

### Which Client to Use?
| Client | When |
|--------|------|
| **RestClient** (3.2+) | New projects, synchronous calls, modern API |
| **WebClient** | Reactive apps, non-blocking I/O, streaming |
| **RestTemplate** | Legacy projects (still works, but less preferred) |

---

## 31. EMAIL SENDING

### Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

### Configuration
```properties
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=your@gmail.com
spring.mail.password=app-password          # Use App Password, not your real password
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```

### Simple Email
```java
@Service
@Slf4j
public class EmailService {

    @Autowired
    private JavaMailSender mailSender;

    // Plain text email
    public void sendSimpleEmail(String to, String subject, String body) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom("noreply@example.com");
        message.setTo(to);
        message.setSubject(subject);
        message.setText(body);
        mailSender.send(message);
        log.info("Email sent to {}", to);
    }

    // HTML email
    public void sendHtmlEmail(String to, String subject, String htmlBody) throws MessagingException {
        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
        helper.setFrom("noreply@example.com");
        helper.setTo(to);
        helper.setSubject(subject);
        helper.setText(htmlBody, true);   // true = HTML
        mailSender.send(message);
    }

    // Email with attachment
    public void sendEmailWithAttachment(String to, String subject, String body,
                                         String filePath) throws MessagingException {
        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true);
        helper.setFrom("noreply@example.com");
        helper.setTo(to);
        helper.setSubject(subject);
        helper.setText(body);
        helper.addAttachment("file.pdf", new File(filePath));
        mailSender.send(message);
    }
}
```

### Email with Thymeleaf Template
```java
@Service
public class EmailService {
    @Autowired private JavaMailSender mailSender;
    @Autowired private SpringTemplateEngine templateEngine;

    public void sendWelcomeEmail(String to, String name) throws MessagingException {
        Context context = new Context();
        context.setVariable("name", name);
        context.setVariable("appUrl", "https://example.com");
        String html = templateEngine.process("emails/welcome", context);

        MimeMessage message = mailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true);
        helper.setTo(to);
        helper.setSubject("Welcome to Our App!");
        helper.setText(html, true);
        mailSender.send(message);
    }
}
```

### Async Email (Non-Blocking)
```java
@Async
public void sendSimpleEmail(String to, String subject, String body) {
    // Same code — runs in background thread
}
```

---

## 32. FILE UPLOAD & DOWNLOAD

### Configuration
```properties
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=50MB
spring.servlet.multipart.enabled=true

# Custom upload directory
app.upload.dir=uploads
```

### File Upload Controller
```java
@RestController
@RequestMapping("/api/files")
@Slf4j
public class FileController {

    @Value("${app.upload.dir:uploads}")
    private String uploadDir;

    // Single file upload
    @PostMapping("/upload")
    public ResponseEntity<Map<String, String>> uploadFile(
            @RequestParam("file") MultipartFile file) throws IOException {

        if (file.isEmpty()) {
            throw new BadRequestException("File is empty");
        }

        // Validate file type
        String contentType = file.getContentType();
        if (!List.of("image/png", "image/jpeg", "application/pdf").contains(contentType)) {
            throw new BadRequestException("Unsupported file type: " + contentType);
        }

        // Generate unique filename
        String originalName = StringUtils.cleanPath(file.getOriginalFilename());
        String fileName = UUID.randomUUID() + "_" + originalName;

        // Save file
        Path uploadPath = Path.of(uploadDir);
        Files.createDirectories(uploadPath);
        Path filePath = uploadPath.resolve(fileName);
        Files.copy(file.getInputStream(), filePath, StandardCopyOption.REPLACE_EXISTING);

        log.info("Uploaded: {} ({} bytes)", fileName, file.getSize());

        return ResponseEntity.ok(Map.of(
                "fileName", fileName,
                "size", String.valueOf(file.getSize()),
                "type", contentType
        ));
    }

    // Multiple file upload
    @PostMapping("/upload-multiple")
    public ResponseEntity<List<String>> uploadMultiple(
            @RequestParam("files") List<MultipartFile> files) throws IOException {

        List<String> fileNames = new ArrayList<>();
        for (MultipartFile file : files) {
            String fileName = UUID.randomUUID() + "_" + file.getOriginalFilename();
            Path filePath = Path.of(uploadDir).resolve(fileName);
            Files.copy(file.getInputStream(), filePath, StandardCopyOption.REPLACE_EXISTING);
            fileNames.add(fileName);
        }
        return ResponseEntity.ok(fileNames);
    }

    // File download
    @GetMapping("/download/{fileName}")
    public ResponseEntity<Resource> downloadFile(@PathVariable String fileName) throws IOException {
        // Prevent path traversal attack
        Path filePath = Path.of(uploadDir).resolve(fileName).normalize();
        if (!filePath.startsWith(Path.of(uploadDir))) {
            throw new BadRequestException("Invalid file path");
        }

        Resource resource = new UrlResource(filePath.toUri());
        if (!resource.exists()) {
            throw new ResourceNotFoundException("File", 0L);
        }

        String contentType = Files.probeContentType(filePath);
        return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType(
                        contentType != null ? contentType : "application/octet-stream"))
                .header(HttpHeaders.CONTENT_DISPOSITION,
                        "attachment; filename=\"" + resource.getFilename() + "\"")
                .body(resource);
    }
}
```

### Upload to Database (Store as bytes)
```java
@Entity
public class FileEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String fileName;
    private String contentType;
    private long size;

    @Lob
    private byte[] data;
}

@PostMapping("/upload-db")
public ResponseEntity<Long> uploadToDb(@RequestParam("file") MultipartFile file)
        throws IOException {
    FileEntity entity = new FileEntity();
    entity.setFileName(file.getOriginalFilename());
    entity.setContentType(file.getContentType());
    entity.setSize(file.getSize());
    entity.setData(file.getBytes());
    FileEntity saved = fileRepo.save(entity);
    return ResponseEntity.ok(saved.getId());
}
```

---

## 33. SPRING BOOT WITH DOCKER

### Dockerfile
```dockerfile
# Multi-stage build
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
COPY . .
RUN ./mvnw clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080

# Run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Layered JAR (Optimized Docker Build)
```dockerfile
FROM eclipse-temurin:17-jre-alpine AS builder
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

### docker-compose.yml
```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/mydb
      - SPRING_DATASOURCE_USERNAME=root
      - SPRING_DATASOURCE_PASSWORD=secret
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: mydb
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mysql_data:
```

### Build Image via Maven Plugin (No Dockerfile)
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <image>
            <name>myapp:${project.version}</name>
        </image>
    </configuration>
</plugin>
```
```bash
./mvnw spring-boot:build-image
```

---

## 34. BUILDING FAT/UBER JARS & DEPLOYMENT

### Build & Run
```bash
# Build
./mvnw clean package          # Creates target/my-app-0.0.1-SNAPSHOT.jar
./mvnw clean package -DskipTests

# Run
java -jar target/my-app-0.0.1-SNAPSHOT.jar
java -jar app.jar --spring.profiles.active=prod --server.port=9090
java -jar -Xms256m -Xmx512m app.jar
```

### Build as WAR (for external server)
```xml
<!-- pom.xml -->
<packaging>war</packaging>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```
```java
public class MyApp extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(MyApp.class);
    }

    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

### Graceful Shutdown (Boot 2.3+)
```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

### Running as a System Service (Linux)
```bash
# Systemd service file: /etc/systemd/system/myapp.service
[Unit]
Description=My Spring Boot App
After=network.target

[Service]
Type=simple
User=appuser
ExecStart=/usr/bin/java -jar /opt/myapp/app.jar
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

## 35. SPRING BOOT CLI

### Install
```bash
# SDKMAN
sdk install springboot

# Homebrew
brew install spring-boot
```

### Commands
```bash
spring init --list                                    # List options
spring init -d web,data-jpa -b maven my-project      # Create project
spring run app.groovy                                  # Run Groovy script
spring jar app.jar app.groovy                          # Package
spring --version                                       # Version
```

---

## 36. CUSTOM AUTO-CONFIGURATION

### Creating a Custom Starter
```java
// 1. Properties class
@ConfigurationProperties(prefix = "mylib")
public class MyLibProperties {
    private boolean enabled = true;
    private String apiKey;
    // getters, setters
}

// 2. Service
public class MyLibService {
    private final MyLibProperties props;

    public MyLibService(MyLibProperties props) {
        this.props = props;
    }

    public String process(String input) {
        // Business logic using props.getApiKey()
        return "processed: " + input;
    }
}

// 3. Auto-configuration
@AutoConfiguration
@ConditionalOnClass(MyLibService.class)
@ConditionalOnProperty(prefix = "mylib", name = "enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(MyLibProperties.class)
public class MyLibAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyLibService myLibService(MyLibProperties props) {
        return new MyLibService(props);
    }
}
```

### Registration (Boot 3.x)
```
# src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.mylib.MyLibAutoConfiguration
```

### Registration (Boot 2.x — Legacy)
```
# src/main/resources/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.mylib.MyLibAutoConfiguration
```

---

## 37. SPRING BOOT WITH KAFKA

### Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### Configuration
```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=my-group
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=*

spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
```

### Producer
```java
@Service
public class OrderProducer {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void sendOrder(OrderEvent event) {
        kafkaTemplate.send("orders", event.getOrderId(), event)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    log.info("Sent: {} offset: {}", event,
                        result.getRecordMetadata().offset());
                } else {
                    log.error("Failed to send: {}", event, ex);
                }
            });
    }
}
```

### Consumer
```java
@Component
public class OrderConsumer {

    @KafkaListener(topics = "orders", groupId = "order-service")
    public void consume(OrderEvent event,
                        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                        @Header(KafkaHeaders.OFFSET) long offset) {
        log.info("Received: {} from partition: {} offset: {}", event, partition, offset);
        // Process the order
    }

    // Batch consumer
    @KafkaListener(topics = "orders", groupId = "batch-group",
                   containerFactory = "batchFactory")
    public void consumeBatch(List<OrderEvent> events) {
        log.info("Received batch of {} events", events.size());
        events.forEach(this::processOrder);
    }
}
```

---

## 38. SPRING BOOT WITH RABBITMQ

### Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### Configuration
```properties
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
```

```java
@Configuration
public class RabbitConfig {

    public static final String QUEUE = "order-queue";
    public static final String EXCHANGE = "order-exchange";
    public static final String ROUTING_KEY = "order.created";

    @Bean
    public Queue queue() {
        return new Queue(QUEUE, true);  // durable
    }

    @Bean
    public TopicExchange exchange() {
        return new TopicExchange(EXCHANGE);
    }

    @Bean
    public Binding binding(Queue queue, TopicExchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with(ROUTING_KEY);
    }

    @Bean
    public Jackson2JsonMessageConverter jsonConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory cf,
                                         Jackson2JsonMessageConverter converter) {
        RabbitTemplate template = new RabbitTemplate(cf);
        template.setMessageConverter(converter);
        return template;
    }
}
```

### Producer & Consumer
```java
// Producer
@Service
public class OrderPublisher {
    @Autowired private RabbitTemplate rabbitTemplate;

    public void publish(OrderEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitConfig.EXCHANGE, RabbitConfig.ROUTING_KEY, event);
    }
}

// Consumer
@Component
public class OrderListener {

    @RabbitListener(queues = RabbitConfig.QUEUE)
    public void handleOrder(OrderEvent event) {
        log.info("Received order: {}", event);
    }
}
```

---

## 39. SPRING BOOT WITH REDIS

### Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### Configuration
```properties
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.data.redis.password=
spring.data.redis.timeout=60000
```

### RedisTemplate Usage
```java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}

@Service
public class CacheService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public void save(String key, Object value, Duration ttl) {
        redisTemplate.opsForValue().set(key, value, ttl);
    }

    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    public void delete(String key) {
        redisTemplate.delete(key);
    }

    // Hash operations
    public void saveHash(String key, String field, Object value) {
        redisTemplate.opsForHash().put(key, field, value);
    }
}
```

### Redis as Cache Provider
```properties
spring.cache.type=redis
spring.cache.redis.time-to-live=600000
spring.cache.redis.cache-null-values=false
```

### Redis for Session Storage
```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```
```properties
spring.session.store-type=redis
spring.session.redis.flush-mode=on_save
server.servlet.session.timeout=30m
```

---

## 40. SPRING BOOT WITH GRAPHQL

### Setup (Boot 3.x)
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### Schema (`src/main/resources/graphql/schema.graphqls`)
```graphql
type Query {
    userById(id: ID!): User
    allUsers(page: Int, size: Int): [User]
}

type Mutation {
    createUser(input: CreateUserInput!): User
    updateUser(id: ID!, input: UpdateUserInput!): User
    deleteUser(id: ID!): Boolean
}

type User {
    id: ID!
    name: String!
    email: String!
    orders: [Order]
}

input CreateUserInput {
    name: String!
    email: String!
}

input UpdateUserInput {
    name: String
    email: String
}

type Order {
    id: ID!
    product: String!
    amount: Float!
}
```

### Controller
```java
@Controller
public class UserGraphQLController {

    @Autowired private UserService userService;

    @QueryMapping
    public User userById(@Argument Long id) {
        return userService.findById(id);
    }

    @QueryMapping
    public List<User> allUsers(@Argument int page, @Argument int size) {
        return userService.findAll(page, size);
    }

    @MutationMapping
    public User createUser(@Argument CreateUserInput input) {
        return userService.create(input);
    }

    @SchemaMapping(typeName = "User")
    public List<Order> orders(User user) {
        return orderService.findByUserId(user.getId());  // N+1 → use DataLoader
    }
}
```

### Properties
```properties
spring.graphql.graphiql.enabled=true    # GraphiQL UI at /graphiql
spring.graphql.path=/graphql
spring.graphql.schema.printer.enabled=true
```

---

## 41. RESILIENCE — RETRY & CIRCUIT BREAKER (Resilience4j)

### Why?
- In microservices, external calls **can fail** (network issues, service down, timeouts).
- **Resilience4j** provides: Retry, Circuit Breaker, Rate Limiter, Bulkhead, Time Limiter.
- Industry standard for building **fault-tolerant** Spring Boot applications.

### Setup
```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 41.1 Retry
```java
@Service
public class PaymentService {

    @Retry(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest req) {
        // This will be retried on failure
        return restClient.post()
                .uri("/payments")
                .body(req)
                .retrieve()
                .body(PaymentResponse.class);
    }

    // Fallback — must have same return type + extra Throwable param
    public PaymentResponse paymentFallback(PaymentRequest req, Throwable t) {
        log.error("Payment failed after retries: {}", t.getMessage());
        return PaymentResponse.builder().status("FAILED").build();
    }
}
```
```properties
resilience4j.retry.instances.paymentService.max-attempts=3
resilience4j.retry.instances.paymentService.wait-duration=2s
resilience4j.retry.instances.paymentService.exponential-backoff-multiplier=2
resilience4j.retry.instances.paymentService.retry-exceptions=java.io.IOException,java.net.SocketTimeoutException
```

### 41.2 Circuit Breaker
```java
@Service
public class InventoryService {

    @CircuitBreaker(name = "inventoryService", fallbackMethod = "inventoryFallback")
    public StockDTO checkStock(Long productId) {
        return restClient.get()
                .uri("/inventory/{id}", productId)
                .retrieve()
                .body(StockDTO.class);
    }

    public StockDTO inventoryFallback(Long productId, Throwable t) {
        log.warn("Circuit breaker open for inventory: {}", t.getMessage());
        return StockDTO.builder().productId(productId).available(false).build();
    }
}
```
```properties
# Circuit breaker config
resilience4j.circuitbreaker.instances.inventoryService.sliding-window-size=10
resilience4j.circuitbreaker.instances.inventoryService.failure-rate-threshold=50
resilience4j.circuitbreaker.instances.inventoryService.wait-duration-in-open-state=30s
resilience4j.circuitbreaker.instances.inventoryService.permitted-number-of-calls-in-half-open-state=5
resilience4j.circuitbreaker.instances.inventoryService.minimum-number-of-calls=5
```

### Circuit Breaker States
```
  CLOSED ──(failure rate > threshold)──► OPEN
    ▲                                      │
    │                                  (wait duration)
    │                                      ▼
    └───(success rate OK)──── HALF_OPEN ◄──┘
                                │
                    (failure rate > threshold)
                                │
                                ▼
                              OPEN
```

### 41.3 Rate Limiter
```java
@RateLimiter(name = "apiRateLimiter", fallbackMethod = "rateLimitFallback")
@GetMapping("/api/data")
public DataResponse getData() { ... }

public DataResponse rateLimitFallback(Throwable t) {
    throw new TooManyRequestsException("Rate limit exceeded. Try again later.");
}
```
```properties
resilience4j.ratelimiter.instances.apiRateLimiter.limit-for-period=100
resilience4j.ratelimiter.instances.apiRateLimiter.limit-refresh-period=1m
resilience4j.ratelimiter.instances.apiRateLimiter.timeout-duration=0s
```

### 41.4 Time Limiter
```java
@TimeLimiter(name = "slowService", fallbackMethod = "timeoutFallback")
public CompletableFuture<Response> callSlowService() {
    return CompletableFuture.supplyAsync(() -> restClient.get()...);
}
```
```properties
resilience4j.timelimiter.instances.slowService.timeout-duration=3s
```

### 41.5 Combining Patterns
```java
@CircuitBreaker(name = "backend", fallbackMethod = "fallback")
@Retry(name = "backend")
@RateLimiter(name = "backend")
public Response callBackend() { ... }
// Order: RateLimiter → Retry → CircuitBreaker (outermost to innermost)
```

---

## 42. AOP — ASPECT-ORIENTED PROGRAMMING

### What is AOP?
- **Cross-cutting concerns** (logging, timing, security, auditing) that span multiple classes.
- Instead of duplicating code in every method, define it **once** in an **Aspect**.
- Spring Boot uses **proxy-based AOP** (Spring AOP, not full AspectJ).

### Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### Key Concepts
| Term | Meaning |
|------|---------|
| **Aspect** | Class containing cross-cutting logic |
| **Advice** | The action (before, after, around) |
| **Pointcut** | Expression that matches methods |
| **JoinPoint** | The actual method being intercepted |

### 42.1 Logging Aspect (Most Common Use Case)
```java
@Aspect
@Component
@Slf4j
public class LoggingAspect {

    // Log all service methods
    @Before("execution(* com.example.myapp.service.*.*(..))")
    public void logBefore(JoinPoint jp) {
        log.info("→ {}.{}() args={}",
                jp.getTarget().getClass().getSimpleName(),
                jp.getSignature().getName(),
                Arrays.toString(jp.getArgs()));
    }

    @AfterReturning(pointcut = "execution(* com.example.myapp.service.*.*(..))",
                    returning = "result")
    public void logAfter(JoinPoint jp, Object result) {
        log.info("← {}.{}() returned={}",
                jp.getTarget().getClass().getSimpleName(),
                jp.getSignature().getName(),
                result);
    }

    @AfterThrowing(pointcut = "execution(* com.example.myapp.service.*.*(..))",
                   throwing = "ex")
    public void logException(JoinPoint jp, Throwable ex) {
        log.error("✖ {}.{}() threw {}",
                jp.getTarget().getClass().getSimpleName(),
                jp.getSignature().getName(),
                ex.getMessage());
    }
}
```

### 42.2 Performance Monitoring Aspect
```java
@Aspect
@Component
@Slf4j
public class PerformanceAspect {

    @Around("@annotation(com.example.myapp.annotation.TrackExecutionTime)")
    public Object trackTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            log.info("⏱ {}.{}() took {}ms",
                    pjp.getTarget().getClass().getSimpleName(),
                    pjp.getSignature().getName(), elapsed);
        }
    }
}

// Custom annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TrackExecutionTime { }

// Usage
@Service
public class ReportService {
    @TrackExecutionTime
    public Report generateReport() { ... }
}
```

### 42.3 Pointcut Expressions
```java
// All methods in service package
@Pointcut("execution(* com.example.myapp.service.*.*(..))")

// All public methods
@Pointcut("execution(public * *(..))")

// Methods with specific annotation
@Pointcut("@annotation(com.example.Auditable)")

// All methods in classes annotated with @Service
@Pointcut("@within(org.springframework.stereotype.Service)")

// Specific method signature
@Pointcut("execution(* com.example.myapp.service.UserService.findById(Long))")

// Combine pointcuts
@Pointcut("serviceLayer() && !execution(* *.get*(..))")
```

### Advice Types
| Advice | When | Use Case |
|--------|------|----------|
| `@Before` | Before method | Logging, auth checks |
| `@After` | After method (always) | Cleanup |
| `@AfterReturning` | After successful return | Logging result |
| `@AfterThrowing` | After exception | Error logging |
| `@Around` | Wraps entire method | Timing, TX, caching |

---

## 43. SPRING BOOT OBSERVABILITY — MICROMETER & TRACING

### Prometheus + Grafana Setup
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```
```properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.prometheus.metrics.export.enabled=true
```
- Prometheus scrapes `/actuator/prometheus`
- Grafana visualizes metrics

### Distributed Tracing (Boot 3.x)
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```
```properties
management.tracing.sampling.probability=1.0    # 100% sampling (dev)
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
```
- Each request gets a **traceId** and **spanId** automatically added to MDC → visible in logs.

### Log Pattern with Trace Info
```properties
logging.pattern.console=%d{HH:mm:ss} %-5level [%X{traceId},%X{spanId}] %logger{36} - %msg%n
```

---

## 44. GRAALVM NATIVE IMAGES

### What?
- Compile Spring Boot app to a **native executable** (no JVM needed at runtime).
- **Near-instant startup** (~50ms) and **lower memory footprint**.
- Requires **Spring Boot 3.x** + **GraalVM**.

### Setup
```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
</plugin>
```

### Build
```bash
# Requires GraalVM installed
./mvnw -Pnative native:compile

# Or build native Docker image (no local GraalVM needed)
./mvnw -Pnative spring-boot:build-image
```

### Limitations
- No runtime reflection (must declare hints)
- No dynamic proxies without config
- Some libraries may not be compatible
- Longer build time

### Runtime Hints
```java
@RegisterReflectionForBinding({User.class, OrderDTO.class})
@ImportRuntimeHints(MyRuntimeHints.class)
@SpringBootApplication
public class MyApp { }

public class MyRuntimeHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.reflection().registerType(User.class,
            MemberCategory.PUBLIC_FIELDS, MemberCategory.INVOKE_PUBLIC_METHODS);
        hints.resources().registerPattern("data/*.json");
    }
}
```

---

## 45. BEST PRACTICES & PRODUCTION CHECKLIST

### Project Structure
```
com.example.myapp/
├── MyAppApplication.java
├── config/          ← @Configuration classes
├── controller/      ← @RestController
├── service/         ← @Service (business logic)
├── repository/      ← @Repository (data access)
├── model/
│   ├── entity/      ← JPA entities
│   └── dto/         ← Data Transfer Objects
├── exception/       ← Custom exceptions + @ControllerAdvice
├── security/        ← Security config, filters
├── mapper/          ← Entity ↔ DTO mappers
└── util/            ← Utility classes
```

### Best Practices
1. **Use constructor injection** — avoid field injection
2. **Use DTOs** — never expose entities directly in APIs
3. **Validate at controller layer** — `@Valid` / `@Validated`
4. **Use @Transactional on service layer** — not on repositories
5. **Externalize config** — never hardcode secrets
6. **Use profiles** — separate dev/test/prod config
7. **Enable graceful shutdown** — `server.shutdown=graceful`
8. **Use Flyway/Liquibase** — never `ddl-auto=update` in prod
9. **Health checks** — `/actuator/health` for container orchestration
10. **Structured logging** — JSON logs in production
11. **Use `@ConfigurationProperties`** — over scattered `@Value`
12. **Handle exceptions globally** — `@RestControllerAdvice`
13. **Document APIs** — OpenAPI / Swagger
14. **Write tests** — unit + integration + contract
15. **Use connection pooling** — tune HikariCP for your load

### Production Checklist
```properties
# Security
server.ssl.enabled=true
management.server.port=8081        # Separate actuator port
management.endpoints.web.exposure.include=health,info,metrics,prometheus

# Performance
server.tomcat.max-threads=200
spring.datasource.hikari.maximum-pool-size=20
server.compression.enabled=true

# Reliability
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
spring.jpa.open-in-view=false      # Avoid lazy loading in views

# Logging
logging.level.root=WARN
logging.level.com.example=INFO

# Database
spring.jpa.hibernate.ddl-auto=validate
spring.flyway.enabled=true
```

---

## 46. IMPORTANT ANNOTATIONS REFERENCE

### Core / Boot
| Annotation | Purpose |
|------------|---------|
| `@SpringBootApplication` | Main class — combines `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| `@EnableAutoConfiguration` | Enables auto-configuration |
| `@ConfigurationProperties` | Bind properties to POJO |
| `@EnableConfigurationProperties` | Register `@ConfigurationProperties` class |
| `@ConditionalOnProperty` | Conditional bean creation based on property |
| `@ConditionalOnClass` | Conditional bean creation based on class presence |
| `@ConditionalOnMissingBean` | Create bean only if not already defined |
| `@Profile` | Activate bean for specific profile |

### Web
| Annotation | Purpose |
|------------|---------|
| `@RestController` | `@Controller` + `@ResponseBody` |
| `@RequestMapping` | Map URL to handler |
| `@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` / `@PatchMapping` | HTTP method mapping |
| `@PathVariable` | Extract from URL path |
| `@RequestParam` | Extract from query string |
| `@RequestBody` | Deserialize request body |
| `@ResponseStatus` | Set HTTP status code |
| `@CrossOrigin` | Enable CORS |

### Data
| Annotation | Purpose |
|------------|---------|
| `@Entity` | JPA entity |
| `@Table` | Specify table name |
| `@Id` / `@GeneratedValue` | Primary key |
| `@Column` | Column mapping |
| `@OneToMany` / `@ManyToOne` / `@ManyToMany` | Relationships |
| `@Query` | Custom JPQL/SQL query |
| `@Modifying` | For UPDATE/DELETE queries |
| `@Transactional` | Wrap in transaction |
| `@CreatedDate` / `@LastModifiedDate` | Auditing timestamps |

### Validation
| Annotation | Purpose |
|------------|---------|
| `@Valid` / `@Validated` | Trigger validation |
| `@NotNull` / `@NotBlank` / `@NotEmpty` | Null checks |
| `@Size(min, max)` | String/collection size |
| `@Min` / `@Max` | Numeric bounds |
| `@Email` | Email format |
| `@Pattern` | Regex match |

### Testing
| Annotation | Purpose |
|------------|---------|
| `@SpringBootTest` | Full context integration test |
| `@WebMvcTest` | Web layer only |
| `@DataJpaTest` | JPA layer only |
| `@MockBean` | Mock a bean in context |
| `@TestConfiguration` | Test-specific config |

### Caching
| Annotation | Purpose |
|------------|---------|
| `@EnableCaching` | Enable cache support |
| `@Cacheable` | Cache method result |
| `@CachePut` | Update cache |
| `@CacheEvict` | Remove from cache |

### Scheduling & Async
| Annotation | Purpose |
|------------|---------|
| `@EnableScheduling` | Enable scheduling |
| `@Scheduled` | Mark scheduled method |
| `@EnableAsync` | Enable async support |
| `@Async` | Run method asynchronously |

### Lombok
| Annotation | Purpose |
|------------|---------|
| `@Data` | Getter + Setter + ToString + Equals + RequiredArgsConstructor |
| `@Builder` | Builder pattern |
| `@Slf4j` | Auto-generate SLF4J logger |
| `@RequiredArgsConstructor` | Constructor for final fields (DI) |
| `@NoArgsConstructor` / `@AllArgsConstructor` | Constructors |

### Resilience4j
| Annotation | Purpose |
|------------|---------|
| `@Retry` | Retry on failure |
| `@CircuitBreaker` | Circuit breaker pattern |
| `@RateLimiter` | Rate limiting |
| `@TimeLimiter` | Timeout handling |

### AOP
| Annotation | Purpose |
|------------|---------|
| `@Aspect` | Declare AOP aspect class |
| `@Before` / `@After` / `@Around` | Advice types |
| `@Pointcut` | Define reusable join point expression |

### OpenAPI / Swagger
| Annotation | Purpose |
|------------|---------|
| `@Tag` | Group endpoints |
| `@Operation` | Describe endpoint |
| `@ApiResponse` | Document response |
| `@Schema` | Describe DTO field |
| `@Parameter` | Describe parameter |

---

## 47. COMMON CONFIGURATION PROPERTIES REFERENCE

```properties
# ===================== SERVER =====================
server.port=8080
server.servlet.context-path=/api
server.error.include-message=always
server.error.include-binding-errors=always
server.shutdown=graceful

# ===================== DATASOURCE =====================
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=secret
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5

# ===================== JPA =====================
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.open-in-view=false
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.default_batch_fetch_size=20

# ===================== JACKSON (JSON) =====================
spring.jackson.serialization.write-dates-as-timestamps=false
spring.jackson.deserialization.fail-on-unknown-properties=false
spring.jackson.default-property-inclusion=non_null
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=UTC

# ===================== LOGGING =====================
logging.level.root=INFO
logging.level.com.example=DEBUG
logging.file.name=logs/app.log

# ===================== ACTUATOR =====================
management.endpoints.web.exposure.include=health,info,metrics
management.endpoint.health.show-details=when_authorized

# ===================== SECURITY =====================
spring.security.user.name=admin
spring.security.user.password=admin123

# ===================== MAIL =====================
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=your@email.com
spring.mail.password=app-password
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true

# ===================== FILE UPLOAD =====================
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=50MB

# ===================== CACHE =====================
spring.cache.type=caffeine
spring.cache.caffeine.spec=maximumSize=500,expireAfterWrite=10m

# ===================== KAFKA =====================
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=my-group

# ===================== REDIS =====================
spring.data.redis.host=localhost
spring.data.redis.port=6379

# ===================== FLYWAY =====================
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
```

---

> **End of Spring Boot Notes — 47 Sections — Beginner to Advanced**
