# Kubernetes Architecture — Pods, Services, Deployments

> **What you'll learn**: How Kubernetes orchestrates containers at scale — its architecture, core objects (Pods, Services, Deployments, ReplicaSets), and how it keeps your applications running reliably across a cluster of machines.

---

## Real-Life Analogy: Kubernetes is Like a Ship Fleet Manager

Imagine you run a **shipping company** with hundreds of cargo ships (servers). You have thousands of containers (literally!) that need to be:

- **Placed** on the right ships (scheduling)
- **Monitored** — if a container falls overboard, put a new one on another ship (self-healing)
- **Scaled** — during holiday season, add more containers (auto-scaling)
- **Routed** — make sure customers know which port to go to (service discovery)
- **Updated** — swap old containers with new ones without stopping operations (rolling updates)

You can't do all this manually. You need an **automated fleet management system**. That's Kubernetes.

```
You (Developer): "I need 5 copies of my web app running at all times"

Kubernetes: "Got it. I'll:
  1. Find servers with enough CPU/RAM
  2. Start 5 containers
  3. Monitor them 24/7
  4. Restart any that crash
  5. Replace servers that die
  6. Route traffic to healthy ones
  Done. You go home."
```

> **The name "Kubernetes"** comes from Greek, meaning "helmsman" or "pilot" — the person who steers a ship. Often abbreviated as **K8s** (K + 8 letters + s).

---

## Why Kubernetes? The Problems It Solves

```
Without Kubernetes:                    With Kubernetes:
┌──────────────────────────┐          ┌──────────────────────────┐
│ • Manually SSH into       │          │ • Declare desired state   │
│   servers to deploy       │          │ • K8s makes it happen     │
│ • Manually restart failed │          │ • Auto-restarts failures  │
│   containers              │          │ • Auto-scales on demand   │
│ • Manually balance load   │          │ • Built-in load balancing │
│ • Manually scale up/down  │          │ • Rolling updates         │
│ • Write custom scripts    │          │ • Self-healing            │
│   for everything          │          │ • Works on any cloud      │
└──────────────────────────┘          └──────────────────────────┘

Developer effort: HIGH                 Developer effort: LOW
Reliability: LOW                       Reliability: HIGH
```

---

## Kubernetes Architecture — The Big Picture

```
┌─────────────────────────────────── KUBERNETES CLUSTER ───────────────────────────────────┐
│                                                                                           │
│  ┌─────────────── Control Plane (The Brain) ──────────────────────────────────────┐      │
│  │                                                                                 │      │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │      │
│  │  │  API Server  │  │  Scheduler   │  │  Controller  │  │      etcd        │   │      │
│  │  │  (kube-api)  │  │              │  │   Manager    │  │  (key-value DB)  │   │      │
│  │  │              │  │  "Where to   │  │              │  │                  │   │      │
│  │  │  Front door  │  │   place      │  │  "Ensure     │  │  "Source of     │   │      │
│  │  │  for ALL     │  │   pods?"     │  │   desired    │  │   truth for     │   │      │
│  │  │  requests    │  │              │  │   state"     │  │   cluster state" │   │      │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘   │      │
│  │         ▲                                                                       │      │
│  └─────────┼───────────────────────────────────────────────────────────────────────┘      │
│            │ kubectl / API calls                                                           │
│            │                                                                              │
│  ┌─────────┼──────────── Worker Nodes (The Muscles) ──────────────────────────────┐      │
│  │         │                                                                       │      │
│  │  ┌──────┴────── Node 1 ──────────┐  ┌──────────── Node 2 ──────────────┐      │      │
│  │  │                                │  │                                   │      │      │
│  │  │  ┌────────┐  ┌────────┐       │  │  ┌────────┐  ┌────────┐          │      │      │
│  │  │  │ Pod A  │  │ Pod B  │       │  │  │ Pod C  │  │ Pod D  │          │      │      │
│  │  │  │┌──────┐│  │┌──────┐│       │  │  │┌──────┐│  │┌──────┐│          │      │      │
│  │  │  ││ App  ││  ││ API  ││       │  │  ││ App  ││  ││Worker││          │      │      │
│  │  │  │└──────┘│  │└──────┘│       │  │  │└──────┘│  │└──────┘│          │      │      │
│  │  │  └────────┘  └────────┘       │  │  └────────┘  └────────┘          │      │      │
│  │  │                                │  │                                   │      │      │
│  │  │  ┌─────────┐ ┌──────────────┐ │  │  ┌─────────┐ ┌──────────────┐   │      │      │
│  │  │  │ kubelet │ │ kube-proxy   │ │  │  │ kubelet │ │ kube-proxy   │   │      │      │
│  │  │  │(agent)  │ │(networking)  │ │  │  │(agent)  │ │(networking)  │   │      │      │
│  │  │  └─────────┘ └──────────────┘ │  │  └─────────┘ └──────────────┘   │      │      │
│  │  │                                │  │                                   │      │      │
│  │  │  ┌──────────────────────────┐  │  │  ┌──────────────────────────┐   │      │      │
│  │  │  │  Container Runtime       │  │  │  │  Container Runtime       │   │      │      │
│  │  │  │  (containerd / CRI-O)    │  │  │  │  (containerd / CRI-O)    │   │      │      │
│  │  │  └──────────────────────────┘  │  │  └──────────────────────────┘   │      │      │
│  │  └────────────────────────────────┘  └───────────────────────────────────┘      │      │
│  └─────────────────────────────────────────────────────────────────────────────────┘      │
└───────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Control Plane Components (The Brain)

### 1. API Server (kube-apiserver)

The **single entry point** for all cluster operations. Everything talks to it:

```
kubectl ────────────┐
                    │
