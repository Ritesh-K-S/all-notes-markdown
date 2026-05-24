# Chaos Engineering — Breaking Things on Purpose (Netflix Chaos Monkey)

> **What you'll learn**: How to proactively inject failures into production systems to discover weaknesses BEFORE they cause real outages — the discipline that transformed Netflix from "hope it works" to "we KNOW it works."

---

## Real-Life Analogy

Think about **fire drills** in a building:

- You KNOW a fire is unlikely today.
- But you still practice evacuating.
- Why? Because when a real fire happens, you don't want the first time people use the fire exits to be during actual panic.

**Chaos Engineering** is fire drills for your software:

- You KNOW your database probably won't crash today.
- But you **intentionally crash it in production** (in a controlled way).
- Why? To discover: Do your fallbacks work? Does your team respond quickly? Does the system recover automatically?

```
TRADITIONAL APPROACH:
┌─────────────────────────────────────────┐
│  "Our system is resilient because       │
│   we DESIGNED it to be resilient."      │
│                                         │
│  Reality: Untested resilience =         │
│           imaginary resilience          │
└─────────────────────────────────────────┘

CHAOS ENGINEERING APPROACH:
┌─────────────────────────────────────────┐
│  "Our system is resilient because       │
│   we BROKE it and WATCHED it recover."  │
│                                         │
│  Reality: Tested resilience =           │
│           REAL resilience               │
└─────────────────────────────────────────┘
```

---

## Core Concept Explained Step-by-Step

### What is Chaos Engineering?

**Chaos Engineering** is the discipline of experimenting on a system to build confidence in its ability to withstand turbulent conditions in production.

```
┌─────────────────────────────────────────────────────────────────┐
│           THE CHAOS ENGINEERING PROCESS                          │
│                                                                 │
│   Step 1: DEFINE "steady state"                                │
│     What does "normal" look like?                               │
│     (e.g., p99 latency < 200ms, error rate < 0.1%)             │
│                                                                 │
│   Step 2: HYPOTHESIZE                                          │
│     "If Service X goes down, the system should                  │
│      continue serving users with fallback data"                 │
│                                                                 │
│   Step 3: INJECT FAILURE                                       │
│     Kill Service X in production (controlled blast radius)      │
│                                                                 │
│   Step 4: OBSERVE                                              │
│     Did steady state hold? Did fallbacks work?                  │
│     Did alerts fire? Did system auto-recover?                   │
│                                                                 │
│   Step 5: LEARN & FIX                                          │
│     Found a weakness? Fix it!                                   │
│     Then run the experiment again to verify.                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Types of Failures to Inject

```
┌────────────────────────────────────────────────────────────────────┐
│              CHAOS EXPERIMENT CATEGORIES                            │
├─────────────────────┬──────────────────────────────────────────────┤
│ Category            │ Examples                                     │
├─────────────────────┼──────────────────────────────────────────────┤
│ INFRASTRUCTURE      │ • Kill a VM/container                        │
│                     │ • Terminate a random pod                     │
│                     │ • Detach a disk                              │
│                     │ • Fill disk to 100%                          │
│                     │ • Exhaust CPU/memory                         │
├─────────────────────┼──────────────────────────────────────────────┤
│ NETWORK             │ • Add latency (200ms delay)                  │
│                     │ • Drop packets (10% packet loss)             │
│                     │ • Partition network (split brain)            │
│                     │ • DNS failure                                │
│                     │ • Block port                                 │
├─────────────────────┼──────────────────────────────────────────────┤
│ APPLICATION         │ • Kill a service process                     │
│                     │ • Return errors from API                     │
│                     │ • Slow down responses (artificial latency)   │
│                     │ • Exhaust thread pool                        │
│                     │ • Trigger garbage collection                 │
├─────────────────────┼──────────────────────────────────────────────┤
│ STATE               │ • Corrupt cache entries                      │
│                     │ • Database failover                          │
│                     │ • Delete Kafka consumer offsets              │
│                     │ • Redis cluster node failure                 │
├─────────────────────┼──────────────────────────────────────────────┤
│ DEPENDENCY          │ • Third-party API timeout                    │
│                     │ • CDN failure                                │
│                     │ • DNS provider outage                        │
│                     │ • Certificate expiration                     │
└─────────────────────┴──────────────────────────────────────────────┘
```

### The Chaos Engineering Maturity Model

```
Level 0: NO CHAOS
  "We hope our system is resilient"
  → No testing, no confidence

