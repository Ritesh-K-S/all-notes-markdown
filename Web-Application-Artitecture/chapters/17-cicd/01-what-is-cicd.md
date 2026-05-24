# What is CI/CD? (Continuous Integration / Continuous Deployment)

> **What you'll learn**: How CI/CD automates the tedious, error-prone process of building, testing, and deploying software — so teams can ship code confidently, multiple times a day.

---

## Real-Life Analogy

Imagine you're running a **newspaper printing press** in the 1990s.

Every night, editors write articles, designers lay them out, proof-readers check for typos, and finally the press prints thousands of copies. If someone finds a mistake at 3 AM, the entire press run is wasted. The next day's edition is delayed. Chaos.

Now imagine a **modern digital newsroom**:
- A journalist types an article → it's **instantly spell-checked** (automated testing)
- An editor reviews it → **one click** publishes it to the website (automated deployment)
- If something's wrong, they fix it and re-publish in **minutes**, not hours

**CI/CD is like turning your software team from a 1990s printing press into a modern digital newsroom.** Every change is automatically checked, validated, and delivered — fast, safe, and repeatable.

---

## Core Concept Explained Step-by-Step

### Step 1: The Pain Without CI/CD

Without CI/CD, a typical release looks like this:

```
Developer A writes code for 2 weeks
Developer B writes code for 2 weeks
                    │
                    ▼
        ┌─────────────────────┐
        │   "MERGE DAY" 😱    │
        │ Conflicts everywhere │
        │ Tests broken         │
        │ Blame game begins    │
        └─────────────────────┘
                    │
                    ▼
        Manual testing (3 days)
                    │
                    ▼
        Manual deployment (Friday night 🍕)
                    │
                    ▼
        Something breaks in production (Saturday 2 AM 😭)
```

This is known as **"integration hell"** — and it was the norm for decades.

---

### Step 2: What is Continuous Integration (CI)?

**Continuous Integration** means every developer integrates (merges) their code into the main branch **frequently** — at least once a day — and each integration is **automatically verified** by building the code and running tests.

```
┌──────────────────────────────────────────────────────────────────┐
│                    CONTINUOUS INTEGRATION                          │
└──────────────────────────────────────────────────────────────────┘

Developer A ─── git push ───┐
                            │
Developer B ─── git push ───┼──▶ Shared Repository (GitHub/GitLab)
                            │           │
Developer C ─── git push ───┘           │
                                        ▼
                              ┌───────────────────┐
                              │   CI Server        │
                              │                   │
                              │  1. Pull code     │
                              │  2. Build it      │
                              │  3. Run tests     │
                              │  4. Report result │
                              └───────────────────┘
                                        │
                              ┌─────────┴─────────┐
                              │                   │
                              ▼                   ▼
                         ✅ PASS              ❌ FAIL
                      (merge OK)         (fix immediately!)
```

**Key rules of CI:**
1. Everyone commits to the main branch at least once daily
2. Every commit triggers an automated build + test
3. If the build breaks, fixing it is the **#1 priority** (drop everything!)
4. Tests must be fast (under 10 minutes ideally)

---

### Step 3: What is Continuous Delivery (CD)?

**Continuous Delivery** extends CI by ensuring code is **always in a deployable state**. After passing all automated tests, the code is ready to go to production with a **single button click**.

```
┌───────────────────────────────────────────────────────────────────────┐
│                     CONTINUOUS DELIVERY                                 │
└───────────────────────────────────────────────────────────────────────┘

  Code Change ──▶ Build ──▶ Unit Tests ──▶ Integration Tests ──▶ Staging
                                                                    │
                                                                    ▼
                                                        ┌────────────────┐
                                                        │  MANUAL GATE   │
                                                        │  (Human clicks │
                                                        │   "Deploy")    │
                                                        └────────────────┘
                                                                    │
                                                                    ▼
                                                              Production
```

The keyword here is **"always deployable."** You don't have to deploy every commit, but you *could* if you wanted to.

---

### Step 4: What is Continuous Deployment (CD)?

**Continuous Deployment** goes one step further — every change that passes all automated tests is **automatically deployed to production** without any human intervention.

```
┌───────────────────────────────────────────────────────────────────────┐
│                    CONTINUOUS DEPLOYMENT                                │
└───────────────────────────────────────────────────────────────────────┘

  Code Change ──▶ Build ──▶ Unit Tests ──▶ Integration Tests ──▶ Staging
                                                                    │
                                                                    ▼
                                                        ┌────────────────┐
                                                        │  NO HUMAN GATE │
                                                        │  (Automatic!)  │
                                                        └────────────────┘
                                                                    │
                                                                    ▼
                                                              Production
```

> **Note**: Continuous Delivery ≠ Continuous Deployment. Delivery means "ready to deploy." Deployment means "actually deployed." One requires a human click; the other doesn't.