Kubernetes Dashboard┼──────▶  API Server ──────▶ etcd
                    │              │
CI/CD pipelines ────┘              │
                              ┌────┴────┐
                              ▼         ▼
                         Scheduler   Controllers
```

- Validates and processes REST requests
- Authenticates and authorizes users
- Serves as the gateway to etcd (nothing else talks to etcd directly)

### 2. etcd — The Cluster's Memory

A distributed key-value store that holds ALL cluster state:

```
etcd stores:
├── /registry/pods/default/webapp-abc123      → Pod spec & status
├── /registry/services/default/webapp-service → Service config
├── /registry/deployments/default/webapp      → Deployment spec
├── /registry/nodes/worker-1                  → Node information
└── /registry/secrets/default/db-password     → Encrypted secrets
```

- Strongly consistent (uses Raft consensus)
- Typically 3 or 5 replicas for high availability
- If etcd dies, the cluster is blind (but running containers keep running)

### 3. Scheduler (kube-scheduler)

Decides **which node** a new Pod should run on:

```
New Pod needs:                    Scheduler evaluates:
- 2 CPU cores                     Node 1: 4 CPU free, 8GB free → ✅ Fits!
- 4 GB RAM                        Node 2: 1 CPU free, 2GB free → ❌ Not enough
- SSD storage                     Node 3: 3 CPU free, 6GB free → ✅ Fits!
- Not on same node as Pod X       
                                  Winner: Node 1 (most resources available)
```

**Scheduling factors:**
- Resource requests/limits (CPU, memory)
- Node affinity/anti-affinity rules
- Taints and tolerations
- Pod topology spread constraints
- Priority and preemption

### 4. Controller Manager (kube-controller-manager)

Runs **control loops** that watch the cluster state and make corrections:

```
Desired State: 3 replicas          Actual State: 2 replicas (one crashed)
       │                                  │
       └──────────┐    ┌──────────────────┘
                  ▼    ▼
          ┌────────────────────┐
          │  ReplicaSet        │
          │  Controller        │
          │                    │
          │  "3 desired,       │
          │   2 running...     │
          │   Creating 1 more!"│
          └────────────────────┘