Level 1: CHAOS IN DEV/STAGING
  "We test failures in non-production"
  → Better than nothing, but staging ≠ production

Level 2: CHAOS IN PRODUCTION (manual)
  "We manually inject failures during business hours"
  → Game days, controlled experiments
  → Real production insights

Level 3: AUTOMATED CHAOS IN PRODUCTION
  "Failures are injected automatically and continuously"
  → Chaos Monkey kills instances randomly every day
  → System MUST be resilient to survive

Level 4: CHAOS AS CULTURE
  "Everyone builds for failure from day one"
  → Resilience is a requirement, not an afterthought
  → New services must pass chaos tests before production
```

---

## How It Works Internally

### Netflix Chaos Monkey Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│              NETFLIX CHAOS MONKEY                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌────────────────────────┐                                        │
│  │   Chaos Monkey Process  │                                        │
│  │   (Runs on schedule)    │                                        │
│  └────────────┬────────────┘                                        │
│               │                                                     │
│               │  1. Query: "What instances are running?"             │
│               ▼                                                     │
│  ┌────────────────────────┐                                        │
│  │   AWS Auto Scaling      │                                        │
│  │   Groups                │                                        │
│  └────────────┬────────────┘                                        │
│               │                                                     │
│               │  2. Select random instance                          │
│               ▼                                                     │
│  ┌────────────────────────┐                                        │
│  │   Terminate Instance!   │  ← Random production server killed!   │
│  └────────────┬────────────┘                                        │
│               │                                                     │
│               │  3. Observe: Does the system recover?               │
│               ▼                                                     │
│  ┌────────────────────────┐                                        │
│  │   Monitoring & Alerts   │                                        │
│  │   • Did latency spike?  │                                        │
│  │   • Did errors increase?│                                        │
│  │   • Did auto-scaling    │                                        │
│  │     replace instance?   │                                        │
│  └────────────────────────┘                                        │
│                                                                     │
│  Runs: During business hours only (weekdays, 9am-3pm)              │
│  Rule: Every service MUST survive random instance termination       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### The Simian Army (Netflix's Full Suite)

```
┌────────────────────────────────────────────────────────────────────┐
│            NETFLIX'S SIMIAN ARMY                                    │
├──────────────────┬─────────────────────────────────────────────────┤
│ Tool             │ What it does                                    │
├──────────────────┼─────────────────────────────────────────────────┤
│ Chaos Monkey     │ Kills random instances                          │
│                  │ (Tests: instance-level resilience)              │
├──────────────────┼─────────────────────────────────────────────────┤
│ Chaos Kong       │ Takes down an ENTIRE AWS region                 │
│                  │ (Tests: multi-region failover)                  │
├──────────────────┼─────────────────────────────────────────────────┤
│ Latency Monkey   │ Adds artificial delays to network calls         │
│                  │ (Tests: timeout handling, fallbacks)            │
├──────────────────┼─────────────────────────────────────────────────┤
│ Chaos Gorilla    │ Takes down an entire availability zone          │
│                  │ (Tests: AZ-level failover)                      │
├──────────────────┼─────────────────────────────────────────────────┤
│ Conformity Monkey│ Finds instances not following best practices    │
│                  │ (Tests: configuration compliance)               │
├──────────────────┼─────────────────────────────────────────────────┤
│ Security Monkey  │ Finds security vulnerabilities                  │
│                  │ (Tests: security posture)                       │
└──────────────────┴─────────────────────────────────────────────────┘
```

### Experiment Design Template

```
┌─────────────────────────────────────────────────────────────────┐
│            CHAOS EXPERIMENT TEMPLATE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EXPERIMENT: "Payment service instance failure"                 │
│                                                                 │
│  STEADY STATE:                                                  │
│    • Order success rate: 99.9%                                  │
│    • Payment p99 latency: < 500ms                              │
│    • Error rate: < 0.1%                                        │
│                                                                 │
│  HYPOTHESIS:                                                    │
│    "If one payment service instance dies, the load balancer     │
│     will route to healthy instances within 5 seconds.           │
│     Order success rate will remain above 99.5%."               │
│                                                                 │
│  BLAST RADIUS:                                                  │
│    • Target: 1 out of 4 payment service pods                   │
│    • Impact: ~25% of payment traffic briefly affected          │
│    • Abort if: error rate > 5% for > 30 seconds               │
│                                                                 │
│  METHOD:                                                        │
│    1. Kill one payment-service pod                              │
│    2. Observe for 5 minutes                                    │
│    3. Check: did K8s restart the pod?                           │
│    4. Check: did traffic route to healthy pods?                 │
│    5. Check: did circuit breaker activate?                      │
│                                                                 │
│  ABORT CONDITIONS:                                              │
│    • Error rate > 5% for > 30 seconds                          │
│    • Any customer-facing 500 errors                            │
│    • Revenue impact detected                                    │
│                                                                 │
│  RESULT: [PASS/FAIL + findings]                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Simple Chaos Monkey for Kubernetes

