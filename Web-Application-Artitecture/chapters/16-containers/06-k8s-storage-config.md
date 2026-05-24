# Kubernetes Storage, ConfigMaps & Secrets

> **What you'll learn**: How to persist data in Kubernetes (Volumes, PersistentVolumes, StorageClasses), manage application configuration (ConfigMaps), and handle sensitive data (Secrets) — the three pillars of stateful applications in K8s.

---

## Real-Life Analogy: An Apartment Building

Think of a Kubernetes cluster like an **apartment building**:

- **Pods** are tenants — they move in and out frequently
- **Ephemeral storage** is like a whiteboard in the apartment — erased when the tenant moves out
- **PersistentVolume** is like a **storage locker** in the basement — it stays even if the tenant leaves
- **ConfigMap** is like the **building's notice board** — settings everyone can read (Wi-Fi password, pool hours)
- **Secret** is like a **personal safe** in the apartment — only the tenant with the key can access it

```
Pod dies and restarts:

Ephemeral:  Data = GONE (whiteboard erased)
PV-backed:  Data = SAFE (storage locker persists)
ConfigMap:  Settings = Available to new pod
Secret:     Credentials = Available to new pod
```

---

## Part 1: Storage in Kubernetes

### The Problem: Containers Are Ephemeral

```
┌──────── Pod Lifecycle ────────┐
│                                │
│  Pod starts → writes to /data  │
│       │                        │
│       ▼                        │
│  Pod crashes / restarts        │
│       │                        │
│       ▼                        │
│  /data is EMPTY again! 😱     │
│                                │
└────────────────────────────────┘

This is fine for stateless apps (web servers).
This is TERRIBLE for databases, file storage, ML models.
```

### Volume Types Hierarchy

```
┌────────────────────────────────────────────────────────────────────┐
│                     KUBERNETES STORAGE                              │
│                                                                     │
│  ┌─── Ephemeral ──────────────────────────────────────────────┐   │
│  │                                                             │   │
│  │  emptyDir     → Created when pod starts, deleted when dies  │   │
│  │  configMap    → Mount config files into pods                 │   │
│  │  secret       → Mount secret data into pods                  │   │
│  │  downwardAPI  → Expose pod metadata as files                 │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─── Persistent ─────────────────────────────────────────────┐   │
│  │                                                             │   │
│  │  PersistentVolume (PV)       → Actual storage resource      │   │
│  │  PersistentVolumeClaim (PVC) → Request for storage          │   │
│  │  StorageClass                → Template for dynamic PVs     │   │
│  │                                                             │   │
│  │  Backed by:                                                  │   │
│  │    AWS EBS, GCP PD, Azure Disk, NFS, Ceph, local disk      │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

### emptyDir — Temporary Shared Storage

Exists for the **lifetime of the pod** (shared between containers in same pod):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-processor
spec:
  containers:
    # App writes logs
    - name: webapp
      image: myapp:v1
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app

    # Sidecar reads and ships logs
    - name: log-shipper
      image: fluent-bit:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
          readOnly: true

  volumes:
    - name: shared-logs
      emptyDir:
        medium: Memory    # Use RAM (tmpfs) for speed
        sizeLimit: 100Mi  # Cap at 100 MB
```

```
┌─────────────── Pod ─────────────────────┐
│                                          │
│  ┌─── webapp ────┐  ┌── log-shipper ──┐│
│  │ writes to     │  │ reads from      ││
│  │ /var/log/app  │  │ /var/log/app    ││
│  └───────┬───────┘  └───────┬─────────┘│
│          │                   │          │
│          └─────────┬─────────┘          │
│                    ▼                    │
│          ┌─────────────────┐            │
│          │  emptyDir volume │            │
│          │  (shared space)  │            │
│          └─────────────────┘            │
│                                          │
│  Pod deleted → emptyDir is GONE         │
└──────────────────────────────────────────┘
```

### PersistentVolumes (PV) & PersistentVolumeClaims (PVC)

The **PV/PVC model** separates storage provisioning from consumption:

```
┌───── Cluster Admin ─────┐        ┌───── Developer ──────────────┐
│                          │        │                               │
│  "I provision 100 GB    │        │  "I need 20 GB of fast       │
│   of SSD storage"       │        │   storage for my database"   │
│                          │        │                               │
│  Creates: PersistentVolume│        │  Creates: PersistentVolumeClaim│
│  (PV)                    │        │  (PVC)                        │
└────────────┬─────────────┘        └──────────────┬────────────────┘
             │                                      │
             │         ┌────── K8s ──────┐         │
             └────────▶│                  │◀────────┘
                       │  Binds PVC to PV │
                       │  (matching size, │
                       │   access mode,   │
                       │   storage class) │
                       └──────────────────┘
```