```

**Key controllers:**
- **ReplicaSet Controller**: Ensures correct number of pod replicas
- **Deployment Controller**: Manages rollouts and rollbacks
- **Node Controller**: Detects when nodes go down
- **Job Controller**: Manages batch jobs
- **Service Controller**: Manages cloud load balancers

---

## Worker Node Components (The Muscles)

### kubelet — The Node Agent

```
┌─────────── Worker Node ───────────┐
│                                    │
│  kubelet:                          │
│  ┌────────────────────────────┐   │
│  │ • Receives Pod specs from  │   │
│  │   API Server               │   │
│  │ • Tells container runtime  │   │
│  │   to start/stop containers │   │
│  │ • Reports Pod health back  │   │
│  │   to API Server            │   │
│  │ • Mounts volumes           │   │
│  │ • Runs liveness probes     │   │
│  └────────────────────────────┘   │
│                                    │
└────────────────────────────────────┘
```

### kube-proxy — Network Rules

Maintains network rules for Pod-to-Pod and Service-to-Pod communication. More details in [Chapter 16.5: Kubernetes Networking](./05-k8s-networking.md).

---

## Core Kubernetes Objects

### 1. Pod — The Smallest Deployable Unit

A Pod is **one or more containers** that share network and storage:

```
┌─────────────── Pod ─────────────────────┐
│                                          │
│  Shared Network Namespace:              │
│  IP: 10.244.1.5                         │
│                                          │
│  ┌─────────────┐  ┌─────────────────┐  │
│  │ Main        │  │ Sidecar         │  │
│  │ Container   │  │ Container       │  │
│  │             │  │                 │  │
│  │ webapp:v2   │  │ log-shipper     │  │
│  │ Port 8080   │  │ Port 9090       │  │
│  └──────┬──────┘  └────────┬────────┘  │
│         │                   │           │
│         └─────────┬─────────┘           │
│                   │                     │
│  ┌────────────────┴───────────────────┐ │
│  │   Shared Volume: /var/log          │ │
│  └────────────────────────────────────┘ │
└──────────────────────────────────────────┘
```

```yaml
# pod.yaml — Simple pod definition
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    app: webapp
    version: v2
spec:
  containers:
    - name: webapp
      image: myregistry.com/webapp:v2.1.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: "250m"        # 0.25 CPU cores
          memory: "256Mi"    # 256 MB RAM
        limits:
          cpu: "500m"        # Max 0.5 CPU
          memory: "512Mi"    # Max 512 MB RAM
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 3
    - name: log-shipper
      image: fluent/fluent-bit:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  volumes:
    - name: logs
      emptyDir: {}
```

> **Important**: You almost NEVER create Pods directly. You use Deployments (which create ReplicaSets, which create Pods).

### 2. ReplicaSet — Running Multiple Copies

```
┌────────── ReplicaSet (replicas: 3) ──────────┐
│                                               │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   │
│  │  Pod 1  │   │  Pod 2  │   │  Pod 3  │   │
│  │ webapp  │   │ webapp  │   │ webapp  │   │
│  │  :v2    │   │  :v2    │   │  :v2    │   │
│  └─────────┘   └─────────┘   └─────────┘   │
│                                               │
│  If Pod 2 crashes → automatically creates     │
│  a new Pod 2' to maintain 3 replicas         │
└───────────────────────────────────────────────┘
```

### 3. Deployment — The Standard Way to Run Apps

A **Deployment** manages ReplicaSets and enables rolling updates:

```
┌────────────── Deployment ──────────────────────────────┐
│                                                         │
│  Strategy: RollingUpdate                                │
│  Replicas: 3                                           │
│                                                         │
│  ┌──── ReplicaSet (v2 - current) ────────────────┐    │
│  │                                                │    │
│  │  ┌───────┐  ┌───────┐  ┌───────┐             │    │
│  │  │Pod v2 │  │Pod v2 │  │Pod v2 │             │    │
│  │  └───────┘  └───────┘  └───────┘             │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  ┌──── ReplicaSet (v1 - previous, scaled to 0) ──┐   │
│  │  (kept for rollback capability)                │   │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

```yaml
# deployment.yaml — Production-ready deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # At most 1 extra pod during update
      maxUnavailable: 0     # Never have fewer than 3 running
  template:
    metadata:
      labels:
        app: webapp
        version: v2
    spec:
      containers:
        - name: webapp
          image: myregistry.com/webapp:v2.1.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: db_host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: db_password
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
      terminationGracePeriodSeconds: 30
```

### Rolling Update in Action