```python
import random
import time
import logging
from kubernetes import client, config

logger = logging.getLogger(__name__)

class ChaosMonkey:
    """
    A simple Chaos Monkey that randomly kills pods in a namespace.
    Runs during business hours only.
    """
    
    def __init__(self, namespace: str, excluded_labels: list = None):
        config.load_incluster_config()  # Running inside K8s
        self.v1 = client.CoreV1Api()
        self.namespace = namespace
        self.excluded_labels = excluded_labels or ["app=database", "critical=true"]
    
    def get_killable_pods(self) -> list:
        """Get list of pods that are safe to kill."""
        pods = self.v1.list_namespaced_pod(self.namespace)
        killable = []
        
        for pod in pods.items:
            # Skip system pods and excluded labels
            if pod.metadata.labels and any(
                self._has_label(pod, label) for label in self.excluded_labels
            ):
                continue
            
            # Only kill running pods
            if pod.status.phase == "Running":
                killable.append(pod)
        
        return killable
    
    def kill_random_pod(self) -> dict:
        """Kill a random pod and return experiment details."""
        killable = self.get_killable_pods()
        
        if not killable:
            logger.info("No killable pods found")
            return {"action": "none", "reason": "no eligible pods"}
        
        victim = random.choice(killable)
        pod_name = victim.metadata.name
        
        logger.warning(f"🐒 Chaos Monkey killing pod: {pod_name}")
        
        # Record pre-kill state
        experiment = {
            "pod_name": pod_name,
            "namespace": self.namespace,
            "killed_at": time.time(),
            "labels": victim.metadata.labels
        }
        
        # Kill it!
        self.v1.delete_namespaced_pod(
            name=pod_name,
            namespace=self.namespace,
            body=client.V1DeleteOptions(grace_period_seconds=0)
        )
        
        logger.warning(f"🐒 Pod {pod_name} terminated!")
        return experiment
    
    def _has_label(self, pod, label_str: str) -> bool:
        key, value = label_str.split("=")
        return pod.metadata.labels.get(key) == value
    
    def run_experiment(self):
        """Run a chaos experiment with safety checks."""
        # Safety: Only run during business hours
        import datetime
        now = datetime.datetime.now()
        if now.weekday() >= 5:  # Weekend
            logger.info("Weekend — Chaos Monkey sleeping 🐒💤")
            return
        if now.hour < 9 or now.hour >= 15:  # Outside 9am-3pm
            logger.info("Outside business hours — sleeping")
            return
        
        experiment = self.kill_random_pod()
        
        # Wait and observe
        time.sleep(30)
        
        # Check: Did K8s restart the pod?
        pods = self.v1.list_namespaced_pod(self.namespace)
        running = [p for p in pods.items if p.status.phase == "Running"]
        logger.info(f"Running pods after kill: {len(running)}")
        
        return experiment

# Schedule to run periodically
if __name__ == "__main__":
    monkey = ChaosMonkey(
        namespace="production",
        excluded_labels=["critical=true", "tier=0"]
    )
    
    while True:
        monkey.run_experiment()
        # Run every 2 hours during business hours
        time.sleep(7200)
```