```yaml
# 1. PersistentVolume — the actual storage (often auto-created)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce       # One node can mount read-write
  persistentVolumeReclaimPolicy: Retain  # Keep data after PVC deleted
  storageClassName: fast-ssd
  awsElasticBlockStore:   # AWS EBS volume
    volumeID: vol-0abc123def
    fsType: ext4

---
# 2. PersistentVolumeClaim — request for storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast-ssd

---
# 3. Pod uses the PVC
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
    - name: postgres
      image: postgres:15
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: postgres-data    # References the PVC
```

### StorageClass — Dynamic Provisioning

Instead of manually creating PVs, let Kubernetes **automatically provision** them:

```yaml
# StorageClass — template for auto-provisioning
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com    # AWS EBS CSI driver
parameters:
  type: gp3                      # SSD type
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete            # Delete EBS volume when PVC is deleted
volumeBindingMode: WaitForFirstConsumer  # Don't provision until pod is scheduled
allowVolumeExpansion: true       # Allow growing the volume later

---
# StorageClass for cheaper workloads
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
reclaimPolicy: Delete
```

```
Dynamic Provisioning Flow:
                                                              
Developer creates PVC ──▶ K8s checks StorageClass ──▶ Calls provisioner
(50Gi, fast-ssd)           (fast-ssd → ebs.csi)       (creates EBS volume)
                                                              │
                                                              ▼
Pod starts ◀── mounts ◀── PVC bound ◀── PV auto-created ◀── EBS ready
```

### Access Modes

| Mode | Abbreviation | Description |
|------|-------------|-------------|
| ReadWriteOnce | RWO | Single node read-write (most block storage) |
| ReadOnlyMany | ROX | Many nodes read-only |
| ReadWriteMany | RWX | Many nodes read-write (NFS, EFS) |
| ReadWriteOncePod | RWOP | Single pod read-write (K8s 1.22+) |

### StatefulSet — For Databases and Stateful Apps

```yaml
# StatefulSet with volumeClaimTemplates (auto-creates one PVC per replica)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres    # Headless service for stable DNS
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
  volumeClaimTemplates:     # Each replica gets its OWN PVC!
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
```

```
StatefulSet creates:
  postgres-0 → PVC: data-postgres-0 → PV: pv-abc123 (100Gi EBS)
  postgres-1 → PVC: data-postgres-1 → PV: pv-def456 (100Gi EBS)
  postgres-2 → PVC: data-postgres-2 → PV: pv-ghi789 (100Gi EBS)

Each replica has its OWN persistent storage!
Delete postgres-1 → PVC data-postgres-1 still exists → new postgres-1 gets same data
```

---

## Part 2: ConfigMaps — Application Configuration

### The Problem

Hardcoding configuration in images means rebuilding for every environment:

```
❌ Bad: Config baked into image
   image: webapp:v1-dev    (DB_HOST=dev-db)
   image: webapp:v1-staging (DB_HOST=staging-db)
   image: webapp:v1-prod   (DB_HOST=prod-db)

✅ Good: Same image, different config
   image: webapp:v1 + ConfigMap(dev)
   image: webapp:v1 + ConfigMap(staging)
   image: webapp:v1 + ConfigMap(prod)
```

### Creating ConfigMaps

```yaml
# configmap.yaml — Application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  # Simple key-value pairs
  DB_HOST: "postgres.databases.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "myapp"
  REDIS_HOST: "redis.caching.svc.cluster.local"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  
  # Entire config file
  application.yml: |
    server:
      port: 8080
      shutdown: graceful
    spring:
      datasource:
        url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
        hikari:
          maximum-pool-size: 20
    logging:
      level:
        root: INFO
        com.mycompany: DEBUG
  
  # Nginx configuration file
  nginx.conf: |
    upstream backend {
        server webapp:8080;
    }
    server {
        listen 80;
        location / {
            proxy_pass http://backend;
        }
    }
```