```
Time T0: All v1 pods running
┌───────┐ ┌───────┐ ┌───────┐
│Pod v1 │ │Pod v1 │ │Pod v1 │
└───────┘ └───────┘ └───────┘

Time T1: New v2 pod starting (maxSurge: 1)
┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
│Pod v1 │ │Pod v1 │ │Pod v1 │ │Pod v2 │ ← Starting
└───────┘ └───────┘ └───────┘ └───────┘

Time T2: v2 pod ready, terminating one v1
┌───────┐ ┌───────┐ ┌ ─ ─ ─ ┐ ┌───────┐
│Pod v1 │ │Pod v1 │  Termin.   │Pod v2 │ ✅
└───────┘ └───────┘ └ ─ ─ ─ ┘ └───────┘

Time T3: Another v2 starting
┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
│Pod v1 │ │Pod v1 │ │Pod v2 │ │Pod v2 │ ← Starting
└───────┘ └───────┘ └───────┘ └───────┘

...continues until all pods are v2...

Time T6: Complete — all v2
┌───────┐ ┌───────┐ ┌───────┐
│Pod v2 │ │Pod v2 │ │Pod v2 │  ✅ Zero downtime!
└───────┘ └───────┘ └───────┘
```

### 4. Service — Stable Networking for Pods

Pods come and go (get new IPs each time). A **Service** provides a **stable endpoint**:

```
Problem: Pods have random IPs that change!
Pod webapp-abc: 10.244.1.5 → crashes → new pod: 10.244.2.9

Solution: Service gives a stable IP + DNS name

┌──────────────── Service: webapp-service ─────────────────┐
│  Cluster IP: 10.96.45.12                                  │
│  DNS: webapp-service.default.svc.cluster.local            │
│                                                           │
│  Routes to pods with label: app=webapp                    │
│                                                           │
│        ┌────────────┐                                    │
│        │  10.96.45.12  (stable!)                         │
│        └──────┬─────┘                                    │
│               │                                           │
│      ┌────────┼────────┐                                 │
│      ▼        ▼        ▼                                 │
│  ┌───────┐┌───────┐┌───────┐                            │
│  │Pod    ││Pod    ││Pod    │                            │
│  │10.244.││10.244.││10.244.│                            │
│  │1.5    ││1.6    ││2.3    │                            │
│  └───────┘└───────┘└───────┘                            │
└───────────────────────────────────────────────────────────┘
```

**Service Types:**

```
┌─────────────────────────────────────────────────────────────────┐
│                       SERVICE TYPES                               │
│                                                                   │
│  ClusterIP (default)     NodePort              LoadBalancer       │
│  ─────────────────────   ──────────────        ──────────────    │
│  Internal only           Exposes on each       Cloud LB + NodePort│
│  Accessible within       node's IP at a        External access    │
│  cluster only            static port                              │
│                                                                   │
│  ┌────────┐              ┌────────┐            ┌─── Cloud LB ──┐│
│  │ClusterIP│             │Node:30080│           │  Public IP    ││
│  │10.96.x.x│            │Node:30080│           └───────┬───────┘│
│  └────┬───┘              └────┬───┘                    │        │
│       │                       │                        ▼        │
│       ▼                       ▼                 ┌──────────┐    │
│  ┌────────┐              ┌────────┐            │NodePort   │    │
│  │  Pods  │              │  Pods  │            └─────┬────┘    │
│  └────────┘              └────────┘                  ▼         │
│                                                  ┌────────┐    │
│                                                  │  Pods  │    │
│                                                  └────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

```yaml
# service.yaml — ClusterIP (internal) service
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: ClusterIP
  selector:
    app: webapp      # Routes to pods with this label
  ports:
    - port: 80         # Service port (what clients connect to)
      targetPort: 8080  # Pod port (where app listens)
      protocol: TCP

---
# LoadBalancer service (external access)
apiVersion: v1
kind: Service
metadata:
  name: webapp-public
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
    - port: 443
      targetPort: 8080
```

---

## Code Examples

### Python — Interacting with Kubernetes API

```python
from kubernetes import client, config

# Load kubeconfig (local dev) or in-cluster config (running in K8s)
try:
    config.load_incluster_config()  # Running inside K8s
except:
    config.load_kube_config()       # Local development

# Create API clients
apps_v1 = client.AppsV1Api()
core_v1 = client.CoreV1Api()

# Create a deployment programmatically
def create_deployment(name, image, replicas=3):
    container = client.V1Container(
        name=name,
        image=image,
        ports=[client.V1ContainerPort(container_port=8080)],
        resources=client.V1ResourceRequirements(
            requests={"cpu": "250m", "memory": "256Mi"},
            limits={"cpu": "500m", "memory": "512Mi"}
        )
    )
    
    template = client.V1PodTemplateSpec(
        metadata=client.V1ObjectMeta(labels={"app": name}),
        spec=client.V1PodSpec(containers=[container])
    )
    
    spec = client.V1DeploymentSpec(
        replicas=replicas,
        selector=client.V1LabelSelector(match_labels={"app": name}),
        template=template
    )
    
    deployment = client.V1Deployment(
        metadata=client.V1ObjectMeta(name=name),
        spec=spec
    )
    
    apps_v1.create_namespaced_deployment(namespace="default", body=deployment)
    print(f"Deployment '{name}' created with {replicas} replicas")