### Python — Network Chaos Injection

```python
import subprocess
import time
import contextlib

class NetworkChaos:
    """Inject network failures using Linux tc (traffic control)."""
    
    @staticmethod
    def add_latency(interface: str, delay_ms: int, jitter_ms: int = 0):
        """Add artificial latency to network interface."""
        cmd = f"tc qdisc add dev {interface} root netem delay {delay_ms}ms"
        if jitter_ms:
            cmd += f" {jitter_ms}ms"
        subprocess.run(cmd.split(), check=True)
        print(f"Added {delay_ms}ms (±{jitter_ms}ms) latency to {interface}")
    
    @staticmethod
    def add_packet_loss(interface: str, loss_percent: float):
        """Add packet loss to network interface."""
        cmd = f"tc qdisc add dev {interface} root netem loss {loss_percent}%"
        subprocess.run(cmd.split(), check=True)
        print(f"Added {loss_percent}% packet loss to {interface}")
    
    @staticmethod
    def remove_chaos(interface: str):
        """Remove all network chaos."""
        subprocess.run(f"tc qdisc del dev {interface} root".split(), check=False)
        print(f"Removed all network chaos from {interface}")
    
    @staticmethod
    @contextlib.contextmanager
    def temporary_latency(interface: str, delay_ms: int, duration_sec: int):
        """Context manager: add latency for a fixed duration, then clean up."""
        try:
            NetworkChaos.add_latency(interface, delay_ms)
            print(f"Chaos active for {duration_sec} seconds...")
            time.sleep(duration_sec)
        finally:
            NetworkChaos.remove_chaos(interface)
            print("Chaos experiment completed. Network restored.")

# Usage:
# with NetworkChaos.temporary_latency("eth0", delay_ms=500, duration_sec=60):
#     # For 60 seconds, all outgoing traffic has 500ms added latency
#     # Observe: Do timeouts fire? Do circuit breakers trip?
#     pass
```

### Java — Chaos Engineering with LitmusChaos (Kubernetes)