---

### Step 5: The Full CI/CD Spectrum

```
┌─────────────────────────────────────────────────────────────────┐
│                      CI/CD MATURITY LEVELS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Level 0: Manual everything                                       │
│  ─────────────────────────                                        │
│  Build on developer's laptop, manual deploy via FTP/SSH           │
│                                                                   │
│  Level 1: Continuous Integration                                  │
│  ─────────────────────────────                                    │
│  Automated build + tests on every commit                          │
│                                                                   │
│  Level 2: Continuous Delivery                                     │
│  ────────────────────────────                                     │
│  Automated pipeline to staging, one-click deploy to prod          │
│                                                                   │
│  Level 3: Continuous Deployment                                   │
│  ─────────────────────────────                                    │
│  Fully automated: commit → production (no human in the loop)      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### The CI/CD Engine — What Happens Under the Hood

When you push code, here's what happens inside a CI/CD system:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CI/CD INTERNALS                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. TRIGGER (Webhook)                                                     │
│     GitHub/GitLab sends HTTP POST to CI server                            │
│     Payload: { repo, branch, commit_sha, author }                         │
│                                                                           │
│  2. SCHEDULER                                                             │
│     CI server queues a new "pipeline run"                                 │
│     Assigns it to an available "runner" (worker machine)                  │
│                                                                           │
│  3. RUNNER PROVISIONING                                                   │
│     Runner spins up a clean environment:                                  │
│     - Docker container (isolated)                                         │
│     - OR virtual machine                                                  │
│     - OR bare-metal server                                                │
│                                                                           │
│  4. CHECKOUT                                                              │
│     git clone + git checkout <commit_sha>                                 │
│                                                                           │
│  5. EXECUTION                                                             │
│     Run steps defined in pipeline config (YAML file):                     │
│     - Install dependencies                                                │
│     - Compile/Build                                                       │
│     - Run unit tests                                                      │
│     - Run integration tests                                               │
│     - Build Docker image                                                  │
│     - Push to registry                                                    │
│     - Deploy to environment                                               │
│                                                                           │
│  6. REPORTING                                                             │
│     - Send status back to Git (✅ or ❌)                                  │
│     - Notify team (Slack, email)                                          │
│     - Store logs and artifacts                                            │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Webhooks — The Trigger Mechanism

```
Developer pushes code
        │
        ▼
┌──────────────┐         HTTP POST (webhook)        ┌──────────────────┐
│   GitHub     │ ───────────────────────────────▶   │   CI Server      │
│              │   {                                 │   (Jenkins/       │
│              │     "ref": "refs/heads/main",      │    GitHub Actions)│
│              │     "after": "abc123...",           │                  │
│              │     "repository": {...}             │                  │
│              │   }                                 │                  │
└──────────────┘                                    └──────────────────┘
```

### Pipeline Configuration (YAML)

Almost all modern CI/CD tools use a YAML file to define the pipeline:

```yaml
# .github/workflows/ci.yml (GitHub Actions example)
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run tests
        run: pytest --cov=app tests/
```

---

## Code Examples

### Python: A Simple CI-Ready Project Structure

```python
# project structure for CI/CD readiness
# my_app/
# ├── app/
# │   ├── __init__.py
# │   └── calculator.py
# ├── tests/
# │   ├── __init__.py
# │   └── test_calculator.py
# ├── requirements.txt
# ├── Dockerfile
# └── .github/workflows/ci.yml

# app/calculator.py
class Calculator:
    """A simple calculator to demonstrate testable code."""
    
    def add(self, a: float, b: float) -> float:
        return a + b
    
    def divide(self, a: float, b: float) -> float:
        if b == 0:
            raise ValueError("Cannot divide by zero")
        return a / b

# tests/test_calculator.py
import pytest
from app.calculator import Calculator

class TestCalculator:
    def setup_method(self):
        self.calc = Calculator()
    
    def test_add(self):
        assert self.calc.add(2, 3) == 5
    
    def test_divide(self):
        assert self.calc.divide(10, 2) == 5.0
    
    def test_divide_by_zero(self):
        with pytest.raises(ValueError):
            self.calc.divide(10, 0)
```

### Java: A Simple CI-Ready Project with Maven

```java
// src/main/java/com/example/Calculator.java
package com.example;

public class Calculator {
    public double add(double a, double b) {
        return a + b;
    }
    
    public double divide(double a, double b) {
        if (b == 0) {
            throw new ArithmeticException("Cannot divide by zero");
        }
        return a / b;
    }
}