### Using ConfigMaps in Pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: myapp:v2.1.0
          
          # Method 1: Individual environment variables
          env:
            - name: DATABASE_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_HOST
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: LOG_LEVEL
          
          # Method 2: All keys as environment variables
          envFrom:
            - configMapRef:
                name: app-config
                # All keys become env vars: DB_HOST, DB_PORT, etc.
          
          # Method 3: Mount as files
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
              readOnly: true
      
      volumes:
        - name: config-volume
          configMap:
            name: app-config
            items:
              - key: application.yml
                path: application.yml   # Mounted at /etc/config/application.yml
        - name: nginx-config
          configMap:
            name: app-config
            items:
              - key: nginx.conf
                path: default.conf
```

### Live Config Reload

```
ConfigMap updated → Mounted files update (within ~1 minute)
                 → Env vars do NOT update (requires pod restart)

┌─────────────────────────────────────────────────────────────┐
│  Config change strategy:                                     │
│                                                              │
│  Option 1: Mounted files + file watcher in app              │
│    ConfigMap updated → file changes → app detects → reloads │
│                                                              │
│  Option 2: Rolling restart                                   │
│    kubectl rollout restart deployment/webapp                 │
│                                                              │
│  Option 3: Reloader (open source tool)                      │
│    Watches ConfigMap changes → auto-triggers rolling restart │
└─────────────────────────────────────────────────────────────┘
```

---

## Part 3: Secrets — Sensitive Data

### Why Not Just Use ConfigMaps for Everything?

```
ConfigMap:                          Secret:
- Stored in plain text in etcd      - Base64 encoded (not encrypted by default!)
- Visible in `kubectl describe`     - Hidden in `kubectl describe`
- No special access controls        - Can restrict with RBAC
- For: URLs, ports, feature flags   - For: passwords, API keys, TLS certs
                                    - CAN be encrypted at rest (EncryptionConfig)
```

> ⚠️ **Important**: Kubernetes Secrets are base64-encoded, NOT encrypted by default. You MUST enable encryption at rest or use external secret managers for production.

### Creating Secrets

```bash
# From literal values
kubectl create secret generic db-credentials \
  --from-literal=username=appuser \
  --from-literal=password='S3cur3P@ss!'

# From files
kubectl create secret generic tls-cert \
  --from-file=cert.pem=./tls/server.crt \
  --from-file=key.pem=./tls/server.key

# TLS secret (special type)
kubectl create secret tls webapp-tls \
  --cert=./tls/server.crt \
  --key=./tls/server.key
```

```yaml
# secret.yaml — Declarative (values must be base64-encoded)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  username: YXBwdXNlcg==        # echo -n "appuser" | base64
  password: UzNjdXIzUEBzcyE=   # echo -n "S3cur3P@ss!" | base64

---
# stringData: K8s will base64-encode it for you
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
type: Opaque
stringData:                      # Plain text (K8s encodes automatically)
  stripe-key: "sk_live_abc123xyz"
  sendgrid-key: "SG.abcdef..."
```

### Using Secrets in Pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  template:
    spec:
      containers:
        - name: webapp
          image: myapp:v2
          env:
            # Individual secret key
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            - name: STRIPE_KEY
              valueFrom:
                secretKeyRef:
                  name: api-keys
                  key: stripe-key
          
          # Mount secret as file (for TLS certs, config files)
          volumeMounts:
            - name: tls-certs
              mountPath: /etc/tls
              readOnly: true
      
      volumes:
        - name: tls-certs
          secret:
            secretName: webapp-tls
            defaultMode: 0400    # Read-only by owner
```

### External Secret Managers (Production Best Practice)

```
┌─────────────────────────────────────────────────────────────────┐
│              PRODUCTION SECRETS ARCHITECTURE                      │
│                                                                   │
│  ┌──── External Vault ────┐                                     │
│  │                         │                                     │
│  │  AWS Secrets Manager    │                                     │
│  │  HashiCorp Vault        │                                     │
│  │  GCP Secret Manager     │                                     │
│  │  Azure Key Vault        │                                     │
│  └───────────┬─────────────┘                                     │
│              │                                                    │
│              ▼                                                    │
│  ┌────────────────────────┐     ┌──────────────────────┐        │
│  │  External Secrets      │────▶│  K8s Secret          │        │
│  │  Operator (ESO)        │     │  (auto-synced)       │        │
│  │                        │     │                      │        │
│  │  Syncs vault → K8s     │     │  Referenced by Pods  │        │
│  │  Auto-rotates          │     │                      │        │
│  └────────────────────────┘     └──────────────────────┘        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

```yaml
# External Secrets Operator — sync from AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h           # Check for updates every hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials        # Creates K8s Secret with this name
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: production/database
        property: username
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
```

---

## Code Examples

### Python — Reading Config and Secrets in K8s

```python
import os
from pathlib import Path

