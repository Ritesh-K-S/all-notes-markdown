# Chapter 65 — Disaster Recovery on GCP

---

## Table of Contents

- [Overview](#overview)
- [Part 1: DR Fundamentals (RPO & RTO)](#part-1--dr-fundamentals-rpo--rto)
- [Part 2: DR Patterns Overview](#part-2--dr-patterns-overview)
- [Part 3: Cold Standby Pattern](#part-3--cold-standby-pattern)
- [Part 4: Warm Standby Pattern](#part-4--warm-standby-pattern)
- [Part 5: Hot Standby (Active-Passive)](#part-5--hot-standby-active-passive)
- [Part 6: Active-Active (Multi-Region)](#part-6--active-active-multi-region)
- [Part 7: Compute DR Strategies](#part-7--compute-dr-strategies)
- [Part 8: Database DR Strategies](#part-8--database-dr-strategies)
- [Part 9: Storage DR Strategies](#part-9--storage-dr-strategies)
- [Part 10: Networking & DNS Failover](#part-10--networking--dns-failover)
- [Part 11: GKE Disaster Recovery](#part-11--gke-disaster-recovery)
- [Part 12: Backup Strategies & Testing](#part-12--backup-strategies--testing)
- [Part 13: DR Automation & Runbooks](#part-13--dr-automation--runbooks)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World DR Patterns](#part-15--real-world-dr-patterns)
- [Quick Reference](#quick-reference)
- [What is Disaster Recovery? (Beginner Explanation)](#what-is-disaster-recovery-beginner-explanation)
- [Console Walkthrough: Key DR Operations](#console-walkthrough-key-dr-operations)
- [DR Testing Runbook](#dr-testing-runbook)
- [What's Next?](#whats-next)

---

## Overview

Disaster Recovery (DR) ensures business continuity when failures occur — from zone outages and regional disasters to accidental data deletion and ransomware attacks. GCP's global infrastructure provides the building blocks for every DR tier: cold standby (lowest cost, highest RTO), warm standby, hot standby, and active-active (lowest RTO/RPO, highest cost). This chapter covers DR strategies for every GCP service, failover automation, backup testing, and production-tested DR architectures.

---

## Part 1 — DR Fundamentals (RPO & RTO)

### Key Metrics

```
┌────────────────────────────────────────────────────────────────────┐
│         RPO & RTO                                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  RPO (Recovery Point Objective):                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  "How much data can we afford to lose?"                   │     │
│  │                                                            │     │
│  │  ◄───── data loss window ─────►                          │     │
│  │  Last backup              Disaster              Now      │     │
│  │  ────────●──────────────────●──────────────────●────     │     │
│  │                                                            │     │
│  │  RPO = 0:      Zero data loss (synchronous replication)  │     │
│  │  RPO = 1 hour: Lose up to 1 hour of data                │     │
│  │  RPO = 24 hours: Lose up to 1 day of data (daily backup)│     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  RTO (Recovery Time Objective):                                    │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  "How quickly must we be back online?"                    │     │
│  │                                                            │     │
│  │                  ◄── recovery time ──►                    │     │
│  │  Disaster         Recovery starts      Service restored  │     │
│  │  ────●──────────────●──────────────────●────             │     │
│  │                                                            │     │
│  │  RTO = 0:       Instant failover (active-active)        │     │
│  │  RTO = minutes: Automated failover (hot standby)        │     │
│  │  RTO = hours:   Manual promotion (warm standby)          │     │
│  │  RTO = days:    Rebuild from backups (cold standby)     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Cost vs Recovery:                                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Pattern       │ RPO        │ RTO        │ Cost         │     │
│  │  ────────────── │ ────────── │ ────────── │ ──────────── │     │
│  │  Cold           │ 24 hours   │ Days       │ $            │     │
│  │  Warm           │ Hours      │ Hours      │ $$           │     │
│  │  Hot            │ Minutes    │ Minutes    │ $$$          │     │
│  │  Active-Active  │ 0          │ ~0         │ $$$$         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| DR Feature | GCP | AWS | Azure |
|-----------|-----|-----|-------|
| Global load balancer | Global HTTP(S) LB | Route 53 + ALB | Traffic Manager + App GW |
| DB cross-region | Cloud SQL read replicas | RDS read replicas | Geo-replication |
| Global database | Cloud Spanner (native) | Aurora Global | Cosmos DB |
| Object storage DR | Multi-region / dual-region GCS | S3 Cross-Region Replication | GRS / RA-GRS |
| DNS failover | Cloud DNS + health checks | Route 53 | Traffic Manager |
| Backup | Cloud SQL backups, GCS versioning | AWS Backup | Azure Backup |
| IaC rebuild | Terraform + GCS state | Terraform + S3 state | Terraform + Blob state |

---

## Part 2 — DR Patterns Overview

### Four Tiers of DR

```
┌────────────────────────────────────────────────────────────────────┐
│         DR PATTERNS                                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Tier 1: COLD STANDBY                                              │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Primary (active)      DR region (nothing running)       │     │
│  │  ┌──────────────┐     ┌──────────────┐                  │     │
│  │  │ App + DB     │     │ Backups only │                  │     │
│  │  │ (running)    │     │ (GCS, snapshots)│                │     │
│  │  └──────────────┘     └──────────────┘                  │     │
│  │  Recovery: rebuild from Terraform + restore backups     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Tier 2: WARM STANDBY                                              │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Primary (active)      DR region (minimal running)       │     │
│  │  ┌──────────────┐     ┌──────────────┐                  │     │
│  │  │ App + DB     │     │ DB replica   │                  │     │
│  │  │ (full scale) │     │ (read-only)  │                  │     │
│  │  │              │     │ App (scaled  │                  │     │
│  │  │              │     │  down to min)│                  │     │
│  │  └──────────────┘     └──────────────┘                  │     │
│  │  Recovery: promote DB, scale up app, switch DNS         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Tier 3: HOT STANDBY                                               │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Primary (active)      DR region (ready, not serving)   │     │
│  │  ┌──────────────┐     ┌──────────────┐                  │     │
│  │  │ App + DB     │     │ App + DB     │                  │     │
│  │  │ (full scale) │     │ (full scale, │                  │     │
│  │  │ ◄── traffic  │     │  DB sync,    │                  │     │
│  │  │              │     │  no traffic) │                  │     │
│  │  └──────────────┘     └──────────────┘                  │     │
│  │  Recovery: promote DB, switch DNS (minutes)             │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Tier 4: ACTIVE-ACTIVE                                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Region 1 (active)     Region 2 (active)                │     │
│  │  ┌──────────────┐     ┌──────────────┐                  │     │
│  │  │ App + DB     │ ←──→│ App + DB     │                  │     │
│  │  │ (serving     │     │ (serving     │                  │     │
│  │  │  50% traffic)│     │  50% traffic)│                  │     │
│  │  └──────────────┘     └──────────────┘                  │     │
│  │       └───── Global LB routes to nearest ─────┘         │     │
│  │  Recovery: automatic (LB detects failure, reroutes)     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 3 — Cold Standby Pattern

### Lowest Cost DR

```
┌────────────────────────────────────────────────────────────────────┐
│         COLD STANDBY                                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  What runs in DR region: NOTHING (just stored backups)            │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Primary: us-central1                                     │     │
│  │  ┌──────────────────────────────────────────┐            │     │
│  │  │ Cloud Run / GKE (app)                     │            │     │
│  │  │ Cloud SQL (database)                      │            │     │
│  │  │ GCS (files)                               │            │     │
│  │  └──────────────────────────────────────────┘            │     │
│  │       │ Backups stored in DR region:                     │     │
│  │       │ • Cloud SQL automated backups (cross-region)     │     │
│  │       │ • GCE disk snapshots (multi-regional)            │     │
│  │       │ • GCS dual-region or multi-region bucket         │     │
│  │       │ • Terraform state in GCS (multi-region)          │     │
│  │       │ • Container images in Artifact Registry          │     │
│  │       ▼                                                    │     │
│  │  DR: europe-west1 (nothing running)                      │     │
│  │  ┌──────────────────────────────────────────┐            │     │
│  │  │ GCS: backups, Terraform state, snapshots │            │     │
│  │  │ Artifact Registry: container images      │            │     │
│  │  │                                           │            │     │
│  │  │ Recovery steps:                           │            │     │
│  │  │ 1. terraform apply (create infra)         │            │     │
│  │  │ 2. Restore Cloud SQL from backup          │            │     │
│  │  │ 3. Deploy app from Artifact Registry      │            │     │
│  │  │ 4. Update DNS to DR region                │            │     │
│  │  │ 5. Verify and test                        │            │     │
│  │  └──────────────────────────────────────────┘            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  RPO: 24 hours (daily backups) or less                            │
│  RTO: 4-24 hours (time to rebuild infrastructure)                 │
│  Cost: ~5-10% of primary (just backup storage)                    │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 4 — Warm Standby Pattern

### Balanced Cost and Recovery Time

```
┌────────────────────────────────────────────────────────────────────┐
│         WARM STANDBY                                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  DR region has: DB replica + minimal compute (scaled down)        │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Primary: us-central1              DR: europe-west1      │     │
│  │                                                            │     │
│  │  ┌───────────────┐               ┌───────────────┐      │     │
│  │  │ Cloud Run     │               │ Cloud Run     │      │     │
│  │  │ (auto-scaled) │               │ (min=1, idle) │      │     │
│  │  └───────┬───────┘               └───────┬───────┘      │     │
│  │          │                               │                │     │
│  │  ┌───────▼───────┐    replication ┌──────▼────────┐     │     │
│  │  │ Cloud SQL     │ ──────────────►│ Cloud SQL     │     │     │
│  │  │ Primary       │   async        │ Read Replica  │     │     │
│  │  │ (read/write)  │               │ (read-only)   │     │     │
│  │  └───────────────┘               └───────────────┘      │     │
│  │                                                            │     │
│  │  ┌───────────────┐               ┌───────────────┐      │     │
│  │  │ GCS bucket    │ ──dual-region──│ GCS bucket   │      │     │
│  │  │ (Standard)    │  or STS sync  │ (replicated)  │      │     │
│  │  └───────────────┘               └───────────────┘      │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Failover steps:                                                   │
│  1. Promote Cloud SQL read replica to primary (2-5 min)          │
│  2. Scale up Cloud Run in DR region (1-2 min)                    │
│  3. Update DNS to point to DR region (TTL-dependent)             │
│  4. Verify application functionality                              │
│                                                                      │
│  RPO: minutes to 1 hour (async replication lag)                   │
│  RTO: 30 minutes to 2 hours                                       │
│  Cost: ~25-35% of primary                                         │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 5 — Hot Standby (Active-Passive)

### Fast Recovery

```
┌────────────────────────────────────────────────────────────────────┐
│         HOT STANDBY                                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  DR region has: full infrastructure running, DB synced            │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Primary: us-central1              DR: europe-west1      │     │
│  │                                                            │     │
│  │  Global HTTP(S) Load Balancer                             │     │
│  │  ├── us-central1 backend (100% traffic)                  │     │
│  │  └── europe-west1 backend (0% traffic — ready)          │     │
│  │                                                            │     │
│  │  ┌───────────────┐               ┌───────────────┐      │     │
│  │  │ GKE Autopilot │               │ GKE Autopilot │      │     │
│  │  │ (full scale)  │               │ (full scale)  │      │     │
│  │  │ ◄── 100%      │               │ 0% traffic    │      │     │
│  │  └───────┬───────┘               └───────┬───────┘      │     │
│  │          │                               │                │     │
│  │  ┌───────▼───────┐    sync        ┌──────▼────────┐     │     │
│  │  │ Cloud SQL     │ ──────────────►│ Cloud SQL     │     │     │
│  │  │ Primary (HA)  │   near-zero    │ Cross-region  │     │     │
│  │  │               │   lag          │ Read Replica  │     │     │
│  │  └───────────────┘               └───────────────┘      │     │
│  │                                                            │     │
│  │  ┌───────────────┐               ┌───────────────┐      │     │
│  │  │ Memorystore   │               │ Memorystore   │      │     │
│  │  │ Redis         │               │ Redis         │      │     │
│  │  └───────────────┘               └───────────────┘      │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Failover (automated or manual):                                  │
│  1. Health check detects primary failure                          │
│  2. Promote Cloud SQL replica (2-5 min)                          │
│  3. Update LB backend weights (0% → 100% DR)                    │
│  4. Cache warm-up in DR region                                    │
│                                                                      │
│  RPO: near-zero (sync/async replication)                          │
│  RTO: 5-15 minutes                                                │
│  Cost: ~80-100% of primary (full infrastructure)                  │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 6 — Active-Active (Multi-Region)

### Zero-Downtime Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│         ACTIVE-ACTIVE                                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Both regions serve traffic simultaneously:                       │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  Global HTTP(S) Load Balancer                             │     │
│  │  (routes to nearest healthy region)                      │     │
│  │  ├── us-central1 (50% traffic)                           │     │
│  │  └── europe-west1 (50% traffic)                          │     │
│  │                                                            │     │
│  │  ┌───────────────┐         ┌───────────────┐            │     │
│  │  │ us-central1   │         │ europe-west1  │            │     │
│  │  │               │         │               │            │     │
│  │  │ Cloud Run     │         │ Cloud Run     │            │     │
│  │  │ (auto-scaled) │         │ (auto-scaled) │            │     │
│  │  │       │       │         │       │       │            │     │
│  │  │ Cloud Spanner │ ◄──────►│ Cloud Spanner │            │     │
│  │  │ (multi-region)│ sync    │ (multi-region)│            │     │
│  │  │ 99.999% SLA   │ repl.  │ 99.999% SLA   │            │     │
│  │  │       │       │         │       │       │            │     │
│  │  │ GCS (dual-    │         │ GCS (dual-    │            │     │
│  │  │  region)      │         │  region)      │            │     │
│  │  └───────────────┘         └───────────────┘            │     │
│  │                                                            │     │
│  │  Failover: AUTOMATIC                                     │     │
│  │  LB detects unhealthy backend → routes 100% to healthy  │     │
│  │  No manual intervention required                         │     │
│  │  No DNS changes needed (Global LB handles it)           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  RPO: 0 (synchronous replication with Spanner)                   │
│  RTO: ~0 (automatic failover, no intervention)                   │
│  Cost: 2x primary + Spanner premium                              │
│                                                                      │
│  Best suited for:                                                  │
│  • Payment processing, banking                                    │
│  • Global SaaS with strict SLAs                                   │
│  • Systems where any downtime = significant revenue loss         │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 7 — Compute DR Strategies

### Per-Service DR Approaches

```
┌────────────────────────────────────────────────────────────────────┐
│         COMPUTE DR                                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Cloud Run:                                                        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Deploy same service to multiple regions               │     │
│  │  • Global HTTP(S) LB routes traffic                     │     │
│  │  • Auto-scales independently per region                 │     │
│  │  • Multi-region Artifact Registry for images            │     │
│  │  DR: built-in (just deploy to 2+ regions)              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  GKE:                                                              │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Regional cluster (survives zone failure)              │     │
│  │  • Multi-cluster: deploy to 2+ regions                  │     │
│  │  • Config Sync (GitOps) for consistent deployments     │     │
│  │  • Multi-cluster Ingress with Global LB                 │     │
│  │  • Backup: Velero or GKE Backup for GKE                │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Compute Engine:                                                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Cold: disk snapshots (multi-regional) + Terraform      │     │
│  │  Warm: MIG in DR region (scaled to 0-1)                │     │
│  │  Hot:  MIG in DR region (full scale, no traffic)        │     │
│  │  A-A:  MIG in both regions + Global LB                  │     │
│  │                                                            │     │
│  │  Custom images: stored in multi-region for DR           │     │
│  │  Instance templates: defined in Terraform (any region)  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Cloud Functions:                                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Deploy to multiple regions                            │     │
│  │  • Use Pub/Sub (global) to trigger in any region       │     │
│  │  • Stateless — no data to replicate                    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 8 — Database DR Strategies

### Per-Database DR

```
┌────────────────────────────────────────────────────────────────────┐
│         DATABASE DR                                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Cloud SQL:                                                        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  HA (multi-zone): automatic failover within region      │     │
│  │  Cross-region read replica: async replication           │     │
│  │  → Promote replica on disaster (gcloud sql promote)    │     │
│  │  Automated backups: stored in multi-region              │     │
│  │  Point-in-time recovery: up to 7 days                  │     │
│  │  RPO: seconds (replica), 24h (backup)                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Cloud Spanner:                                                    │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Multi-region config: built-in DR (99.999% SLA)         │     │
│  │  Automatic failover between regions                     │     │
│  │  Synchronous replication (zero data loss)               │     │
│  │  RPO: 0, RTO: ~0                                       │     │
│  │  No manual DR needed — Spanner handles it               │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  AlloyDB:                                                          │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Cross-region replication (preview)                     │     │
│  │  Automated backups                                      │     │
│  │  Point-in-time recovery                                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Firestore:                                                        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Multi-region: nam5, eur3 (automatic replication)       │     │
│  │  99.999% SLA in multi-region mode                       │     │
│  │  Point-in-time recovery (PITR) available                │     │
│  │  Scheduled exports to GCS for additional backup        │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Bigtable:                                                         │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Replication: add clusters in other regions             │     │
│  │  Automatic failover via app-level routing              │     │
│  │  Backups: managed backups or export to GCS             │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Memorystore Redis:                                                │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Cross-region replication (Standard tier)               │     │
│  │  RDB snapshots to GCS                                   │     │
│  │  For caching: rebuild from source on failover           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Cloud SQL: create cross-region read replica
gcloud sql instances create dr-replica \
    --master-instance-name=prod-primary \
    --region=europe-west1 \
    --tier=db-n1-standard-4 \
    --database-version=POSTGRES_15

# Cloud SQL: promote replica (during disaster)
gcloud sql instances promote-replica dr-replica

# Cloud SQL: restore from backup
gcloud sql backups list --instance=prod-primary
gcloud sql backups restore BACKUP_ID \
    --restore-instance=restored-instance \
    --backup-instance=prod-primary
```

---

## Part 9 — Storage DR Strategies

### GCS Redundancy Options

```
┌────────────────────────────────────────────────────────────────────┐
│         STORAGE DR                                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  GCS location types:                                               │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Regional:     1 region      → zone-level redundancy    │     │
│  │  Dual-region:  2 specific    → region-level redundancy  │     │
│  │                regions       → RPO: 0 (turbo replication)│    │
│  │  Multi-region: US/EU/Asia   → continent-level DR        │     │
│  │                              → RPO: near-zero           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Additional protections:                                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Object versioning: recover from accidental delete      │     │
│  │  Object lock / retention: prevent deletion              │     │
│  │  Soft delete: recover recently deleted objects (days)   │     │
│  │  Cross-bucket replication (STS): explicit DR copy       │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Cost comparison (per GB/month):                                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Regional Standard:      $0.020                          │     │
│  │  Dual-region Standard:   $0.036                          │     │
│  │  Multi-region Standard:  $0.026                          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Create dual-region bucket (turbo replication)
gcloud storage buckets create gs://my-dr-bucket \
    --location=us-central1+us-east1 \
    --default-storage-class=STANDARD \
    --enable-turbo-replication

# Enable object versioning
gcloud storage buckets update gs://my-bucket --versioning

# Enable soft delete (7-day recovery window)
gcloud storage buckets update gs://my-bucket \
    --soft-delete-duration=7d

# Cross-bucket replication (hourly sync)
gcloud transfer jobs create \
    gs://primary-bucket \
    gs://dr-bucket \
    --schedule-repeats-every=1h
```

---

## Part 10 — Networking & DNS Failover

### Automated Traffic Failover

```
┌────────────────────────────────────────────────────────────────────┐
│         DNS & NETWORK FAILOVER                                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Option 1: Global HTTP(S) Load Balancer (recommended)             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Global LB with backends in multiple regions             │     │
│  │  Health checks detect backend failure                    │     │
│  │  Automatic traffic rerouting (no DNS change needed)     │     │
│  │  Failover time: seconds                                  │     │
│  │                                                            │     │
│  │  Users → Global LB → healthy backend (auto-selected)    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Option 2: Cloud DNS Routing Policies                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Failover routing policy:                                │     │
│  │  • Primary: us-central1 IP                              │     │
│  │  • Backup: europe-west1 IP                              │     │
│  │  • Health check on primary                              │     │
│  │  • Auto-switch to backup when primary fails             │     │
│  │  Failover time: TTL-dependent (set low, e.g., 60s)     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Option 3: Geolocation Routing + Health Checks                   │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Route users to nearest region                           │     │
│  │  If region unhealthy → route to next nearest            │     │
│  │  Good for active-active multi-region                    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Cloud DNS failover routing policy
gcloud dns record-sets create app.example.com \
    --zone=my-zone \
    --type=A \
    --ttl=60 \
    --routing-policy-type=FAILOVER \
    --routing-policy-primary-data="us-central1-ip" \
    --routing-policy-backup-data-type=GEO \
    --routing-policy-backup-data="us=us-east1-ip;europe=europe-west1-ip" \
    --health-check=my-health-check \
    --backup-data-trickle-ratio=0.0
```

---

## Part 11 — GKE Disaster Recovery

### Kubernetes-Specific DR

```
┌────────────────────────────────────────────────────────────────────┐
│         GKE DR                                                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  GKE Backup for GKE:                                              │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Managed backup & restore for GKE workloads:             │     │
│  │  • Backup K8s resources (deployments, services, PVCs)   │     │
│  │  • Backup persistent volumes (PV data)                  │     │
│  │  • Schedule backups (daily, weekly)                      │     │
│  │  • Restore to same or different cluster/region          │     │
│  │  • Encryption with CMEK                                  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Multi-cluster DR:                                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Primary cluster: us-central1                             │     │
│  │  DR cluster: europe-west1                                 │     │
│  │                                                            │     │
│  │  Sync method:                                             │     │
│  │  • Config Sync (GitOps — same manifests deployed)       │     │
│  │  • CI/CD deploys to both clusters                       │     │
│  │  • Multi-cluster Ingress (Global LB)                    │     │
│  │                                                            │     │
│  │  Failover: Global LB detects unhealthy cluster          │     │
│  │  → routes to healthy cluster automatically              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Enable GKE Backup
gcloud container clusters update my-cluster \
    --zone=us-central1-a \
    --update-addons=BackupRestore=ENABLED

# Create backup plan
gcloud beta container backup-restore backup-plans create daily-backup \
    --cluster=projects/my-project/locations/us-central1-a/clusters/my-cluster \
    --location=us-central1 \
    --all-namespaces \
    --include-volume-data \
    --cron-schedule="0 2 * * *" \
    --backup-retain-days=30

# List backups
gcloud beta container backup-restore backups list \
    --backup-plan=daily-backup \
    --location=us-central1

# Restore to different cluster
gcloud beta container backup-restore restores create restore-1 \
    --restore-plan=my-restore-plan \
    --backup=projects/my-project/locations/us-central1/backupPlans/daily-backup/backups/backup-20240115 \
    --location=europe-west1
```

---

## Part 12 — Backup Strategies & Testing

### Comprehensive Backup Plan

```
┌────────────────────────────────────────────────────────────────────┐
│         BACKUP STRATEGY                                              │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Service              │ Backup Method           │ Retention        │
│  ──────────────────── │ ─────────────────────── │ ──────────────── │
│  Cloud SQL            │ Automated backups       │ 30 days          │
│  Cloud SQL            │ On-demand backup        │ Until deleted    │
│  Cloud SQL            │ Export to GCS (weekly)  │ 1 year           │
│  GCE Disks            │ Scheduled snapshots     │ 14 days          │
│  GCS                  │ Object versioning       │ 90 days          │
│  GCS                  │ Cross-region copy (STS) │ Same as primary  │
│  Firestore            │ Scheduled exports (GCS) │ 90 days          │
│  GKE                  │ GKE Backup              │ 30 days          │
│  BigQuery             │ Table snapshots         │ 7 days (auto)    │
│  BigQuery             │ Dataset copy (BQ DTS)   │ Per policy       │
│  Terraform state      │ GCS versioning          │ 30 versions      │
│                                                                      │
│  TESTING (critical — untested backups are worthless):             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Monthly: restore Cloud SQL backup to test instance     │     │
│  │  Monthly: restore GKE backup to test cluster            │     │
│  │  Quarterly: full DR drill (failover to DR region)       │     │
│  │  Annually: complete rebuild from scratch (Terraform)    │     │
│  │                                                            │     │
│  │  Document: time to restore, data integrity, issues      │     │
│  │  Update runbooks based on drill findings               │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Schedule Cloud SQL backup
gcloud sql instances patch prod-db \
    --backup-start-time=02:00 \
    --enable-bin-log \
    --retained-backups-count=30 \
    --backup-location=us

# Create on-demand backup
gcloud sql backups create --instance=prod-db

# Schedule GCE disk snapshots
gcloud compute resource-policies create snapshot-schedule daily-snap \
    --region=us-central1 \
    --max-retention-days=14 \
    --daily-schedule \
    --start-time=03:00

gcloud compute disks add-resource-policies prod-disk \
    --resource-policies=daily-snap \
    --zone=us-central1-a

# Export Firestore to GCS
gcloud firestore export gs://my-backup-bucket/firestore/$(date +%Y%m%d)
```

---

## Part 13 — DR Automation & Runbooks

### Automated Failover

```
┌────────────────────────────────────────────────────────────────────┐
│         DR AUTOMATION                                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Automated failover pipeline:                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │                                                            │     │
│  │  Health Check (Cloud Monitoring)                          │     │
│  │       │ alert fires                                       │     │
│  │       ▼                                                    │     │
│  │  Cloud Monitoring Alert → Pub/Sub topic                  │     │
│  │       │                                                    │     │
│  │       ▼                                                    │     │
│  │  Cloud Function: failover-orchestrator                   │     │
│  │       │                                                    │     │
│  │       ├── 1. Verify primary is truly down (2nd check)   │     │
│  │       ├── 2. Promote Cloud SQL replica                   │     │
│  │       ├── 3. Update Cloud Run traffic                    │     │
│  │       ├── 4. Clear Memorystore cache in DR              │     │
│  │       ├── 5. Update DNS (if not using Global LB)        │     │
│  │       ├── 6. Notify on-call (PagerDuty / Slack)         │     │
│  │       └── 7. Log failover event to BigQuery             │     │
│  │                                                            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  DR Runbook template:                                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  RUNBOOK: Regional Failover                               │     │
│  │                                                            │     │
│  │  Trigger: Primary region (us-central1) unavailable       │     │
│  │  Duration: > 5 minutes of downtime                       │     │
│  │                                                            │     │
│  │  Steps:                                                   │     │
│  │  1. Confirm outage (not false alarm)                     │     │
│  │     Check: status.cloud.google.com                       │     │
│  │     Check: Cloud Monitoring dashboard                    │     │
│  │                                                            │     │
│  │  2. Notify stakeholders                                  │     │
│  │     Slack: #incident channel                             │     │
│  │     PagerDuty: page on-call SRE                         │     │
│  │                                                            │     │
│  │  3. Execute failover (automated or manual)               │     │
│  │     gcloud sql instances promote-replica dr-replica      │     │
│  │     Update LB backend weights                            │     │
│  │                                                            │     │
│  │  4. Verify DR region is serving correctly                │     │
│  │     Smoke tests: health endpoint, sample transactions   │     │
│  │                                                            │     │
│  │  5. Monitor DR region (first 4 hours critical)          │     │
│  │                                                            │     │
│  │  6. Plan failback when primary recovers                  │     │
│  │     Set up reverse replication                           │     │
│  │     Schedule maintenance window for failback             │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Cloud SQL with Cross-Region Replica ──────────────────────
resource "google_sql_database_instance" "primary" {
  name             = "prod-primary"
  database_version = "POSTGRES_15"
  region           = "us-central1"
  project          = var.project_id

  settings {
    tier              = "db-custom-4-16384"
    availability_type = "REGIONAL"         # multi-zone HA

    backup_configuration {
      enabled                        = true
      start_time                     = "02:00"
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 30
      }
      location = "us"
    }
  }
}

resource "google_sql_database_instance" "dr_replica" {
  name                 = "dr-replica"
  master_instance_name = google_sql_database_instance.primary.name
  database_version     = "POSTGRES_15"
  region               = "europe-west1"
  project              = var.project_id

  replica_configuration {
    failover_target = false
  }

  settings {
    tier              = "db-custom-4-16384"
    availability_type = "REGIONAL"
  }
}

# ─── GCS Dual-Region Bucket ──────────────────────────────────
resource "google_storage_bucket" "dr_bucket" {
  name     = "app-data-dr"
  location = "US-CENTRAL1+US-EAST1"
  project  = var.project_id

  custom_placement_config {
    data_locations = ["US-CENTRAL1", "US-EAST1"]
  }

  versioning {
    enabled = true
  }

  soft_delete_policy {
    retention_duration_seconds = 604800   # 7 days
  }
}

# ─── Global LB with Multi-Region Backends ────────────────────
resource "google_compute_backend_service" "api" {
  name        = "api-backend"
  protocol    = "HTTP"
  timeout_sec = 30
  project     = var.project_id

  backend {
    group = google_compute_region_network_endpoint_group.us.id
  }

  backend {
    group = google_compute_region_network_endpoint_group.eu.id
  }

  health_checks = [google_compute_health_check.api.id]
}

resource "google_compute_health_check" "api" {
  name               = "api-health"
  check_interval_sec = 10
  timeout_sec        = 5
  project            = var.project_id

  http_health_check {
    port         = 8080
    request_path = "/health"
  }
}

# ─── Disk Snapshot Schedule ───────────────────────────────────
resource "google_compute_resource_policy" "daily_snapshot" {
  name    = "daily-snapshot"
  region  = var.region
  project = var.project_id

  snapshot_schedule_policy {
    schedule {
      daily_schedule {
        days_in_cycle = 1
        start_time    = "03:00"
      }
    }
    retention_policy {
      max_retention_days = 14
    }
    snapshot_properties {
      storage_locations = ["us"]
    }
  }
}

# ─── Cloud DNS Failover Policy ────────────────────────────────
resource "google_dns_record_set" "failover" {
  name         = "app.${google_dns_managed_zone.zone.dns_name}"
  managed_zone = google_dns_managed_zone.zone.name
  type         = "A"
  ttl          = 60

  routing_policy {
    primary_backup {
      primary {
        internal_load_balancers {
          load_balancer_type = "regionalL4ilb"
          ip_address         = google_compute_forwarding_rule.primary.ip_address
          ip_protocol        = "tcp"
          network_url        = google_compute_network.vpc.id
          port               = "443"
          project            = var.project_id
          region             = "us-central1"
        }
      }
      backup_geo {
        location = "us"
        rrdatas  = [google_compute_address.backup.address]
      }
      trickle_ratio = 0
    }
  }
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# CLOUD SQL DR
# ═══════════════════════════════════════════════════════════════
# Create cross-region read replica
gcloud sql instances create REPLICA --master-instance-name=PRIMARY \
    --region=REGION --tier=TIER

# Promote replica (failover)
gcloud sql instances promote-replica REPLICA

# Restore from backup
gcloud sql backups list --instance=INSTANCE
gcloud sql backups restore BACKUP_ID --restore-instance=NEW \
    --backup-instance=ORIGINAL

# Point-in-time restore
gcloud sql instances clone INSTANCE CLONE \
    --point-in-time=2024-01-15T10:00:00Z

# ═══════════════════════════════════════════════════════════════
# GCS DR
# ═══════════════════════════════════════════════════════════════
gcloud storage buckets create gs://BUCKET --location=REGION1+REGION2
gcloud storage buckets update gs://BUCKET --versioning
gcloud storage buckets update gs://BUCKET --soft-delete-duration=7d

# ═══════════════════════════════════════════════════════════════
# DISK SNAPSHOTS
# ═══════════════════════════════════════════════════════════════
gcloud compute disks snapshot DISK --zone=ZONE --snapshot-names=NAME
gcloud compute snapshots list
gcloud compute disks create NEW-DISK --source-snapshot=SNAP --zone=ZONE

# Snapshot schedule
gcloud compute resource-policies create snapshot-schedule NAME \
    --region=R --daily-schedule --max-retention-days=14 --start-time=03:00
gcloud compute disks add-resource-policies DISK \
    --resource-policies=NAME --zone=ZONE

# ═══════════════════════════════════════════════════════════════
# GKE BACKUP
# ═══════════════════════════════════════════════════════════════
gcloud container clusters update CLUSTER --zone=Z \
    --update-addons=BackupRestore=ENABLED
gcloud beta container backup-restore backup-plans create NAME \
    --cluster=CLUSTER --location=L --all-namespaces \
    --include-volume-data --cron-schedule="0 2 * * *"

# ═══════════════════════════════════════════════════════════════
# FIRESTORE BACKUP
# ═══════════════════════════════════════════════════════════════
gcloud firestore export gs://BUCKET/firestore/$(date +%Y%m%d)
gcloud firestore import gs://BUCKET/firestore/20240115
```

---

## Part 15 — Real-World DR Patterns

### Pattern 1: SaaS Application (Warm Standby)

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 1: SAAS WARM STANDBY                                     │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Primary: us-central1          DR: us-east1              │        │
│  │                                                            │        │
│  │  Global HTTP(S) LB (single anycast IP)                   │        │
│  │                                                            │        │
│  │  Cloud Run (API)               Cloud Run (API)           │        │
│  │  auto-scale 1-100              min=1 (kept warm)         │        │
│  │       │                              │                    │        │
│  │  Cloud SQL PG                  Cloud SQL PG              │        │
│  │  Primary (HA)  ──async──►      Read Replica              │        │
│  │       │                              │                    │        │
│  │  Memorystore Redis             Memorystore Redis         │        │
│  │  (sessions)                    (cold, rebuild on need)   │        │
│  │       │                              │                    │        │
│  │  GCS (uploads)                 GCS (dual-region auto)    │        │
│  │                                                            │        │
│  │  Failover (semi-automated, ~15 min):                     │        │
│  │  1. Monitoring alert triggers PagerDuty                  │        │
│  │  2. SRE confirms and runs failover script               │        │
│  │  3. Script: promote SQL replica + scale Cloud Run       │        │
│  │  4. Global LB automatically routes to healthy backend   │        │
│  │  5. Sessions reset (users re-login once)                │        │
│  │                                                            │        │
│  │  Monthly cost: Primary $3,000 + DR $800 = $3,800        │        │
│  │  RPO: < 1 minute │ RTO: 15 minutes                      │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Financial System (Active-Active)

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 2: FINANCIAL ACTIVE-ACTIVE                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │                                                            │        │
│  │  Global HTTP(S) LB (geo-routing)                         │        │
│  │  ├── US users → us-central1                              │        │
│  │  └── EU users → europe-west1                             │        │
│  │                                                            │        │
│  │  us-central1                   europe-west1              │        │
│  │  ┌───────────────┐           ┌───────────────┐          │        │
│  │  │ GKE (private) │           │ GKE (private) │          │        │
│  │  │ API services  │           │ API services  │          │        │
│  │  └───────┬───────┘           └───────┬───────┘          │        │
│  │          │                           │                    │        │
│  │          └───────────┬───────────────┘                    │        │
│  │                      ▼                                    │        │
│  │              Cloud Spanner                                │        │
│  │              (multi-region: nam-eur-asia1)               │        │
│  │              Synchronous replication                      │        │
│  │              99.999% SLA                                  │        │
│  │              Zero data loss on failover                  │        │
│  │                                                            │        │
│  │  Failover: FULLY AUTOMATIC                               │        │
│  │  • Global LB detects unhealthy region → reroutes        │        │
│  │  • Spanner handles DB failover internally               │        │
│  │  • No manual intervention needed                        │        │
│  │  • RTO: seconds │ RPO: 0                                │        │
│  │                                                            │        │
│  │  Monthly cost: ~2.5x single region                       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Data Platform (Cold DR + Hot Analytics)

```
┌──────────────────────────────────────────────────────────────────────┐
│     PATTERN 3: DATA PLATFORM DR                                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Primary: us-central1          DR: us-east1              │        │
│  │                                                            │        │
│  │  Data ingestion:                                         │        │
│  │  Pub/Sub (global) → Dataflow → BigQuery                 │        │
│  │  (Pub/Sub is global — no DR needed for ingestion)       │        │
│  │                                                            │        │
│  │  BigQuery:                                                │        │
│  │  ├── Primary datasets (us-central1)                      │        │
│  │  ├── BQDTS cross-region copy (daily) → us-east1         │        │
│  │  └── RPO: 24 hours for analytics data                   │        │
│  │                                                            │        │
│  │  GCS (data lake):                                        │        │
│  │  ├── Multi-region bucket (US) — automatic replication   │        │
│  │  └── RPO: near-zero                                      │        │
│  │                                                            │        │
│  │  Cloud SQL (metadata DB):                                │        │
│  │  ├── Cross-region read replica                           │        │
│  │  └── Promote on failover                                 │        │
│  │                                                            │        │
│  │  Dataflow (processing):                                  │        │
│  │  ├── Pipelines deployed via Terraform                    │        │
│  │  ├── Can restart in any region from GCS templates       │        │
│  │  └── Streaming: restart in DR region (catch up from PS) │        │
│  │                                                            │        │
│  │  Composer (orchestration):                               │        │
│  │  ├── DAGs in GCS (multi-region)                          │        │
│  │  ├── Cold standby: create new env from Terraform        │        │
│  │  └── RTO: 30-60 minutes (environment creation time)     │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Service | DR Strategy | RPO | RTO |
|---------|-----------|-----|-----|
| Cloud Run | Multi-region deploy + Global LB | 0 | Seconds |
| GKE | Multi-cluster + GKE Backup | Minutes | Minutes |
| Compute Engine | Snapshots + Terraform | Hours | Hours |
| Cloud SQL | Cross-region replica + promote | Seconds | Minutes |
| Cloud Spanner | Multi-region (built-in) | 0 | 0 |
| AlloyDB | Cross-region replication | Minutes | Minutes |
| Firestore | Multi-region (built-in) | 0 | 0 |
| Bigtable | Multi-cluster replication | Minutes | Minutes |
| GCS (multi-region) | Built-in replication | Near-zero | 0 |
| GCS (dual-region turbo) | Synchronous replication | 0 | 0 |
| BigQuery | Cross-region dataset copy | Hours | Hours |
| Pub/Sub | Global (built-in) | 0 | 0 |

---

## What is Disaster Recovery? (Beginner Explanation)

### The Fire Escape Analogy

```
Think of it this way:

DR is like having a fire escape plan for your building.

  🏢 Your office building = Your cloud infrastructure
  🔥 A fire breaks out    = A region goes down, data gets corrupted, etc.
  🚪 Fire escape plan     = Your disaster recovery strategy
  🏗️ Backup office        = Your DR region

You HOPE you never need the fire escape plan. But when a disaster
strikes — and eventually one will — that plan saves your business.

Without DR:  Fire → everyone panics → business stops → revenue lost
With DR:     Fire → follow the plan → move to backup → business continues

The question isn't "will a disaster happen?" — it's "when it happens,
how fast can we recover?"
```

### RPO and RTO — Real-World Analogy

```
Imagine you're writing a book on your laptop:

  RPO (Recovery Point Objective) — "How much work can I afford to lose?"
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                    │
  │  You save your book every hour. Your laptop crashes.              │
  │                                                                    │
  │  ◄── you lose up to 1 hour of writing ──►                        │
  │  Last save                    Crash                               │
  │  ───●──────────────────────────●───                               │
  │                                                                    │
  │  RPO = 1 hour → you accept losing up to 1 hour of work          │
  │                                                                    │
  │  If you save every 5 minutes → RPO = 5 minutes (less data loss) │
  │  If you use Google Docs (auto-save) → RPO ≈ 0 (no data loss)   │
  │  If you only save daily → RPO = 24 hours (lots of data loss)    │
  │                                                                    │
  │  In cloud terms:                                                  │
  │  • Database backups every hour = RPO of 1 hour                   │
  │  • Synchronous replication = RPO of 0                            │
  └──────────────────────────────────────────────────────────────────┘

  RTO (Recovery Time Objective) — "How fast must I be working again?"
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                    │
  │  Your laptop crashes. How long until you can write again?        │
  │                                                                    │
  │                  ◄── downtime ──►                                  │
  │  Crash           Fix / get new laptop    Writing again           │
  │  ───●──────────────────●──────────────────●───                   │
  │                                                                    │
  │  RTO = 4 hours → you must be working again within 4 hours       │
  │                                                                    │
  │  Buy a new laptop (days) → high RTO                              │
  │  Switch to your tablet (minutes) → low RTO                      │
  │  Already have 2 laptops synced (seconds) → near-zero RTO       │
  │                                                                    │
  │  In cloud terms:                                                  │
  │  • Rebuild from Terraform (hours) = high RTO                    │
  │  • Promote a read replica (minutes) = low RTO                   │
  │  • Active-active with Global LB (seconds) = near-zero RTO      │
  └──────────────────────────────────────────────────────────────────┘

  The key trade-off: Lower RPO/RTO = Higher cost
```

### DR Tiers with Cost Comparison

```
┌──────────────────────────────────────────────────────────────────────┐
│     DR TIERS — FROM CHEAPEST TO MOST EXPENSIVE                       │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  COLD STANDBY — "The cheapest backup plan"                           │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  What:    Only backups stored in DR region               │        │
│  │           Nothing is running — rebuild from scratch      │        │
│  │  RPO:     24 hours (daily backups)                       │        │
│  │  RTO:     4-24 hours (time to rebuild infrastructure)   │        │
│  │  Cost:    ~5-10% of primary ($500/mo if primary=$5,000) │        │
│  │  Analogy: "Storing your book manuscript in a safe       │        │
│  │            deposit box — safe but slow to recover"       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  WARM STANDBY — "A minimal backup office ready to go"                │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  What:    DB replica running + app at minimum scale      │        │
│  │           Promote DB + scale up app on failure           │        │
│  │  RPO:     Minutes to 1 hour                              │        │
│  │  RTO:     30 min to 2 hours                              │        │
│  │  Cost:    ~25-35% of primary ($1,500/mo)                │        │
│  │  Analogy: "A small backup office with one desk and a    │        │
│  │            copy of your files — needs setup but fast"    │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  HOT STANDBY — "A fully equipped backup office, waiting"             │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  What:    Full infra running in DR region, just no       │        │
│  │           traffic — flip the switch and it's live        │        │
│  │  RPO:     Near-zero (sync replication)                   │        │
│  │  RTO:     5-15 minutes                                   │        │
│  │  Cost:    ~80-100% of primary ($4,500/mo)               │        │
│  │  Analogy: "A fully staffed backup office that mirrors   │        │
│  │            your main office — just send people there"    │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  ACTIVE-ACTIVE — "Two offices both serving customers"                │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  What:    Both regions serve live traffic simultaneously │        │
│  │           If one fails, the other absorbs all traffic    │        │
│  │  RPO:     0 (synchronous replication)                    │        │
│  │  RTO:     ~0 (automatic, no human needed)               │        │
│  │  Cost:    ~200%+ of single region ($10,000+/mo)         │        │
│  │  Analogy: "Two fully operational offices — if one burns │        │
│  │            down, customers don't even notice"            │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Summary:                                                             │
│  ┌────────────────────────────────────────────────────────┐          │
│  │  Tier          │ Data Loss │ Downtime  │ Cost         │          │
│  │  ────────────── │ ───────── │ ───────── │ ──────────── │          │
│  │  Cold           │ Hours     │ Hours/Days│ $            │          │
│  │  Warm           │ Minutes   │ 30min-2hr │ $$           │          │
│  │  Hot            │ Seconds   │ 5-15 min  │ $$$          │          │
│  │  Active-Active  │ Zero      │ Seconds   │ $$$$         │          │
│  └────────────────────────────────────────────────────────┘          │
│                                                                        │
│  Pick the tier that matches your business needs:                     │
│  • Blog/portfolio → Cold is fine (save money)                       │
│  • SaaS product → Warm or Hot (balance cost vs downtime)           │
│  • Banking/payments → Active-Active (zero tolerance for downtime)  │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Console Walkthrough: Key DR Operations

### Cloud SQL: Promote a Read Replica (Failover)

```
When to use: Your primary Cloud SQL instance is in a region that
went down, and you need to make the read replica the new primary.

Step-by-step in Google Cloud Console:

1. Go to Cloud SQL → Instances
   (Navigation menu → SQL)

2. Find your read replica instance (e.g., "dr-replica")
   It will show "Read replica" as the instance type

3. Click on the replica instance name to open it

4. Click the "PROMOTE REPLICA" button at the top
   ┌──────────────────────────────────────────────────────────────┐
   │  ⚠️  WARNING — This action is irreversible:                  │
   │                                                                │
   │  • The replica will become an independent primary instance   │
   │  • Replication from the old primary will STOP permanently    │
   │  • The replica will accept read AND write operations         │
   │  • You cannot undo this — you'll need to set up             │
   │    a NEW replica later for reverse replication               │
   └──────────────────────────────────────────────────────────────┘

5. Confirm the promotion in the dialog

6. Wait for the operation to complete (typically 2-5 minutes)
   The instance type will change from "Read replica" to "Primary"

7. Post-promotion steps:
   ┌──────────────────────────────────────────────────────────────┐
   │  □ Update your application's DB connection string           │
   │    to point to the new primary's IP address                 │
   │  □ Verify writes are working (test INSERT/UPDATE)           │
   │  □ Set up a new read replica for future DR                  │
   │  □ Update monitoring alerts for the new primary             │
   └──────────────────────────────────────────────────────────────┘
```

### Cloud SQL: Restore from Backup

```
When to use: Your database is corrupted, or you need to recover
data from a specific point in time.

Step-by-step in Google Cloud Console:

1. Go to Cloud SQL → Instances → click your instance

2. Click "Backups" tab in the left sidebar

3. You'll see a list of automated and on-demand backups:
   ┌──────────────────────────────────────────────────────────────┐
   │  Backup ID    │ Type      │ Created            │ Status     │
   │  ────────────  │ ───────── │ ────────────────── │ ────────── │
   │  1705305600   │ Automated │ Jan 15, 2:00 AM    │ Successful │
   │  1705219200   │ Automated │ Jan 14, 2:00 AM    │ Successful │
   │  1705132800   │ On-demand │ Jan 13, 5:30 PM    │ Successful │
   └──────────────────────────────────────────────────────────────┘

4. Option A — Restore to the SAME instance (overwrites current data):
   • Click the ⋮ menu next to the backup → "Restore"
   • Confirm: this REPLACES all current data with the backup

5. Option B — Restore to a NEW instance (safer, recommended):
   • Click the ⋮ menu next to the backup → "Restore"
   • Choose "Restore to a different instance"
   • Enter a new instance name (e.g., "restored-prod-db")
   • Select region and tier
   • Click "RESTORE" — a new instance is created from the backup

6. Option C — Point-in-time recovery (PITR):
   • Click "RECOVER TO A POINT IN TIME" button
   • Select the exact date and time to recover to
   • This creates a NEW instance with data up to that moment
   • Requires binary logging / WAL to be enabled

   ┌──────────────────────────────────────────────────────────────┐
   │  PITR is the most precise recovery method:                   │
   │  "Restore my database to exactly Jan 15 at 3:45 PM"        │
   │  → recovers all transactions up to that second             │
   └──────────────────────────────────────────────────────────────┘

7. After restore: verify data integrity by running test queries
   and checking row counts against known good values.
```

### Cloud Storage: Verify Cross-Region Replication

```
When to use: You want to confirm that your critical data is
replicated across regions for disaster recovery.

Step-by-step in Google Cloud Console:

1. Go to Cloud Storage → Buckets
   (Navigation menu → Cloud Storage → Buckets)

2. Click on your bucket name to view details

3. Check the "Location type" in the bucket overview:
   ┌──────────────────────────────────────────────────────────────┐
   │  Location type: Multi-region (US)                            │
   │  → Data is automatically replicated across US regions       │
   │  → RPO: near-zero, no action needed                        │
   │                                                                │
   │  Location type: Dual-region (us-central1, us-east1)         │
   │  → Data replicated between these two specific regions       │
   │  → Check "Turbo replication" for RPO = 0                   │
   │                                                                │
   │  Location type: Region (us-central1)                         │
   │  → ⚠️ NO cross-region replication!                          │
   │  → Data only has zone-level redundancy                      │
   │  → You need STS or a dual/multi-region bucket for DR       │
   └──────────────────────────────────────────────────────────────┘

4. For dual-region buckets — verify turbo replication:
   • Click "CONFIGURATION" tab
   • Check if "Turbo replication" is enabled
   • Turbo replication guarantees RPO = 0 (15-minute target)
   • Without turbo: RPO ≈ 15 minutes (best effort)

5. For regional buckets — set up cross-region copy:
   • Go to Data Transfer → Transfer Service (STS)
   • Create a transfer job: source bucket → DR bucket
   • Schedule: every 1 hour or on-change
   • This gives you a copy in a different region

6. Verify replication is working:
   • Upload a test file to the primary bucket
   • For multi/dual-region: file is available immediately
     from any region in the location
   • For STS: wait for next sync cycle, then verify in DR bucket
```

### Cloud DNS: Failover DNS Record

```
When to use: You need to switch traffic from a failed primary
region to your DR region by updating DNS.

Step-by-step in Google Cloud Console:

1. Go to Network services → Cloud DNS
   (Navigation menu → Network services → Cloud DNS)

2. Click on your DNS zone (e.g., "example-com")

3. Option A — Manual failover (update A record):
   ┌──────────────────────────────────────────────────────────────┐
   │  Find the A record for your app (e.g., app.example.com)    │
   │  Click "Edit" (pencil icon)                                  │
   │                                                                │
   │  Change the IP address:                                      │
   │  Before: 34.120.10.50  (primary region IP)                  │
   │  After:  34.150.20.100 (DR region IP)                       │
   │                                                                │
   │  Set TTL to 60 seconds (low TTL = faster propagation)      │
   │  Click "SAVE"                                                 │
   │                                                                │
   │  ⚠️ Note: Even with TTL=60s, some DNS resolvers cache      │
   │  longer. Full propagation can take up to the old TTL value. │
   │  Pro tip: Set TTL low (60s) BEFORE a disaster happens!      │
   └──────────────────────────────────────────────────────────────┘

4. Option B — Set up failover routing policy (recommended):
   ┌──────────────────────────────────────────────────────────────┐
   │  Click "ADD RECORD SET" or edit existing record             │
   │                                                                │
   │  DNS name:     app.example.com                               │
   │  Resource type: A                                             │
   │  TTL:          60 seconds                                    │
   │  Routing policy: Failover                                    │
   │                                                                │
   │  Primary:      34.120.10.50 (primary region)                │
   │  Backup:       34.150.20.100 (DR region)                    │
   │  Health check: Select your health check                     │
   │  Trickle ratio: 0% (send 0% to backup normally)            │
   │                                                                │
   │  → DNS automatically switches to backup when primary fails │
   │  → No manual intervention needed during a disaster!        │
   └──────────────────────────────────────────────────────────────┘

5. Verify failover is working:
   • Run: nslookup app.example.com (or dig app.example.com)
   • Confirm the IP returned matches your expected region
   • For failover policies: disable the primary health check
     target temporarily to test automatic DNS switchover
```

---

## DR Testing Runbook

### Step-by-Step: Simulate and Test Failover

```
┌──────────────────────────────────────────────────────────────────────┐
│     DR TESTING RUNBOOK                                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ⚠️  IMPORTANT: Always test in a staging/test environment first!    │
│  Never run DR drills on production without a rollback plan.          │
│                                                                        │
│  Schedule: Quarterly (at minimum)                                    │
│  Duration: 2-4 hours                                                  │
│  Team: SRE + Dev lead + DB admin                                     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Test 1: Cloud SQL Replica Promotion

```bash
# ═══════════════════════════════════════════════════════════════
# PRE-TEST: Verify replica is healthy and replication is active
# ═══════════════════════════════════════════════════════════════
gcloud sql instances describe dr-replica \
    --format="table(state, replicaConfiguration.failoverTarget)"

# Check replication lag (should be seconds, not minutes)
gcloud sql instances describe dr-replica \
    --format="value(replicaConfiguration)"

# ═══════════════════════════════════════════════════════════════
# STEP 1: Promote the read replica to a standalone primary
# ═══════════════════════════════════════════════════════════════
gcloud sql instances promote-replica dr-replica

# This command:
# • Stops replication from the old primary
# • Makes the replica a standalone read/write instance
# • Takes 2-5 minutes to complete
# • ⚠️ IRREVERSIBLE — you cannot undo this

# ═══════════════════════════════════════════════════════════════
# STEP 2: Verify the promoted instance is accepting writes
# ═══════════════════════════════════════════════════════════════
gcloud sql connect dr-replica --user=postgres
# Then run: INSERT INTO test_table VALUES ('dr-test', NOW());
# Then run: SELECT * FROM test_table WHERE id = 'dr-test';
```

### Test 2: Cloud SQL Instance Failover (HA)

```bash
# ═══════════════════════════════════════════════════════════════
# For HA instances (multi-zone) — test automatic failover
# ═══════════════════════════════════════════════════════════════

# Trigger a manual failover (simulates zone failure)
gcloud sql instances failover prod-db

# This command:
# • Switches the primary to the standby zone
# • Typically takes 30-120 seconds
# • Connections are briefly interrupted
# • App should automatically reconnect

# Verify failover completed
gcloud sql instances describe prod-db \
    --format="table(gceZone, state)"
```

### Test 3: Restore from Backup

```bash
# ═══════════════════════════════════════════════════════════════
# List available backups
# ═══════════════════════════════════════════════════════════════
gcloud sql backups list --instance=prod-db \
    --format="table(id, startTime, status, type)"

# ═══════════════════════════════════════════════════════════════
# Restore backup to a NEW test instance (safe — doesn't touch prod)
# ═══════════════════════════════════════════════════════════════
gcloud sql backups restore BACKUP_ID \
    --restore-instance=dr-test-restore \
    --backup-instance=prod-db

# ═══════════════════════════════════════════════════════════════
# Verify restored data integrity
# ═══════════════════════════════════════════════════════════════
gcloud sql connect dr-test-restore --user=postgres
# Run: SELECT COUNT(*) FROM critical_table;
# Compare count with production to verify data completeness

# ═══════════════════════════════════════════════════════════════
# Clean up test instance after verification
# ═══════════════════════════════════════════════════════════════
gcloud sql instances delete dr-test-restore --quiet
```

### Test 4: Point-in-Time Recovery

```bash
# ═══════════════════════════════════════════════════════════════
# Clone the instance to a specific point in time
# ═══════════════════════════════════════════════════════════════
gcloud sql instances clone prod-db pitr-test-instance \
    --point-in-time="2024-01-15T10:00:00Z"

# This creates a new instance with data exactly as it was
# at the specified timestamp

# Verify the restored data matches expectations
gcloud sql connect pitr-test-instance --user=postgres
# Run queries to verify data state at that point in time

# Clean up
gcloud sql instances delete pitr-test-instance --quiet
```

### Post-Failover Verification Checklist

```
┌──────────────────────────────────────────────────────────────────────┐
│     POST-FAILOVER VERIFICATION CHECKLIST                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Database Checks:                                                     │
│  □ Instance state is RUNNABLE                                        │
│  □ Can connect to the database                                       │
│  □ Read queries return expected data                                 │
│  □ Write queries succeed (INSERT, UPDATE)                            │
│  □ Row counts match expected values (compare with known good)       │
│  □ No replication lag errors in logs                                 │
│                                                                        │
│  Application Checks:                                                  │
│  □ Application can connect to the new DB endpoint                    │
│  □ Health check endpoint returns 200 OK                              │
│  □ Login/authentication works                                        │
│  □ Core user flows work (place order, view dashboard, etc.)         │
│  □ API responses return correct data                                 │
│  □ No elevated error rates in Cloud Monitoring                      │
│                                                                        │
│  Infrastructure Checks:                                               │
│  □ DNS resolves to the correct (DR) IP address                      │
│  □ Load balancer health checks show backends as healthy             │
│  □ SSL/TLS certificates are valid for the DR endpoint               │
│  □ Cloud Storage data accessible from DR region                     │
│  □ Pub/Sub subscriptions are processing messages                    │
│  □ Scheduled jobs (Cloud Scheduler) pointing to DR                  │
│                                                                        │
│  Monitoring & Alerts:                                                 │
│  □ Monitoring dashboards showing DR region metrics                  │
│  □ Alert policies updated for DR resources                          │
│  □ Error rate below acceptable threshold (< 1%)                     │
│  □ Latency within acceptable range                                   │
│                                                                        │
│  Documentation:                                                       │
│  □ Record actual RTO (time from disaster to recovery)               │
│  □ Record actual RPO (data loss, if any)                            │
│  □ Document any issues encountered during failover                  │
│  □ Update runbook with lessons learned                               │
│  □ Schedule follow-up to set up reverse replication                 │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Congratulations! You have completed the **Google Cloud Platform Complete Guide** — all 65 chapters covering every major GCP service from account setup to disaster recovery.

**Recommended next steps:**
- Review the **Architecture Framework** (Ch 62) for design best practices
- Practice with **Real-World Patterns** (Ch 63) for hands-on experience
- Set up **Cost Optimization** (Ch 64) to manage your cloud spend
- Implement **Disaster Recovery** (this chapter) for production readiness