```java
import io.fabric8.kubernetes.client.KubernetesClient;
import io.fabric8.kubernetes.client.KubernetesClientBuilder;
import io.fabric8.kubernetes.api.model.Pod;

import java.util.List;
import java.util.Random;
import java.util.stream.Collectors;

public class ChaosExperimentRunner {
    
    private final KubernetesClient k8sClient;
    private final String namespace;
    private final Random random = new Random();
    
    public ChaosExperimentRunner(String namespace) {
        this.k8sClient = new KubernetesClientBuilder().build();
        this.namespace = namespace;
    }
    
    /**
     * Experiment: Kill a random pod and verify recovery.
     */
    public ExperimentResult runPodKillExperiment(String appLabel) {
        // Get target pods
        List<Pod> pods = k8sClient.pods()
            .inNamespace(namespace)
            .withLabel("app", appLabel)
            .list()
            .getItems()
            .stream()
            .filter(p -> "Running".equals(p.getStatus().getPhase()))
            .collect(Collectors.toList());
        
        if (pods.size() <= 1) {
            return ExperimentResult.skipped("Only 1 pod running — too risky");
        }
        
        // Record steady state
        int initialPodCount = pods.size();
        long preKillTimestamp = System.currentTimeMillis();
        
        // Kill random pod
        Pod victim = pods.get(random.nextInt(pods.size()));
        String victimName = victim.getMetadata().getName();
        
        System.out.printf("🐒 Killing pod: %s (1 of %d)%n", victimName, pods.size());
        k8sClient.pods().inNamespace(namespace).withName(victimName).delete();
        
        // Observe recovery
        try { Thread.sleep(30000); } catch (InterruptedException e) {}
        
        // Check: Did Kubernetes restore the pod?
        int currentPodCount = k8sClient.pods()
            .inNamespace(namespace)
            .withLabel("app", appLabel)
            .list()
            .getItems()
            .stream()
            .filter(p -> "Running".equals(p.getStatus().getPhase()))
            .mapToInt(p -> 1)
            .sum();
        
        long recoveryTime = System.currentTimeMillis() - preKillTimestamp;
        
        ExperimentResult result = new ExperimentResult();
        result.setPodKilled(victimName);
        result.setInitialCount(initialPodCount);
        result.setFinalCount(currentPodCount);
        result.setRecoveryTimeMs(recoveryTime);
        result.setPassed(currentPodCount >= initialPodCount);
        
        System.out.printf("Result: %s (recovered in %dms, pods: %d → %d)%n",
            result.isPassed() ? "PASS ✓" : "FAIL ✗",
            recoveryTime, initialPodCount, currentPodCount);
        
        return result;
    }
}
```

### Java — Resilience Testing with Toxiproxy

```java
import eu.rekawek.toxiproxy.ToxiproxyClient;
import eu.rekawek.toxiproxy.Proxy;
import eu.rekawek.toxiproxy.model.ToxicDirection;

/**
 * Use Toxiproxy to inject network failures between services.
 * Toxiproxy sits between your app and its dependency.
 * 
 * App ──▶ Toxiproxy (port 9999) ──▶ Real Database (port 5432)
 *              ↑
 *         Inject failures here!
 */
public class ToxiproxyExperiment {
    
    public static void main(String[] args) throws Exception {
        ToxiproxyClient client = new ToxiproxyClient("localhost", 8474);
        
        // Create a proxy for the database connection
        Proxy dbProxy = client.createProxy(
            "postgres-proxy",
            "0.0.0.0:9999",        // Listen address (app connects here)
            "postgres-host:5432"    // Upstream (real database)
        );
        
        // Experiment 1: Add 500ms latency to all DB queries
        System.out.println("Injecting 500ms latency to DB...");
        dbProxy.toxics().latency("db-latency", ToxicDirection.DOWNSTREAM, 500);
        
        // Run your tests/application for 60 seconds
        Thread.sleep(60000);
        
        // Remove the toxic
        dbProxy.toxics().get("db-latency").remove();
        System.out.println("Latency removed. Checking steady state...");
        
        // Experiment 2: Simulate DB connection timeout
        System.out.println("Simulating DB timeout...");
        dbProxy.toxics().timeout("db-timeout", ToxicDirection.DOWNSTREAM, 0);
        
        // App should activate circuit breaker and use fallback
        Thread.sleep(30000);
        
        dbProxy.toxics().get("db-timeout").remove();
        System.out.println("Experiments complete.");
    }
}
```

---

## Infrastructure Examples

### LitmusChaos — Kubernetes-Native Chaos

```yaml
# pod-delete-experiment.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: payment-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=payment-service"
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"           # Kill for 60 seconds
            - name: CHAOS_INTERVAL
              value: "10"           # Every 10 seconds
            - name: FORCE
              value: "true"         # Force delete (no grace period)
            - name: PODS_AFFECTED_PERC
              value: "25"           # Kill 25% of pods
        probe:
          - name: "check-error-rate"
            type: "promProbe"
            mode: "Continuous"
            runProperties:
              probeTimeout: 5
              interval: 5
            promProbe/inputs:
              endpoint: "http://prometheus:9090"
              query: "rate(http_errors_total[1m])"
              comparator:
                type: "float"
                criteria: "<="
                value: "0.05"       # Abort if error rate > 5%
```