// src/test/java/com/example/CalculatorTest.java
package com.example;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {
    private final Calculator calc = new Calculator();
    
    @Test
    void testAdd() {
        assertEquals(5.0, calc.add(2, 3));
    }
    
    @Test
    void testDivide() {
        assertEquals(5.0, calc.divide(10, 2));
    }
    
    @Test
    void testDivideByZero() {
        assertThrows(ArithmeticException.class, 
            () -> calc.divide(10, 0));
    }
}
```

```xml
<!-- pom.xml (Maven build file for CI) -->
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>calculator</artifactId>
    <version>1.0-SNAPSHOT</version>
    
    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.1.2</version>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Infrastructure Examples

### Minimal CI Pipeline with Docker

```dockerfile
# Dockerfile for the application
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

```yaml
# docker-compose.test.yml — Run tests in containers
version: '3.8'
services:
  test:
    build: .
    command: pytest tests/ --cov=app --cov-report=xml
    environment:
      - DATABASE_URL=postgresql://test:test@db:5432/testdb
    depends_on:
      - db
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: testdb
```

### GitHub Actions Complete Pipeline

```yaml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - run: pytest tests/ --cov=app
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/postgres

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## Real-World Example

### How Big Companies Use CI/CD

| Company | CI/CD Practice | Scale |
|---------|---------------|-------|
| **Google** | 50,000+ commits/day to monorepo, automated testing on every change | ~500M test executions/day |
| **Amazon** | Deploy every 11.7 seconds on average | ~150M deployments/year |
| **Netflix** | Full CI/CD with canary deployments, automated rollback | Deploy hundreds of times/day |
| **Facebook/Meta** | Continuous push to production, feature flags for gradual rollout | Thousands of engineers committing daily |
| **Etsy** | Pioneer of continuous deployment — deploy 50+ times/day | Small teams deploying independently |

### Netflix's CI/CD Approach

```
Developer commits
        │
        ▼
┌─────────────────┐
│  Spinnaker CI   │ ← Netflix's custom CD platform
│  1. Build       │
│  2. Bake AMI    │ (create machine image)
│  3. Test        │
└─────────────────┘
        │
        ▼
┌─────────────────┐
│ Canary Deploy   │ ← Deploy to 1% of users
│ Compare metrics │
│ vs. baseline    │
└─────────────────┘
        │
   Pass? ──── No ──▶ Automatic Rollback
        │
       Yes
        │
        ▼
┌─────────────────┐
│ Regional Deploy │ ← Gradually roll out to all regions
└─────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| **Long-lived feature branches** | Merge conflicts pile up, integration pain | Merge to main daily (use feature flags to hide unfinished work) |
| **Slow test suites** | Developers skip CI or ignore failures | Keep CI under 10 min; parallelize tests |
| **Ignoring broken builds** | "Someone else will fix it" culture develops | Make fixing a broken build the #1 priority |
| **No test coverage** | CI without tests is just "continuous building" | Aim for 80%+ meaningful coverage |
| **Testing only in CI** | Surprises at commit time | Run tests locally before pushing |
| **Deploying on Fridays** | Nobody wants to fix production on weekends | Deploy early in the week, or have reliable rollback |
| **No rollback plan** | Stuck with broken production | Always have a one-click rollback strategy |
| **Hardcoded secrets in CI config** | Security breach waiting to happen | Use secrets management (encrypted variables) |

---

## When to Use / When NOT to Use

### Use CI/CD When:
- ✅ You have more than one developer on the team
- ✅ You deploy more than once a month
- ✅ You want fast feedback on code quality
- ✅ You want reproducible, auditable deployments
- ✅ You're working on any production software

### When to Start Simple:
- 🟡 Solo developer on a hobby project → start with just CI (automated tests)
- 🟡 Regulated industries (healthcare, banking) → use Continuous Delivery (manual approval gate) rather than Continuous Deployment
- 🟡 Legacy systems without tests → add CI gradually (start with build verification)

### NOT Appropriate for Continuous Deployment:
- ❌ Systems requiring regulatory approval before each release
- ❌ Hardware/firmware deployments (can't easily rollback)
- ❌ No automated test coverage exists yet (deploy manually until tests are in place)

---

## Key Takeaways

- 🔑 **CI** = merge frequently + automated build + automated tests on every commit
- 🔑 **Continuous Delivery** = CI + always-deployable code + one-click deploy
- 🔑 **Continuous Deployment** = Continuous Delivery + automatic deploy (no human gate)
- 🔑 CI/CD eliminates "integration hell" and "deployment fear"
- 🔑 The pipeline is defined as **code** (YAML) and versioned alongside your application
- 🔑 Fast feedback loops (< 10 minutes) are critical — slow CI gets ignored
- 🔑 CI/CD is a **cultural practice** as much as a technical one — it requires discipline from the whole team

---

## What's Next?

Now that you understand *what* CI/CD is and *why* it matters, the next chapter dives into the **anatomy of a CI/CD pipeline** — the stages, jobs, and steps that make up a real-world automated pipeline.

**Next: [CI/CD Pipelines — Building, Testing & Deploying Automatically](./02-cicd-pipelines.md)**
