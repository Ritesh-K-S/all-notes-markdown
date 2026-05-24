# Maven Build Tool — Complete Notes

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Project Structure (Convention)](#2-project-structure-convention)
3. [POM.XML (Project Object Model)](#3-pomxml-project-object-model)
4. [Dependencies](#4-dependencies)
5. [Maven Build Lifecycle](#5-maven-build-lifecycle)
6. [Maven Commands](#6-maven-commands)
7. [Properties](#7-properties)
8. [Plugins](#8-plugins)
9. [Profiles](#9-profiles)
10. [Multi-Module Projects](#10-multi-module-projects)
11. [Repositories](#11-repositories)
12. [Maven Wrapper](#12-maven-wrapper)
13. [Dependency Resolution](#13-dependency-resolution)
14. [Resource Filtering](#14-resource-filtering)
15. [Archetypes](#15-archetypes)
16. [Common POM Examples](#16-common-pom-examples)
17. [Troubleshooting](#17-troubleshooting)
18. [Best Practices](#18-best-practices)
19. [Maven vs Gradle vs Ant](#19-maven-vs-gradle-vs-ant)
20. [Quick Reference Cheat Sheet](#20-quick-reference-cheat-sheet)

---

## 1. INTRODUCTION

### What is Maven?
- **Build automation** and **dependency management** tool for Java projects.
- Created by **Apache Software Foundation** (2004).
- Uses **XML** configuration (`pom.xml` — Project Object Model).
- Follows **Convention over Configuration** — standard project structure.
- Downloads dependencies from **Maven Central Repository** automatically.
- Manages the full build lifecycle: compile → test → package → deploy.

### Why Maven?
| Without Maven | With Maven |
|---------------|------------|
| Manually download JARs | Auto-download dependencies |
| No standard structure | Standard project layout |
| Custom build scripts | Standardized build lifecycle |
| Dependency conflicts | Dependency resolution |
| No version management | Version management built-in |

### Install Maven
```bash
# Check if installed
mvn --version
mvn -v

# Linux (apt)
sudo apt install maven

# macOS (Homebrew)
brew install maven

# Windows — download from https://maven.apache.org/download.cgi
# 1. Extract zip
# 2. Set MAVEN_HOME = C:\apache-maven-x.x.x
# 3. Add %MAVEN_HOME%\bin to PATH

# Verify
mvn -v
# Apache Maven 3.9.x
# Maven home: ...
# Java version: ...
```

### Prerequisites
- **JDK** installed (not just JRE)
- `JAVA_HOME` environment variable set
- Maven binary in `PATH`

---

## 2. PROJECT STRUCTURE (Convention)

```
my-project/
├── pom.xml                          ← Project config (heart of Maven)
├── src/
│   ├── main/
│   │   ├── java/                    ← Application source code
│   │   │   └── com/myapp/
│   │   │       └── App.java
│   │   ├── resources/               ← Config files, properties
│   │   │   └── application.properties
│   │   └── webapp/                  ← Web app resources (WAR projects)
│   │       ├── WEB-INF/
│   │       │   └── web.xml
│   │       └── index.html
│   └── test/
│       ├── java/                    ← Test source code
│       │   └── com/myapp/
│       │       └── AppTest.java
│       └── resources/               ← Test config files
│           └── test-config.properties
├── target/                          ← Build output (generated — .gitignore this)
│   ├── classes/                     ← Compiled .class files
│   ├── test-classes/                ← Compiled test classes
│   ├── surefire-reports/            ← Test reports
│   └── my-project-1.0.0.jar        ← Packaged artifact
└── .mvn/                           ← Maven wrapper files (optional)
    └── wrapper/
        └── maven-wrapper.properties
```

### Key Directories
| Directory | Purpose |
|-----------|---------|
| `src/main/java` | Application Java source |
| `src/main/resources` | Config, properties, XML files |
| `src/main/webapp` | Web resources (JSP, HTML, CSS) |
| `src/test/java` | Test Java source |
| `src/test/resources` | Test config files |
| `target/` | Build output (auto-generated) |

---

## 3. POM.XML (Project Object Model)

### Minimal pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- Project Coordinates (GAV) -->
    <groupId>com.mycompany</groupId>       <!-- organization/group -->
    <artifactId>my-project</artifactId>     <!-- project name -->
    <version>1.0.0</version>                <!-- version -->
    <packaging>jar</packaging>              <!-- jar | war | pom | ear -->

    <name>My Project</name>
    <description>A sample project</description>
    <url>https://example.com</url>

</project>
```

### Full pom.xml Structure
```xml
<project>
    <modelVersion>4.0.0</modelVersion>

    <!-- ========== PROJECT COORDINATES ========== -->
    <groupId>com.mycompany</groupId>
    <artifactId>my-project</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>My Project</name>
    <description>Project description</description>
    <url>https://example.com</url>

    <!-- ========== PARENT POM ========== -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/> <!-- lookup from repository -->
    </parent>

    <!-- ========== PROPERTIES ========== -->
    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <junit.version>5.10.0</junit.version>
        <spring.version>6.1.0</spring.version>
    </properties>

    <!-- ========== DEPENDENCIES ========== -->
    <dependencies>
        <!-- ... -->
    </dependencies>

    <!-- ========== DEPENDENCY MANAGEMENT ========== -->
    <dependencyManagement>
        <!-- ... -->
    </dependencyManagement>

    <!-- ========== BUILD ========== -->
    <build>
        <plugins>
            <!-- ... -->
        </plugins>
    </build>

    <!-- ========== PROFILES ========== -->
    <profiles>
        <!-- ... -->
    </profiles>

    <!-- ========== REPOSITORIES ========== -->
    <repositories>
        <!-- ... -->
    </repositories>

    <!-- ========== MODULES (multi-module) ========== -->
    <modules>
        <!-- ... -->
    </modules>

</project>
```

### Project Coordinates (GAV)
| Element | Description | Example |
|---------|-------------|---------|
| `groupId` | Organization identifier (reverse domain) | `com.google.guava` |
| `artifactId` | Project/module name | `guava` |
| `version` | Version number | `31.1-jre` |
| `packaging` | Output type | `jar`, `war`, `pom`, `ear` |
| `classifier` | Variant (optional) | `sources`, `javadoc`, `tests` |

### Version Conventions
```
1.0.0           — release version
1.0.0-SNAPSHOT  — development version (changes frequently)
1.0.0-RELEASE   — explicit release
1.0.0-RC1       — release candidate
1.0.0-beta      — beta
1.0.0-alpha     — alpha
```

**SNAPSHOT** — Maven re-downloads on every build to get latest changes.  
**Release** — Downloaded once, cached forever.

---

## 4. DEPENDENCIES

### Adding Dependencies
```xml
<dependencies>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.0</version>
        <scope>test</scope>
    </dependency>

    <!-- Spring Web -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
        <version>6.1.0</version>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.30</version>
        <scope>provided</scope>
    </dependency>

    <!-- MySQL Connector -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>8.2.0</version>
        <scope>runtime</scope>
    </dependency>

</dependencies>
```

### Dependency Scopes
| Scope | Compile | Test | Runtime | Packaged | Use Case |
|-------|---------|------|---------|----------|----------|
| `compile` (default) | ✅ | ✅ | ✅ | ✅ | Normal dependencies |
| `provided` | ✅ | ✅ | ❌ | ❌ | Server provides (Servlet API, Lombok) |
| `runtime` | ❌ | ✅ | ✅ | ✅ | JDBC drivers, SLF4J impl |
| `test` | ❌ | ✅ | ❌ | ❌ | JUnit, Mockito |
| `system` | ✅ | ✅ | ❌ | ❌ | Local JAR (avoid!) |
| `import` | — | — | — | — | Import BOM in dependencyManagement |

### Dependency with Exclusions
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>6.1.0</version>
    <exclusions>
        <exclusion>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### Optional Dependencies
```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.16.0</version>
    <optional>true</optional>    <!-- not inherited by dependent projects -->
</dependency>
```

### System Scope (Local JAR — avoid if possible)
```xml
<dependency>
    <groupId>com.custom</groupId>
    <artifactId>my-lib</artifactId>
    <version>1.0</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/lib/my-lib.jar</systemPath>
</dependency>
```

### Dependency Management (Version Control for Multi-Module)
```xml
<!-- Parent POM — define versions centrally -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>6.1.0</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.16.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Child POM — no version needed -->
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <!-- version inherited from parent -->
    </dependency>
</dependencies>
```

### BOM (Bill of Materials)
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Now use Spring Boot dependencies without specifying versions -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### Finding Dependencies
- Search on **Maven Central**: https://search.maven.org/
- Or **MVN Repository**: https://mvnrepository.com/
- Copy the `<dependency>` XML snippet into your `pom.xml`

---

## 5. MAVEN BUILD LIFECYCLE

### Three Built-in Lifecycles
| Lifecycle | Purpose |
|-----------|---------|
| **default** | Build & deploy (most used) |
| **clean** | Remove build output |
| **site** | Generate project documentation |

### Default Lifecycle Phases (in order)
| Phase | Description |
|-------|-------------|
| `validate` | Validate project is correct and all info available |
| `compile` | Compile source code (`src/main/java` → `target/classes`) |
| `test` | Run unit tests (using Surefire plugin) |
| `package` | Package compiled code (JAR/WAR) |
| `verify` | Run integration tests and checks |
| `install` | Install to local repository (`~/.m2/repository`) |
| `deploy` | Deploy to remote repository |

**Important:** Running a phase executes ALL prior phases.
```bash
mvn package    # runs: validate → compile → test → package
mvn install    # runs: validate → compile → test → package → verify → install
```

### Clean Lifecycle
| Phase | Description |
|-------|-------------|
| `pre-clean` | Before cleaning |
| `clean` | Delete `target/` directory |
| `post-clean` | After cleaning |

### Site Lifecycle
| Phase | Description |
|-------|-------------|
| `pre-site` | Before site generation |
| `site` | Generate project site docs |
| `post-site` | After site generation |
| `site-deploy` | Deploy site to server |

---

## 6. MAVEN COMMANDS

### Basic Commands
```bash
# Create project from archetype
mvn archetype:generate

# Create a simple Java project
mvn archetype:generate \
  -DgroupId=com.myapp \
  -DartifactId=my-project \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false

# Create a web project
mvn archetype:generate \
  -DarchetypeArtifactId=maven-archetype-webapp
```

### Build Commands
```bash
mvn compile              # compile source code
mvn test                 # run tests
mvn package              # create JAR/WAR
mvn install              # install to local repo (~/.m2)
mvn deploy               # deploy to remote repo
mvn clean                # delete target/ directory
mvn clean install        # clean + full build + install
mvn clean package        # clean + build + package

mvn verify               # run integration tests
mvn validate             # validate POM
mvn site                 # generate site docs
```

### Skip Tests
```bash
mvn package -DskipTests              # compile tests but don't run
mvn package -Dmaven.test.skip=true   # skip compiling AND running tests
```

### Run Specific Test
```bash
mvn test -Dtest=MyClassTest                    # specific class
mvn test -Dtest=MyClassTest#testMethod         # specific method
mvn test -Dtest="MyClassTest, OtherTest"       # multiple classes
mvn test -Dtest="com.myapp.**"                 # by pattern
```

### Dependency Commands
```bash
mvn dependency:tree                      # show dependency tree
mvn dependency:tree -Dincludes=groupId   # filter
mvn dependency:list                      # list all dependencies
mvn dependency:analyze                   # find unused/undeclared
mvn dependency:resolve                   # resolve all
mvn dependency:purge-local-repository    # force re-download
mvn dependency:copy-dependencies         # copy to target/dependency/
mvn dependency:sources                   # download source JARs
mvn dependency:resolve-plugins           # resolve plugin dependencies
```

### Other Useful Commands
```bash
mvn help:effective-pom           # show full resolved POM (with parent)
mvn help:effective-settings      # show resolved settings
mvn help:describe -Dplugin=compiler  # plugin info
mvn help:active-profiles         # show active profiles

mvn versions:display-dependency-updates  # check for newer versions
mvn versions:display-plugin-updates      # check plugin updates
mvn versions:set -DnewVersion=2.0.0     # set project version

mvn exec:java -Dexec.mainClass=com.myapp.App   # run main class
mvn spring-boot:run                              # run Spring Boot app

mvn -o install                   # offline mode (no downloads)
mvn -U install                   # force update snapshots
mvn -X install                   # debug output
mvn -q install                   # quiet mode (errors only)
mvn -pl module1 install          # build specific module
mvn -pl module1 -am install      # build module + dependencies
mvn -T 4 install                 # parallel build (4 threads)
mvn -T 1C install                # parallel (1 thread per core)
mvn -f other/pom.xml install     # use different POM
mvn -P production install        # activate profile
```

---

## 7. PROPERTIES

### Built-in Properties
```xml
<properties>
    <!-- Java version -->
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>

    <!-- Encoding -->
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

    <!-- Custom properties (use as ${property.name}) -->
    <spring.version>6.1.0</spring.version>
    <junit.version>5.10.0</junit.version>
</properties>

<!-- Using custom properties -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>${spring.version}</version>
</dependency>
```

### Predefined Properties
| Property | Value |
|----------|-------|
| `${project.groupId}` | Project groupId |
| `${project.artifactId}` | Project artifactId |
| `${project.version}` | Project version |
| `${project.name}` | Project name |
| `${project.basedir}` | Project root directory |
| `${project.build.directory}` | `target/` |
| `${project.build.sourceDirectory}` | `src/main/java` |
| `${project.build.outputDirectory}` | `target/classes` |
| `${project.build.finalName}` | `artifactId-version` |
| `${settings.localRepository}` | Local repo path (~/.m2/repository) |
| `${java.home}` | JDK home |
| `${java.version}` | Java version |
| `${os.name}` | Operating system |
| `${env.JAVA_HOME}` | Environment variable |

### Property in Command Line
```bash
mvn install -Dspring.version=6.0.0    # override property
```

---

## 8. PLUGINS

### What are Plugins?
- Maven is **plugin-based** — all work is done by plugins.
- Each lifecycle phase maps to a plugin **goal**.
- Core plugins come bundled; others are downloaded as needed.

### Phase → Plugin Mapping (Default Lifecycle for JAR)
| Phase | Plugin : Goal |
|-------|--------------|
| `compile` | `maven-compiler-plugin:compile` |
| `test` | `maven-surefire-plugin:test` |
| `package` | `maven-jar-plugin:jar` |
| `install` | `maven-install-plugin:install` |
| `deploy` | `maven-deploy-plugin:deploy` |
| `clean` | `maven-clean-plugin:clean` |
| `site` | `maven-site-plugin:site` |

### Configuring Plugins
```xml
<build>
    <plugins>

        <!-- Compiler Plugin — set Java version -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.12.0</version>
            <configuration>
                <source>21</source>
                <target>21</target>
                <release>21</release>    <!-- preferred over source/target -->
                <encoding>UTF-8</encoding>
                <compilerArgs>
                    <arg>-Xlint:all</arg>
                    <arg>--enable-preview</arg>
                </compilerArgs>
            </configuration>
        </plugin>

        <!-- Surefire Plugin — unit tests (runs *Test.java) -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.0</version>
            <configuration>
                <includes>
                    <include>**/*Test.java</include>
                    <include>**/*Tests.java</include>
                </includes>
                <excludes>
                    <exclude>**/*IntegrationTest.java</exclude>
                </excludes>
                <argLine>-Xmx1024m</argLine>
            </configuration>
        </plugin>

        <!-- Failsafe Plugin — integration tests (runs *IT.java) -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-failsafe-plugin</artifactId>
            <version>3.2.0</version>
            <executions>
                <execution>
                    <goals>
                        <goal>integration-test</goal>
                        <goal>verify</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>

        <!-- JAR Plugin — customize JAR -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>3.3.0</version>
            <configuration>
                <archive>
                    <manifest>
                        <mainClass>com.myapp.App</mainClass>
                        <addClasspath>true</addClasspath>
                    </manifest>
                </archive>
            </configuration>
        </plugin>

        <!-- WAR Plugin — for web apps -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-war-plugin</artifactId>
            <version>3.4.0</version>
            <configuration>
                <failOnMissingWebXml>false</failOnMissingWebXml>
            </configuration>
        </plugin>

    </plugins>
</build>
```

### Commonly Used Plugins

#### Shade Plugin (Uber/Fat JAR — all dependencies bundled)
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.5.0</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
            <configuration>
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>com.myapp.App</mainClass>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

#### Assembly Plugin (Custom distribution packages)
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>3.6.0</version>
    <configuration>
        <archive>
            <manifest>
                <mainClass>com.myapp.App</mainClass>
            </manifest>
        </archive>
        <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
    </configuration>
    <executions>
        <execution>
            <id>assemble</id>
            <phase>package</phase>
            <goals><goal>single</goal></goals>
        </execution>
    </executions>
</plugin>
```

#### Exec Plugin (Run Java program)
```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>exec-maven-plugin</artifactId>
    <version>3.1.0</version>
    <configuration>
        <mainClass>com.myapp.App</mainClass>
    </configuration>
</plugin>
```
```bash
mvn exec:java
```

#### Source Plugin (Attach source JAR)
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-source-plugin</artifactId>
    <version>3.3.0</version>
    <executions>
        <execution>
            <goals><goal>jar-no-fork</goal></goals>
        </execution>
    </executions>
</plugin>
```

#### Javadoc Plugin
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-javadoc-plugin</artifactId>
    <version>3.6.0</version>
    <executions>
        <execution>
            <goals><goal>jar</goal></goals>
        </execution>
    </executions>
</plugin>
```

#### Resources Plugin (Filter & copy resources)
```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>    <!-- replace ${...} in files -->
        </resource>
    </resources>
</build>
```
```properties
# src/main/resources/application.properties
app.name=${project.name}
app.version=${project.version}
# After filtering: app.name=My Project, app.version=1.0.0
```

#### Spring Boot Plugin
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <version>3.2.0</version>
    <configuration>
        <mainClass>com.myapp.Application</mainClass>
    </configuration>
</plugin>
```
```bash
mvn spring-boot:run        # run application
mvn spring-boot:repackage  # create executable JAR
```

### Plugin Management (Version Centralization)
```xml
<!-- In parent POM -->
<build>
    <pluginManagement>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.12.0</version>
                <configuration>
                    <release>21</release>
                </configuration>
            </plugin>
        </plugins>
    </pluginManagement>
</build>

<!-- Child POM — just reference, no version/config needed -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

---

## 9. PROFILES

### What are Profiles?
- Profiles let you **customize builds** for different environments.
- Activate by CLI flag, environment, JDK version, OS, or file existence.

### Defining Profiles
```xml
<profiles>

    <!-- Development profile -->
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <env>development</env>
            <db.url>jdbc:mysql://localhost:3306/devdb</db.url>
        </properties>
        <dependencies>
            <dependency>
                <groupId>com.h2database</groupId>
                <artifactId>h2</artifactId>
                <version>2.2.224</version>
            </dependency>
        </dependencies>
    </profile>

    <!-- Production profile -->
    <profile>
        <id>prod</id>
        <properties>
            <env>production</env>
            <db.url>jdbc:mysql://prod-server:3306/proddb</db.url>
        </properties>
        <dependencies>
            <dependency>
                <groupId>com.mysql</groupId>
                <artifactId>mysql-connector-j</artifactId>
                <version>8.2.0</version>
            </dependency>
        </dependencies>
    </profile>

    <!-- Test profile -->
    <profile>
        <id>integration-tests</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-failsafe-plugin</artifactId>
                    <executions>
                        <execution>
                            <goals>
                                <goal>integration-test</goal>
                                <goal>verify</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>

</profiles>
```

### Activating Profiles
```bash
# By CLI flag
mvn install -P prod
mvn install -P dev,integration-tests    # multiple profiles
mvn install -P !dev                     # deactivate profile

# Check active profiles
mvn help:active-profiles
```

### Automatic Activation
```xml
<profile>
    <id>windows</id>
    <activation>
        <!-- By OS -->
        <os>
            <family>windows</family>
        </os>
    </activation>
</profile>

<profile>
    <id>jdk21</id>
    <activation>
        <!-- By JDK version -->
        <jdk>21</jdk>
    </activation>
</profile>

<profile>
    <id>ci</id>
    <activation>
        <!-- By system property -->
        <property>
            <name>env.CI</name>
        </property>
    </activation>
</profile>

<profile>
    <id>docker</id>
    <activation>
        <!-- By file existence -->
        <file>
            <exists>Dockerfile</exists>
        </file>
    </activation>
</profile>
```

---

## 10. MULTI-MODULE PROJECTS

### Parent POM
```xml
<!-- parent/pom.xml -->
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.myapp</groupId>
    <artifactId>my-app-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>       <!-- MUST be pom for parent -->

    <modules>
        <module>core</module>
        <module>api</module>
        <module>web</module>
    </modules>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.release>21</maven.compiler.release>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- Central version management -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.2.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

</project>
```

### Child POM
```xml
<!-- core/pom.xml -->
<project>
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.myapp</groupId>
        <artifactId>my-app-parent</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>core</artifactId>    <!-- inherits groupId, version -->

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <!-- version from parent's dependencyManagement -->
        </dependency>
    </dependencies>
</project>
```

```xml
<!-- api/pom.xml -->
<project>
    <parent>
        <groupId>com.myapp</groupId>
        <artifactId>my-app-parent</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>api</artifactId>

    <dependencies>
        <!-- Depend on sibling module -->
        <dependency>
            <groupId>com.myapp</groupId>
            <artifactId>core</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>
</project>
```

### Directory Structure
```
my-app-parent/
├── pom.xml              ← parent POM (packaging=pom)
├── core/
│   ├── pom.xml
│   └── src/main/java/...
├── api/
│   ├── pom.xml
│   └── src/main/java/...
└── web/
    ├── pom.xml
    └── src/main/java/...
```

### Build Commands
```bash
mvn clean install                # build all modules (from parent)
mvn clean install -pl core       # build only core module
mvn clean install -pl api -am    # build api + its dependencies (core)
mvn clean install -pl core -amd  # build core + modules that depend on it
mvn clean install -rf api        # resume build from api module
```

---

## 11. REPOSITORIES

### Repository Types
| Type | Location | Purpose |
|------|----------|---------|
| **Local** | `~/.m2/repository` | Cached downloads |
| **Central** | `repo.maven.apache.org` | Default public repo |
| **Remote** | Custom URL | Organization-specific repos |

### Configure Custom Repositories
```xml
<!-- In pom.xml -->
<repositories>
    <repository>
        <id>jitpack</id>
        <url>https://jitpack.io</url>
    </repository>
    <repository>
        <id>spring-milestones</id>
        <url>https://repo.spring.io/milestone</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>

<!-- Plugin repositories -->
<pluginRepositories>
    <pluginRepository>
        <id>spring-plugins</id>
        <url>https://repo.spring.io/plugins-release</url>
    </pluginRepository>
</pluginRepositories>
```

### Distribution Management (Where to deploy)
```xml
<distributionManagement>
    <repository>
        <id>releases</id>
        <url>https://nexus.mycompany.com/repository/releases/</url>
    </repository>
    <snapshotRepository>
        <id>snapshots</id>
        <url>https://nexus.mycompany.com/repository/snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

### Settings.xml (~/.m2/settings.xml)
```xml
<settings>
    <!-- Server credentials -->
    <servers>
        <server>
            <id>releases</id>       <!-- matches repository id -->
            <username>admin</username>
            <password>secret</password>
        </server>
    </servers>

    <!-- Mirrors (redirect all requests) -->
    <mirrors>
        <mirror>
            <id>nexus</id>
            <mirrorOf>*</mirrorOf>
            <url>https://nexus.mycompany.com/repository/maven-public/</url>
        </mirror>
    </mirrors>

    <!-- Proxy -->
    <proxies>
        <proxy>
            <id>corp-proxy</id>
            <active>true</active>
            <protocol>https</protocol>
            <host>proxy.mycompany.com</host>
            <port>8080</port>
            <username>user</username>
            <password>pass</password>
            <nonProxyHosts>localhost|*.mycompany.com</nonProxyHosts>
        </proxy>
    </proxies>

    <!-- Default active profiles -->
    <activeProfiles>
        <activeProfile>dev</activeProfile>
    </activeProfiles>

    <!-- Local repo path (override default) -->
    <localRepository>/custom/path/.m2/repository</localRepository>

</settings>
```

---

## 12. MAVEN WRAPPER

### What is Maven Wrapper?
- Ensures everyone uses the **same Maven version**.
- No need to install Maven globally — wrapper downloads it.
- Common in CI/CD and team projects.

### Setup
```bash
# Generate wrapper files
mvn wrapper:wrapper

# With specific version
mvn wrapper:wrapper -Dmaven=3.9.6
```

### Generated Files
```
my-project/
├── mvnw             ← Unix/macOS wrapper script
├── mvnw.cmd         ← Windows wrapper script
└── .mvn/
    └── wrapper/
        ├── maven-wrapper.jar
        └── maven-wrapper.properties
```

### Usage
```bash
# Use wrapper instead of `mvn`
./mvnw clean install        # Unix/macOS
mvnw.cmd clean install      # Windows

# Same commands as mvn
./mvnw package -DskipTests
./mvnw spring-boot:run
```

---

## 13. DEPENDENCY RESOLUTION

### How Maven Resolves Dependencies
```
1. Check local repository (~/.m2/repository)
2. Check configured remote repositories
3. Check Maven Central
4. Download and cache locally
```

### Transitive Dependencies
```
Your Project
└── spring-web
    ├── spring-beans (transitive)
    │   └── spring-core (transitive)
    └── spring-core (transitive)
```
Maven automatically includes **all transitive dependencies**.

### Dependency Conflict Resolution
- **Nearest-wins** strategy — closest dependency in the tree wins.
- **First-declaration-wins** — if same depth, first declared wins.

```
A → B 1.0
A → C → B 2.0

Result: B 1.0 wins (nearer to A)
```

### View Dependency Tree
```bash
mvn dependency:tree

# Output example:
# [INFO] com.myapp:my-project:jar:1.0.0
# [INFO] +- org.springframework:spring-web:jar:6.1.0:compile
# [INFO] |  +- org.springframework:spring-beans:jar:6.1.0:compile
# [INFO] |  \- org.springframework:spring-core:jar:6.1.0:compile
# [INFO] +- org.junit.jupiter:junit-jupiter:jar:5.10.0:test
# [INFO] \- com.mysql:mysql-connector-j:jar:8.2.0:runtime

# Filter
mvn dependency:tree -Dincludes=org.springframework
mvn dependency:tree -Dverbose    # show conflicts
```

### Force a Specific Version
```xml
<!-- Direct dependency always wins -->
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>lib-b</artifactId>
        <version>2.0</version>    <!-- override transitive 1.0 -->
    </dependency>
</dependencies>

<!-- Or use dependencyManagement to control transitive versions -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>lib-b</artifactId>
            <version>2.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## 14. RESOURCE FILTERING

### Enable Filtering
```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
            <includes>
                <include>**/*.properties</include>
                <include>**/*.yml</include>
            </includes>
        </resource>
        <!-- Don't filter binary files -->
        <resource>
            <directory>src/main/resources</directory>
            <filtering>false</filtering>
            <excludes>
                <exclude>**/*.properties</exclude>
                <exclude>**/*.yml</exclude>
            </excludes>
        </resource>
    </resources>
</build>
```

### Resource File with Placeholders
```properties
# src/main/resources/app.properties
app.name=${project.name}
app.version=${project.version}
app.environment=${env}
db.url=${db.url}
```

### After `mvn resources:resources`
```properties
# target/classes/app.properties
app.name=My Project
app.version=1.0.0
app.environment=development
db.url=jdbc:mysql://localhost:3306/devdb
```

---

## 15. ARCHETYPES

### What is an Archetype?
- **Project template** — generates a ready-to-use project structure.
- Standardizes project creation across teams.

### Common Archetypes
| Archetype | Description |
|-----------|-------------|
| `maven-archetype-quickstart` | Simple Java project |
| `maven-archetype-webapp` | Java web application |
| `maven-archetype-simple` | Minimal POM only |
| `maven-archetype-j2ee-simple` | J2EE project |
| `spring-boot-starter` | Spring Boot app (via Spring Initializr) |

### Create Project from Archetype
```bash
# Interactive mode
mvn archetype:generate

# Non-interactive — quickstart
mvn archetype:generate \
  -DgroupId=com.myapp \
  -DartifactId=my-project \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false

# Web project
mvn archetype:generate \
  -DgroupId=com.myapp \
  -DartifactId=my-webapp \
  -DarchetypeArtifactId=maven-archetype-webapp \
  -DinteractiveMode=false
```

### Generated quickstart Structure
```
my-project/
├── pom.xml
└── src/
    ├── main/java/com/myapp/App.java
    └── test/java/com/myapp/AppTest.java
```

---

## 16. COMMON POM EXAMPLES

### Simple Java Application
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>com.myapp</groupId>
    <artifactId>simple-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- JUnit 5 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Set main class for JAR -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.3.0</version>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>com.myapp.App</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

### Spring Boot Application
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>com.myapp</groupId>
    <artifactId>spring-app</artifactId>
    <version>1.0.0</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
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
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

---

## 17. TROUBLESHOOTING

### Common Issues & Fixes

| Problem | Solution |
|---------|----------|
| `BUILD FAILURE - Could not resolve dependencies` | Check internet, run `mvn -U install` to force update |
| `Cannot find symbol` / compilation error | Run `mvn clean compile`, check Java version |
| `No compiler is provided` | Install JDK (not JRE), set `JAVA_HOME` |
| `Tests failing` | Run `mvn test -X` for debug, check test names (`*Test.java`) |
| Stale cache / corrupt download | Delete `~/.m2/repository/<group>/<artifact>/<version>` |
| Wrong dependency version | Run `mvn dependency:tree -Dverbose` to trace |
| `Plugin not found` | Check plugin coordinates and repository config |
| `Non-resolvable parent POM` | Check parent coordinates, run with `-U` |
| Encoding issues | Set `<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>` |
| Snapshot not updating | Run `mvn -U install` (force snapshot update) |

### Debug Commands
```bash
mvn -X install                       # full debug output
mvn -e install                       # show stack traces
mvn help:effective-pom               # see resolved POM
mvn help:effective-settings          # see resolved settings
mvn dependency:tree -Dverbose        # see conflict details
mvn dependency:analyze               # unused/undeclared deps
```

### Clean Everything
```bash
mvn clean                            # delete target/
rm -rf ~/.m2/repository              # nuclear option — re-download all
mvn dependency:purge-local-repository  # re-download project deps
```

---

## 18. BEST PRACTICES

### POM Organization
```
1. Always specify encoding: project.build.sourceEncoding = UTF-8
2. Always specify Java version: maven.compiler.release
3. Use properties for version numbers — avoid hardcoding
4. Use dependencyManagement in parent POM
5. Use pluginManagement in parent POM
6. Keep pom.xml clean — remove default/unnecessary configs
```

### Dependencies
```
1. Use the narrowest scope possible (test, provided, runtime)
2. Exclude transitive dependencies that cause conflicts
3. Run mvn dependency:analyze regularly
4. Avoid system scope — use a repository manager instead
5. Pin versions — never rely on LATEST or RELEASE
6. Use BOM imports for framework dependency sets
```

### Versioning
```
1. Use SNAPSHOT for development: 1.0.0-SNAPSHOT
2. Use release versions for stable builds: 1.0.0
3. Follow Semantic Versioning: MAJOR.MINOR.PATCH
   - MAJOR: breaking changes
   - MINOR: new features (backward compatible)
   - PATCH: bug fixes
4. Never deploy SNAPSHOT to production
```

### Build
```
1. Always run from clean: mvn clean install
2. Don't skip tests in CI/CD
3. Use Maven Wrapper (mvnw) in projects
4. Use profiles for environment-specific configs
5. Use parallel builds for multi-module: mvn -T 1C install
6. .gitignore: target/, *.class, *.jar, .idea/, *.iml
```

### Project Structure
```
1. Follow Maven conventions — don't customize directory layout
2. Use multi-module for related projects
3. Separate concerns: api, core, web, data modules
4. Keep parent POM for shared config only (packaging=pom)
```

---

## 19. MAVEN vs GRADLE vs ANT

| Feature | Maven | Gradle | Ant |
|---------|-------|--------|-----|
| Config | XML (pom.xml) | Groovy/Kotlin DSL | XML (build.xml) |
| Convention | Strong | Flexible | None |
| Dependencies | Built-in | Built-in | Manual (Ivy) |
| Build Speed | Moderate | Fast (incremental, cache) | Variable |
| Learning Curve | Medium | Medium-High | Low |
| IDE Support | Excellent | Excellent | Good |
| Multi-module | Built-in | Built-in | Manual |
| Plugins | Extensive | Extensive | Moderate |
| Best For | Enterprise Java | Android, modern Java | Legacy projects |

---

## 20. QUICK REFERENCE CHEAT SHEET

### Most-Used Commands
```bash
mvn clean install              # clean build + install locally
mvn clean package              # clean build + package JAR/WAR
mvn clean test                 # clean + run tests
mvn clean compile              # clean + compile only
mvn dependency:tree            # view dependency tree
mvn versions:display-dependency-updates   # check for updates
mvn help:effective-pom         # see full resolved POM
mvn -DskipTests package        # package without running tests
```

### POM Quick Template
```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>GROUP</groupId>
    <artifactId>NAME</artifactId>
    <version>VERSION</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>...</groupId>
            <artifactId>...</artifactId>
            <version>...</version>
            <scope>...</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>...</groupId>
                <artifactId>...</artifactId>
                <version>...</version>
                <configuration>...</configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### Scope Cheat Sheet
```
compile   → default, available everywhere, packaged
provided  → compile+test only, NOT packaged (server provides)
runtime   → test+runtime only, packaged (e.g., JDBC drivers)
test      → test only, NOT packaged (JUnit, Mockito)
system    → like provided but from local path (avoid!)
import    → BOM imports in dependencyManagement only
```

### Key Files
```
pom.xml                      → Project configuration
~/.m2/settings.xml           → User-level Maven settings
~/.m2/repository/            → Local dependency cache
${MAVEN_HOME}/conf/settings.xml → Global Maven settings
mvnw / mvnw.cmd              → Maven Wrapper scripts
.mvn/wrapper/maven-wrapper.properties → Wrapper config
```

---

*Complete Maven Build Tool Reference — All 20 Topics*
*Last updated: April 2026*