# Scale a deployment
def scale_deployment(name, replicas):
    apps_v1.patch_namespaced_deployment_scale(
        name=name,
        namespace="default",
        body={"spec": {"replicas": replicas}}
    )
    print(f"Scaled '{name}' to {replicas} replicas")

# List all pods and their status
def list_pods():
    pods = core_v1.list_namespaced_pod(namespace="default")
    for pod in pods.items:
        print(f"  {pod.metadata.name}: {pod.status.phase} "
              f"(Node: {pod.spec.node_name})")

create_deployment("webapp", "myregistry.com/webapp:v2.1.0", replicas=3)
list_pods()
```

### Java — Kubernetes Client (Fabric8)

```java
import io.fabric8.kubernetes.client.*;
import io.fabric8.kubernetes.api.model.*;
import io.fabric8.kubernetes.api.model.apps.*;

public class K8sDeployment {
    
    public static void main(String[] args) {
        // Auto-detects in-cluster or kubeconfig
        try (KubernetesClient client = new KubernetesClientBuilder().build()) {
            
            // Create a Deployment
            Deployment deployment = new DeploymentBuilder()
                .withNewMetadata()
                    .withName("webapp")
                    .addToLabels("app", "webapp")
                .endMetadata()
                .withNewSpec()
                    .withReplicas(3)
                    .withNewSelector()
                        .addToMatchLabels("app", "webapp")
                    .endSelector()
                    .withNewTemplate()
                        .withNewMetadata()
                            .addToLabels("app", "webapp")
                        .endMetadata()
                        .withNewSpec()
                            .addNewContainer()
                                .withName("webapp")
                                .withImage("myregistry.com/webapp:v2.1.0")
                                .addNewPort().withContainerPort(8080).endPort()
                                .withNewResources()
                                    .addToRequests("cpu", new Quantity("250m"))
                                    .addToRequests("memory", new Quantity("256Mi"))
                                    .addToLimits("cpu", new Quantity("500m"))
                                    .addToLimits("memory", new Quantity("512Mi"))
                                .endResources()
                            .endContainer()
                        .endSpec()
                    .endTemplate()
                .endSpec()
                .build();
            
            client.apps().deployments()
                .inNamespace("default")
                .resource(deployment)
                .create();
            
            System.out.println("Deployment created!");
            
            // Scale to 5 replicas
            client.apps().deployments()
                .inNamespace("default")
                .withName("webapp")
                .scale(5);
            
            System.out.println("Scaled to 5 replicas");
            
            // Watch for pod events
            client.pods().inNamespace("default")
                .withLabel("app", "webapp")
                .watch(new Watcher<Pod>() {
                    @Override
                    public void eventReceived(Action action, Pod pod) {
                        System.out.printf("Pod %s: %s (%s)%n",
                            pod.getMetadata().getName(),
                            action,
                            pod.getStatus().getPhase());
                    }
                    @Override
                    public void onClose(WatcherException e) {}
                });
        }
    }
}
```

---

## kubectl — The Kubernetes CLI

```bash
# === Cluster Info ===
kubectl cluster-info
kubectl get nodes -o wide

# === Deployments ===
kubectl apply -f deployment.yaml           # Create/update from file
kubectl get deployments                     # List deployments
kubectl describe deployment webapp          # Detailed info
kubectl rollout status deployment/webapp    # Watch rollout progress
kubectl rollout history deployment/webapp   # View revision history
kubectl rollout undo deployment/webapp      # Rollback to previous version
kubectl scale deployment webapp --replicas=5  # Scale manually

# === Pods ===
kubectl get pods -o wide                    # List pods with node info
kubectl logs webapp-abc123 -f               # Stream pod logs
kubectl logs webapp-abc123 -c sidecar       # Specific container logs
kubectl exec -it webapp-abc123 -- /bin/sh   # Shell into pod
kubectl top pods                            # Resource usage