### Gremlin — Enterprise Chaos Platform

```yaml
# Gremlin attack configuration
attack:
  type: "latency"
  target:
    type: "container"
    containers:
      labels:
        app: "recommendation-service"
    limit: 2              # Only affect 2 containers
  params:
    ms: 2000              # Add 2 seconds of latency
    duration: 300         # For 5 minutes
  
  # Safety: Abort conditions
  halt_conditions:
    - metric: "error_rate"
      threshold: 5        # Stop if errors > 5%
    - metric: "p99_latency_ms"
      threshold: 10000    # Stop if p99 > 10s
---
# Network partition attack
attack:
  type: "blackhole"       # Drop all traffic
  target:
    type: "container"
    containers:
      labels:
        app: "cache-service"
  params:
    duration: 120         # 2 minutes
    direction: "ingress"  # Block incoming traffic only
```

### AWS Fault Injection Simulator (FIS)

```json
{
  "description": "Test payment service resilience to EC2 instance failure",
  "targets": {
    "payment-instances": {
      "resourceType": "aws:ec2:instance",
      "selectionMode": "COUNT(1)",
      "resourceTags": {
        "service": "payment"
      },
      "filters": [
        {
          "path": "State.Name",
          "values": ["running"]
        }
      ]
    }
  },
  "actions": {
    "kill-instance": {
      "actionId": "aws:ec2:terminate-instances",
      "parameters": {},
      "targets": {
        "Instances": "payment-instances"
      }
    }
  },
  "stopConditions": [
    {
      "source": "aws:cloudwatch:alarm",
      "value": "arn:aws:cloudwatch:us-east-1:123456789:alarm:PaymentErrorRateHigh"
    }
  ],
  "roleArn": "arn:aws:iam::123456789:role/FISRole",
  "tags": {
    "experiment": "payment-instance-failure"
  }
}
```

### Chaos Mesh (Kubernetes-native, CNCF)

```yaml
# network-delay.yaml — Add latency between services
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: delay-payment-to-db
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: payment-service
  delay:
    latency: "500ms"
    jitter: "100ms"
    correlation: "50"
  target:
    selector:
      namespaces:
        - production
      labelSelectors:
        app: postgres
    mode: all
  duration: "5m"
  scheduler:
    cron: "@every 24h"    # Run daily
---
# pod-kill.yaml — Random pod termination
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-random-pod
spec:
  action: pod-kill
  mode: one               # Kill just one pod
  selector:
    namespaces:
      - production
    labelSelectors:
      tier: "non-critical"
  scheduler:
    cron: "0 10 * * 1-5"  # Weekdays at 10am
```

---

## Real-World Example

### Netflix's Chaos Engineering Journey