class KubernetesConfig:
    """Read configuration from K8s ConfigMaps and Secrets"""
    
    def __init__(self):
        # Environment variables (from ConfigMap/Secret envFrom or env)
        self.db_host = os.environ.get('DB_HOST', 'localhost')
        self.db_port = int(os.environ.get('DB_PORT', 5432))
        self.db_name = os.environ.get('DB_NAME', 'myapp')
        self.log_level = os.environ.get('LOG_LEVEL', 'info')
        
        # Secret from environment variable
        self.db_password = os.environ.get('DB_PASSWORD')
        
        # Secret from mounted file (more secure — not in env dump)
        self.tls_cert = self._read_file('/etc/tls/tls.crt')
        self.tls_key = self._read_file('/etc/tls/tls.key')
        
        # Config from mounted ConfigMap file
        self.app_config = self._read_file('/etc/config/application.yml')
    
    def _read_file(self, path):
        """Read a mounted file (ConfigMap or Secret)"""
        try:
            return Path(path).read_text().strip()
        except FileNotFoundError:
            return None
    
    @property
    def database_url(self):
        return (f"postgresql://{self.db_host}:{self.db_port}"
                f"/{self.db_name}?password={self.db_password}")

# Usage
config = KubernetesConfig()
print(f"Connecting to: {config.db_host}:{config.db_port}/{config.db_name}")
print(f"Log level: {config.log_level}")
print(f"TLS configured: {config.tls_cert is not None}")
```

### Java — Spring Boot with K8s ConfigMaps

```java
// Spring Boot automatically reads from ConfigMaps when using
// spring-cloud-kubernetes (or environment variables)

@Configuration
@ConfigurationProperties(prefix = "app")
public class AppConfig {
    
    private String dbHost;
    private int dbPort;
    private String dbName;
    private String logLevel;
    private int maxConnections;
    
    // These map to ConfigMap keys:
    // app.db-host → DB_HOST env var from ConfigMap
    // app.db-port → DB_PORT env var from ConfigMap
    
    // Getters and setters...
    public String getDbHost() { return dbHost; }
    public void setDbHost(String dbHost) { this.dbHost = dbHost; }
    public int getDbPort() { return dbPort; }
    public void setDbPort(int dbPort) { this.dbPort = dbPort; }
}

// Reading secrets from mounted files (preferred for security)
@Component
public class SecretReader {
    
    @Value("${DB_PASSWORD:}")
    private String dbPasswordFromEnv;
    
    // Read from mounted secret file
    public String readSecretFromFile(String path) {
        try {
            return Files.readString(Path.of(path)).trim();
        } catch (IOException e) {
            throw new RuntimeException("Cannot read secret: " + path, e);
        }
    }
    
    @PostConstruct
    public void init() {
        // Prefer file-based secrets (not visible in /proc/environ)
        String tlsCert = readSecretFromFile("/etc/tls/tls.crt");
        System.out.println("TLS cert loaded: " + (tlsCert != null));
    }
}
```

---

## Infrastructure Example: Complete Stateful Application

```yaml
# Complete PostgreSQL deployment with storage, config, and secrets
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  POSTGRES_DB: "myapp"
  PGDATA: "/var/lib/postgresql/data/pgdata"
  # Custom PostgreSQL configuration
  postgresql.conf: |
    max_connections = 200
    shared_buffers = 2GB
    effective_cache_size = 6GB
    maintenance_work_mem = 512MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 1.1
    work_mem = 10MB

---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  POSTGRES_USER: "appuser"
  POSTGRES_PASSWORD: "ChangeMeInProduction!"
  REPLICATION_PASSWORD: "ReplicaPass123!"

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          envFrom:
            - configMapRef:
                name: postgres-config
            - secretRef:
                name: postgres-secret
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: custom-config
              mountPath: /etc/postgresql/postgresql.conf
              subPath: postgresql.conf
          resources:
            requests:
              cpu: "500m"
              memory: "2Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "appuser"]
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "appuser"]
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: custom-config
          configMap:
            name: postgres-config
            items:
              - key: postgresql.conf
                path: postgresql.conf
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi

---
# Headless service for StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

---

## Real-World Example: How Airbnb Manages Configuration

```
┌────────────── Airbnb's Configuration System ──────────────────────┐
│                                                                     │
│  Challenge: 1000+ microservices, each with different configs       │
│                                                                     │
│  Solution Stack:                                                   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Git Repository (source of truth)                         │     │
│  │  ├── services/                                            │     │
│  │  │   ├── payments/config.yaml                             │     │
│  │  │   ├── search/config.yaml                               │     │
│  │  │   └── booking/config.yaml                              │     │
│  │  └── environments/                                        │     │
│  │      ├── dev.yaml                                         │     │
│  │      ├── staging.yaml                                     │     │
│  │      └── prod.yaml                                        │     │
│  └─────────────────────┬────────────────────────────────────┘     │
│                         │ CI/CD                                     │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Config Controller (Custom K8s operator)                  │     │
│  │  • Generates ConfigMaps from Git                          │     │
│  │  • Merges base + environment overlays                     │     │
│  │  • Triggers rolling restarts on change                    │     │
│  └─────────────────────┬────────────────────────────────────┘     │
│                         │                                           │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  K8s ConfigMaps + Secrets (per service, per environment)  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                     │
│  Secrets: Vault → External Secrets Operator → K8s Secrets         │
│  Rotation: Automatic every 90 days                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes / Pitfalls

| Mistake | Problem | Solution |
|---------|---------|----------|
| Storing secrets in ConfigMaps | Plain text, visible everywhere | Use Secrets + encryption at rest |
| Not enabling etcd encryption | Secrets stored in plain text in etcd | Enable `EncryptionConfiguration` |
| Using default StorageClass | May provision slow/expensive storage | Define explicit StorageClasses |
| Not setting `reclaimPolicy: Retain` | PV deleted when PVC is deleted | Use Retain for databases |
| ConfigMap as env vars + expecting live reload | Env vars don't update | Use file mounts + file watchers |
| Committing Secrets YAML to Git | Secrets exposed in version control | Use External Secrets Operator or Sealed Secrets |
| No resource limits on PVCs | Storage costs spiral | Set quotas per namespace |
| Using ReadWriteMany without need | Limited provisioner support, complex | Use ReadWriteOnce unless multiple pods need write |

---

## When to Use / When NOT to Use

### Storage Decision Matrix

```
┌────────────────────────────────────────────────────────────────┐
│  What you need                        │  Solution               │
├───────────────────────────────────────┼─────────────────────────┤
│  Temp space shared between containers │  emptyDir               │
│  Cache that's OK to lose              │  emptyDir (Memory)      │
│  Database data (single writer)        │  PVC + RWO + StatefulSet│
│  Shared files (multiple readers)      │  PVC + ROX or RWX (NFS) │
│  Log aggregation across pods          │  emptyDir + sidecar     │
│  ML model files (read-only)           │  PVC + ROX              │
│  Config files                         │  ConfigMap volume mount  │
│  TLS certificates                     │  Secret volume mount     │
└───────────────────────────────────────┴─────────────────────────┘
```

### Secrets Management Decision

| Approach | Security | Complexity | Best For |
|----------|----------|-----------|----------|
| K8s Secrets (default) | Low (base64 only) | Simple | Development |
| K8s Secrets + etcd encryption | Medium | Moderate | Small production |
| External Secrets Operator + Vault | High | Complex | Enterprise production |
| Sealed Secrets (Bitnami) | High | Moderate | GitOps workflows |

---

## Key Takeaways

1. **Pods are ephemeral** — always use PersistentVolumes for data that must survive restarts
2. **StorageClasses enable dynamic provisioning** — K8s auto-creates volumes on demand from cloud providers
3. **StatefulSets + volumeClaimTemplates** give each replica its own dedicated persistent storage
4. **ConfigMaps separate config from code** — same image, different config per environment
5. **Secrets are base64-encoded, NOT encrypted** by default — enable encryption at rest and use external vaults for production
6. **Mount secrets as files** rather than environment variables (more secure — not visible in `/proc/environ`)
7. **Never commit Secret manifests to Git** — use External Secrets Operator, Sealed Secrets, or Vault

---

## What's Next?

Managing dozens of YAML files gets complex. Let's learn how Helm Charts package and template Kubernetes manifests in [Chapter 16.7: Helm Charts](./07-helm-charts.md).