# === Services ===
kubectl get services
kubectl expose deployment webapp --port=80 --target-port=8080 --type=LoadBalancer

# === Debugging ===
kubectl describe pod webapp-abc123          # Events, conditions
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl port-forward svc/webapp-service 8080:80  # Local access
```

---

## Real-World Example: How Spotify Runs on Kubernetes

Spotify migrated from their custom orchestrator (Helios) to Kubernetes:

```
┌──────────────── Spotify on Kubernetes ────────────────────────┐
│                                                                │
│  Clusters: 100+ Kubernetes clusters                           │
│  Pods: 100,000+ running at any time                          │
│  Services: 2,000+ microservices                               │
│  Deployments: 10,000+ per week                               │
│                                                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Playlist │  │  Search  │  │  Audio   │  │ Recommend│    │
│  │ Service  │  │ Service  │  │ Delivery │  │  Engine  │    │
│  │  (200    │  │  (150    │  │  (500    │  │  (300    │    │
│  │  pods)   │  │  pods)   │  │  pods)   │  │  pods)   │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
│        │            │              │              │           │
│        └────────────┴──────────────┴──────────────┘           │
│                           │                                   │
│              ┌────────────┴────────────┐                      │
│              │  Backstage (Developer   │                      │
│              │  Platform by Spotify)   │                      │
│              │  - Deploy via UI/CLI    │                      │
│              │  - Auto-creates K8s     │                      │
│              │    manifests            │                      │
│              └─────────────────────────┘                      │
└───────────────────────────────────────────────────────────────┘
```

**Key decisions:**
- Each team owns their microservice and its Kubernetes deployment
- **Backstage** (their developer portal, now open source) abstracts K8s complexity
- Developers don't write YAML — they use templates
- Auto-scaling handles 200M+ monthly active users

---

## Common Mistakes / Pitfalls

| Mistake | Problem | Solution |
|---------|---------|----------|
| No resource requests/limits | Pods get evicted or starve others | Always set both requests and limits |
| No health checks (probes) | K8s can't detect unhealthy pods | Add liveness + readiness probes |
| Using `latest` image tag | Can't rollback, non-reproducible | Use specific version tags |
| Deploying Pods directly | No self-healing or scaling | Always use Deployments |
| Not setting `terminationGracePeriod` | Requests dropped during shutdown | Set 30s+ and handle SIGTERM |
| Putting everything in `default` namespace | No isolation, hard to manage | Use namespaces per team/env |
| No Pod Disruption Budget | Upgrades take down all pods | Set PDB to maintain availability |
| Skipping readiness probes | Traffic sent before app is ready | Readiness probe = "ready for traffic?" |

---

## When to Use / When NOT to Use

### ✅ Use Kubernetes When:
- You have **10+ services** that need independent deployment and scaling
- You need **high availability** (self-healing, rolling updates)
- You're running across **multiple servers/clouds**
- You have a **dedicated platform team** to manage the cluster
- You need **auto-scaling** based on load

### ❌ Don't Use Kubernetes When:
- You have **1-3 simple services** (use Docker Compose or ECS)
- You're a **small team** without K8s expertise (steep learning curve)
- Your app is a **monolith** that doesn't need orchestration
- You just need a **simple deployment** (use Heroku, Railway, Fly.io)
- Cost is critical — K8s control plane has overhead (use serverless instead)

> **Rule of Thumb**: If you can describe your deployment needs in one sentence, you probably don't need Kubernetes.

---

## Key Takeaways

1. **Kubernetes automates container orchestration** — scheduling, scaling, healing, and updating
2. **Control Plane** (API Server, etcd, Scheduler, Controllers) is the brain; **Worker Nodes** (kubelet, containers) are the muscles
3. **Never create Pods directly** — use Deployments which manage ReplicaSets which manage Pods
4. **Services provide stable networking** — a fixed IP/DNS while pods come and go
5. **Always set resource requests/limits** and health checks (liveness + readiness probes)
6. **Rolling updates** enable zero-downtime deployments; rollbacks are one command away
7. **Kubernetes has a steep learning curve** — start simple, add complexity as needed

---

## What's Next?

Understanding how pods communicate with each other and the outside world is crucial. Let's dive into [Chapter 16.5: Kubernetes Networking & Service Discovery](./05-k8s-networking.md).