```
┌─────────────────────────────────────────────────────────────────────┐
│            NETFLIX CHAOS ENGINEERING TIMELINE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  2010: THE INCIDENT                                                 │
│  Netflix moves to AWS. Infrastructure is unreliable.                │
│  Random failures cause outages. Team is reactive.                   │
│                                                                     │
│  2011: CHAOS MONKEY BORN                                           │
│  "If instances fail randomly anyway, let's fail them ON PURPOSE     │
│   so we're FORCED to build resilient systems."                      │
│                                                                     │
│  2012: SIMIAN ARMY                                                  │
│  Expanded to: Latency Monkey, Chaos Gorilla, Conformity Monkey     │
│                                                                     │
│  2014: CHAOS KONG (Region Failure)                                  │
│  Netflix simulates ENTIRE AWS region going down.                    │
│  All traffic fails over to another region automatically.            │
│                                                                     │
│  2015: FAILURE INJECTION TESTING (FIT)                              │
│  More surgical: inject specific failures between specific services. │
│  Test exact hypotheses rather than random chaos.                    │
│                                                                     │
│  2017: CHAOS AUTOMATION PLATFORM (ChAP)                            │
│  Fully automated experiments run continuously.                      │
│  New services must pass chaos tests before production.              │
│                                                                     │
│  TODAY: CHAOS IS CULTURE                                            │
│  Every engineer at Netflix accepts:                                 │
│  "If my service can't survive Chaos Monkey, it's not production-    │
│   ready."                                                           │
│                                                                     │
│  RESULT: Netflix has 99.99% uptime serving 200M+ users.            │
│  Most "outages" are invisible to users.                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### How Google Runs DiRT (Disaster Recovery Testing)

```
┌─────────────────────────────────────────────────────────────────────┐
│           GOOGLE'S DiRT PROGRAM                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  DiRT = Disaster Recovery Testing                                    │
│                                                                     │
│  Scale: Google runs DiRT tests that take down ENTIRE data centers   │
│                                                                     │
│  Examples:                                                          │
│  • Simulated earthquake: "What if our California DC is gone?"       │
│  • Power failure: Shut down racks to test UPS failover              │
│  • Network partition: Cut fiber between data centers                │
│  • Software bug: Deploy bad config to see if rollback works         │
│                                                                     │
│  Key principles:                                                    │
│  • Test during BUSINESS HOURS (not at 3am when it doesn't matter)  │
│  • Real traffic affected (not synthetic)                            │
│  • Clear rollback plan before starting                              │
│  • Entire team participates (not just SRE)                         │
│                                                                     │
│  Outcome: Google found and fixed hundreds of hidden failures         │
│  that would have caused real outages.                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Uber's Chaos Engineering

```
Uber's approach:
  
1. SERVICE-LEVEL CHAOS
   • Random service restarts during peak hours
   • Verify: ride matching still works within SLA

2. NETWORK CHAOS
   • Add latency between driver-matching and pricing
   • Verify: prices still computed correctly (maybe with delay)

3. DATA CENTER FAILOVER
   • Drain an entire DC and shift to another
   • Verify: ongoing rides aren't interrupted

4. DEPENDENCY ISOLATION  
   • Block access to maps API
   • Verify: cached routes used, ETA shows "approximate"

Key metric: "No rides should be lost during an experiment"
```

---

## Common Mistakes / Pitfalls

### 1. Starting Chaos in Production Without Preparation

```
❌ BAD: "Let's kill a database in production and see what happens!"
   → No monitoring in place
   → No abort plan
   → No one knows what "normal" looks like
   → Chaos causes REAL outage

✅ GOOD: 
   1. First: establish baseline metrics
   2. Then: run in staging
   3. Then: run in production with TINY blast radius
   4. Gradually increase scope as confidence grows
```

### 2. No Abort Button (Kill Switch)

```
❌ BAD: Experiment starts, things go wrong, no way to stop it

✅ GOOD: Every experiment has:
   - Automatic abort conditions (error rate > threshold)
   - Manual kill switch (one-click stop)
   - Auto-rollback after duration expires
   - Alert to on-call if things go sideways
```

### 3. Running Chaos During Peak Traffic Without Approval

```
❌ BAD: Chaos experiment during Black Friday sale
   → Slight hiccup + 100x normal traffic = massive revenue loss

✅ GOOD: 
   - Run during business hours (to test real traffic patterns)
   - But NOT during known peak events
   - Get buy-in from team and management
   - Start small (1 pod, not 50% of fleet)
```

### 4. Only Injecting "Easy" Failures

```
❌ BAD: Only killing pods (K8s auto-restarts them in seconds)
   → Never test the HARD failures that actually cause outages

✅ GOOD: Test the scary stuff too:
   - Network partitions (split brain)
   - Slow responses (harder to detect than crashes)
   - Data corruption / inconsistency
   - Clock skew between nodes
   - Certificate expiration
   - DNS failure
```

### 5. Not Fixing What You Find

```
❌ BAD: Run chaos → find weakness → "We'll fix it later" → never fix it

✅ GOOD: Every chaos experiment produces:
   - Finding: "Payment service doesn't handle DB failover"
   - Action item: "Add circuit breaker + read replica fallback"
   - Deadline: "Before next experiment (2 weeks)"
   - Verification: Re-run same experiment to confirm fix
```

---

## When to Use / When NOT to Use

### ✅ Use Chaos Engineering When

| Scenario | Experiment Type |
|----------|----------------|
| Moving to microservices | Kill individual services, verify independence |
| Before major release | Inject failures to catch regressions |
| After adding redundancy | Verify failover actually works |
| New on-call engineers | Game day exercises for training |
| Compliance requirements | Prove disaster recovery works |
| Multi-region setup | Test region failover under real traffic |
| After long stability period | "Calm before the storm" — find hidden issues |

### ❌ When NOT to Use (or Be Very Careful)

| Scenario | Why |
|----------|-----|
| No monitoring/observability | You can't measure impact without metrics! |
| No auto-scaling / redundancy | Nothing to fail over TO |
| During peak traffic events | Black Friday, major launches |
| Without team buy-in | Chaos without consent = sabotage |
| Brand new system (week 1) | Build basic resilience first, then test it |
| One-person team with no backup | Who fixes it if the experiment goes wrong? |

---

## Getting Started: Your First Chaos Experiment

```
┌─────────────────────────────────────────────────────────────────┐
│        CHAOS ENGINEERING STARTER GUIDE                           │
│                                                                 │
│  Week 1: OBSERVE                                               │
│    → Set up monitoring (Prometheus + Grafana)                  │
│    → Define steady-state metrics                               │
│    → Document current architecture & dependencies              │
│                                                                 │
│  Week 2: HYPOTHESIZE (in staging)                              │
│    → "If pod X dies, K8s should restart in <30s"              │
│    → Run experiment in staging                                 │
│    → Did hypothesis hold?                                      │
│                                                                 │
│  Week 3: SMALL PRODUCTION EXPERIMENT                            │
│    → Kill 1 pod out of 10 (tiny blast radius)                 │
│    → During low-traffic hours                                  │
│    → With manual kill switch ready                            │
│                                                                 │
│  Week 4: EXPAND                                                │
│    → Network latency injection                                 │
│    → Service dependency failure                                │
│    → Schedule regular experiments                              │
│                                                                 │
│  Month 2+: AUTOMATE                                            │
│    → Chaos Mesh / LitmusChaos running on schedule             │
│    → Experiments in CI/CD pipeline                            │
│    → Game days for team training                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

- **Chaos Engineering is NOT about breaking things randomly** — it's about running controlled scientific experiments to discover weaknesses.
- **Start small, expand gradually** — begin with one pod in staging, progress to region-level failures in production.
- **Always have an abort condition** — automatic stops when error rate exceeds threshold.
- **Chaos reveals truths** that design documents hide — you don't KNOW your fallbacks work until you test them.
- **Netflix's Chaos Monkey forced resilience** — when failure is constant, engineers build for it from day one.
- **Game days are invaluable** — scheduled chaos experiments where the whole team participates and learns.
- **Fix what you find** — chaos engineering without follow-up is just expensive vandalism.

---

## What's Next?

Congratulations! You've completed Part 12: Resilience & Fault Tolerance. You now understand the full resilience toolkit:

1. ✅ Timeouts — Don't wait forever
2. ✅ Retries — Try again, smartly
3. ✅ Circuit Breakers — Stop calling dead services
4. ✅ Bulkheads — Isolate failures
5. ✅ Fallbacks — Always have a Plan B
6. ✅ Backpressure — Slow down when overwhelmed
7. ✅ Idempotency — Safe to retry
8. ✅ Graceful Degradation — Keep core features alive
9. ✅ Chaos Engineering — Test it all in production

Next up: **Part 13: Distributed Systems — The Hard Stuff**, starting with [What is a Distributed System?](../13-distributed-systems/01-what-is-distributed-system.md) where we explore why distributed systems are fundamentally harder than single-machine systems, and the theoretical limits that constrain every architect's choices.
