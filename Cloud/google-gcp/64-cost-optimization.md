# Chapter 64 — Cost Optimization Strategies

---

## Table of Contents

- [Overview](#overview)
- [Part 1: Cost Optimization Fundamentals](#part-1--cost-optimization-fundamentals)
- [Part 2: Committed Use Discounts (CUDs)](#part-2--committed-use-discounts-cuds)
- [Part 3: Sustained Use Discounts (SUDs)](#part-3--sustained-use-discounts-suds)
- [Part 4: Preemptible & Spot VMs](#part-4--preemptible--spot-vms)
- [Part 5: Right-Sizing Recommendations](#part-5--right-sizing-recommendations)
- [Part 6: Autoscaling & Scale-to-Zero](#part-6--autoscaling--scale-to-zero)
- [Part 7: Storage Cost Optimization](#part-7--storage-cost-optimization)
- [Part 8: Database Cost Optimization](#part-8--database-cost-optimization)
- [Part 9: BigQuery Cost Optimization](#part-9--bigquery-cost-optimization)
- [Part 10: Network Cost Optimization](#part-10--network-cost-optimization)
- [Part 11: GKE & Container Cost Optimization](#part-11--gke--container-cost-optimization)
- [Part 12: Billing Analysis & Budgets](#part-12--billing-analysis--budgets)
- [Part 13: FinOps Practices](#part-13--finops-practices)
- [Part 14: Terraform & gcloud CLI Reference](#part-14--terraform--gcloud-cli-reference)
- [Part 15: Real-World Cost Reduction Playbook](#part-15--real-world-cost-reduction-playbook)
- [Quick Reference](#quick-reference)
- [Console Walkthrough: Cost Management Tools](#console-walkthrough-cost-management-tools)
- [What's Next?](#whats-next)

---

## Overview

Cost optimization on GCP is not just about spending less — it's about maximizing the value per dollar spent. Google Cloud offers a rich set of discount programs (CUDs, SUDs, Spot VMs), intelligent recommendations (Active Assist), and pricing models (on-demand, flat-rate, free tiers) that can reduce your cloud bill by 30-70% when used strategically. This chapter covers every cost lever available, from compute discounts to storage lifecycle policies to network traffic routing.

---

## Part 1 — Cost Optimization Fundamentals

### Cost Levers Overview

```
┌────────────────────────────────────────────────────────────────────┐
│         COST OPTIMIZATION LEVERS                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  PRICING MODELS                                                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  On-Demand          — full price, no commitment          │     │
│  │  Sustained Use (auto) — up to 30% off (automatic)       │     │
│  │  Committed Use (1yr) — up to 37% off                    │     │
│  │  Committed Use (3yr) — up to 57% off                    │     │
│  │  Spot VMs            — up to 91% off (can be preempted) │     │
│  │  Free Tier           — always free usage quotas          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  OPERATIONAL LEVERS                                                │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Right-sizing     — match resource to actual usage       │     │
│  │  Autoscaling      — scale with demand (avoid over-prov) │     │
│  │  Scale-to-zero    — pay nothing when idle               │     │
│  │  Scheduling       — stop dev/test outside work hours    │     │
│  │  Data tiering     — lifecycle policies for storage      │     │
│  │  Network routing  — keep traffic regional               │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  ARCHITECTURE LEVERS                                               │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Serverless        — Cloud Run, Cloud Functions         │     │
│  │  Managed services  — reduce ops cost                    │     │
│  │  Caching           — reduce compute & DB load           │     │
│  │  Compression       — reduce storage & egress cost       │     │
│  │  Multi-region      — only if required (costs more)     │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Typical savings by lever:                                         │
│  CUDs: 37-57% │ Spot: 60-91% │ Right-size: 20-40%               │
│  Autoscale: 30-50% │ Storage tiering: 50-80% │ Caching: 20-60%  │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

### Cross-Cloud Comparison

| Discount Type | GCP | AWS | Azure |
|-------------|-----|-----|-------|
| Commit 1yr | CUD (37% off) | Reserved Instances (up to 40%) | Reserved VMs (up to 40%) |
| Commit 3yr | CUD (57% off) | Reserved Instances (up to 60%) | Reserved VMs (up to 62%) |
| Auto discount | Sustained Use (30%) | — | — |
| Spot/preemptible | Spot VMs (91% off) | Spot Instances (90% off) | Spot VMs (90% off) |
| Recommendations | Active Assist | Compute Optimizer | Advisor |
| Cost explorer | Billing Console + BQ | Cost Explorer | Cost Management |
| Budgets | Budgets & Alerts | AWS Budgets | Cost Management Budgets |

---

## Part 2 — Committed Use Discounts (CUDs)

### Resource-Based and Spend-Based CUDs

```
┌────────────────────────────────────────────────────────────────────┐
│         COMMITTED USE DISCOUNTS                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Resource-based CUDs (Compute Engine):                            │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Commit to specific vCPU + memory quantities             │     │
│  │  Applied automatically to matching VMs in the region     │     │
│  │                                                            │     │
│  │  Duration    │ Discount  │ Example: 8 vCPU + 32 GB      │     │
│  │  ───────────── │ ───────── │ ──────────────────────────── │     │
│  │  1 year      │ 37%       │ On-demand: $200/mo → $126/mo │     │
│  │  3 year      │ 57%       │ On-demand: $200/mo → $86/mo  │     │
│  │                                                            │     │
│  │  Applies to: GCE, GKE nodes, Dataproc, Cloud SQL       │     │
│  │  Covers: vCPU, memory, GPU, local SSD                   │     │
│  │  Scope: per region, per project (or shared across)      │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Spend-based CUDs:                                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Commit to spending $X/hour on specific services         │     │
│  │                                                            │     │
│  │  Services: Cloud SQL, Memorystore, Cloud Run,           │     │
│  │            Cloud Spanner, AlloyDB, BigQuery Editions    │     │
│  │                                                            │     │
│  │  Duration    │ Discount                                  │     │
│  │  ───────────── │ ────────────────────────────             │     │
│  │  1 year      │ 25%                                      │     │
│  │  3 year      │ 52%                                      │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Purchase resource-based CUD
gcloud compute commitments create my-commitment \
    --region=us-central1 \
    --plan=36-month \
    --resources=vcpu=64,memory=256GB

# Purchase with GPU
gcloud compute commitments create gpu-commitment \
    --region=us-central1 \
    --plan=12-month \
    --resources=vcpu=16,memory=64GB \
    --resources-accelerator=type=nvidia-tesla-t4,count=4

# List commitments
gcloud compute commitments list --region=us-central1

# View CUD recommendations
gcloud recommender recommendations list \
    --project=my-project \
    --location=us-central1 \
    --recommender=google.compute.commitment.UsageCommitmentRecommender
```

---

## Part 3 — Sustained Use Discounts (SUDs)

### Automatic Discounts

```
┌────────────────────────────────────────────────────────────────────┐
│         SUSTAINED USE DISCOUNTS                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Automatic — no action required:                                  │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  How it works:                                            │     │
│  │  • GCP tracks vCPU + memory usage per region per month  │     │
│  │  • Discount increases with usage throughout the month   │     │
│  │  • Applied at billing level (across all VMs in region)  │     │
│  │                                                            │     │
│  │  Usage (% of month) │ Effective Discount                │     │
│  │  ──────────────────── │ ──────────────────                │     │
│  │  0 - 25%             │ 0% (full price)                  │     │
│  │  25 - 50%            │ 20% on incremental               │     │
│  │  50 - 75%            │ 40% on incremental               │     │
│  │  75 - 100%           │ 60% on incremental               │     │
│  │  ──────────────────── │ ──────────────────                │     │
│  │  Full month (100%)   │ ~30% net discount                │     │
│  │                                                            │     │
│  │  Important:                                               │     │
│  │  • Only for general-purpose & compute-optimized         │     │
│  │  • NOT for memory-optimized, shared-core, Spot VMs      │     │
│  │  • NOT for GKE Autopilot (uses CUDs instead)           │     │
│  │  • Stacks with CUDs (CUD applied first, SUD on rest)   │     │
│  │  • N1, N2, N2D, C2, C2D, E2 machine types              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Example: 4 VMs running 25% of month each = 1 VM equivalent     │
│  → Google combines usage across VMs for maximum discount         │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 4 — Preemptible & Spot VMs

### Spot VMs (Interruptible Compute)

```
┌────────────────────────────────────────────────────────────────────┐
│         SPOT VMs                                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Up to 60-91% cheaper than on-demand:                             │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Feature              │ Spot VM                          │     │
│  │  ──────────────────── │ ──────────────────────────────── │     │
│  │  Discount             │ 60-91% off on-demand            │     │
│  │  Can be preempted     │ Yes (with 30-second warning)    │     │
│  │  Max runtime          │ No limit (was 24h for preempt.) │     │
│  │  SLA                  │ None                             │     │
│  │  Live migration       │ Not supported                   │     │
│  │  Auto-restart         │ Provisioning model: best-effort │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  GOOD FIT:                                                          │
│  ✓ Batch processing (Dataproc, Dataflow)                         │
│  ✓ CI/CD build workers                                            │
│  ✓ Rendering / transcoding                                        │
│  ✓ ML training (with checkpointing)                              │
│  ✓ Fault-tolerant workloads                                      │
│  ✓ GKE node pools (scale-out)                                    │
│                                                                      │
│  NOT SUITABLE:                                                      │
│  ✗ Production API servers (single instance)                      │
│  ✗ Databases (stateful, consistency-critical)                    │
│  ✗ Workloads that can't handle interruption                      │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Create Spot VM
gcloud compute instances create batch-worker \
    --zone=us-central1-a \
    --machine-type=n1-standard-8 \
    --provisioning-model=SPOT \
    --instance-termination-action=STOP \
    --maintenance-policy=TERMINATE

# Spot VM MIG (autoscaled)
gcloud compute instance-templates create spot-template \
    --machine-type=n1-standard-4 \
    --provisioning-model=SPOT \
    --instance-termination-action=DELETE

gcloud compute instance-groups managed create spot-mig \
    --template=spot-template \
    --size=0 \
    --zone=us-central1-a

gcloud compute instance-groups managed set-autoscaling spot-mig \
    --zone=us-central1-a \
    --max-num-replicas=50 \
    --min-num-replicas=0 \
    --target-cpu-utilization=0.7

# GKE Spot node pool
gcloud container node-pools create spot-pool \
    --cluster=my-cluster \
    --zone=us-central1-a \
    --spot \
    --num-nodes=0 \
    --enable-autoscaling \
    --min-nodes=0 \
    --max-nodes=20
```

---

## Part 5 — Right-Sizing Recommendations

### Active Assist Recommender

```
┌────────────────────────────────────────────────────────────────────┐
│         RIGHT-SIZING                                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Google monitors VM utilization and recommends changes:           │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Recommendation types:                                    │     │
│  │                                                            │     │
│  │  DOWNSIZE: CPU usage < 50% for 14+ days                 │     │
│  │  ┌──────────────────────────────────────────┐           │     │
│  │  │ Current: n1-standard-8 (8 vCPU, 30 GB)  │           │     │
│  │  │ Recommended: n1-standard-4 (4 vCPU, 15 GB)│          │     │
│  │  │ Savings: $73/month (50%)                  │           │     │
│  │  └──────────────────────────────────────────┘           │     │
│  │                                                            │     │
│  │  CHANGE TYPE: workload better suited for different type │     │
│  │  ┌──────────────────────────────────────────┐           │     │
│  │  │ Current: n1-standard-16 (CPU underused)  │           │     │
│  │  │ Recommended: e2-standard-8 (cheaper)     │           │     │
│  │  │ Savings: $85/month                        │           │     │
│  │  └──────────────────────────────────────────┘           │     │
│  │                                                            │     │
│  │  IDLE VM: no CPU activity for 14+ days                  │     │
│  │  ┌──────────────────────────────────────────┐           │     │
│  │  │ VM "test-server" idle for 30 days        │           │     │
│  │  │ Recommended: Stop or delete               │           │     │
│  │  │ Savings: $146/month (100%)               │           │     │
│  │  └──────────────────────────────────────────┘           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Get right-sizing recommendations
gcloud recommender recommendations list \
    --project=my-project \
    --location=us-central1-a \
    --recommender=google.compute.instance.MachineTypeRecommender \
    --format="table(name, description, primaryImpact.costProjection.cost)"

# Get idle VM recommendations
gcloud recommender recommendations list \
    --project=my-project \
    --location=us-central1-a \
    --recommender=google.compute.instance.IdleResourceRecommender

# Get idle disk recommendations
gcloud recommender recommendations list \
    --project=my-project \
    --location=us-central1-a \
    --recommender=google.compute.disk.IdleResourceRecommender

# Get idle IP recommendations
gcloud recommender recommendations list \
    --project=my-project \
    --location=us-central1 \
    --recommender=google.compute.address.IdleResourceRecommender

# Apply recommendation
gcloud recommender recommendations mark-claimed REC_ID \
    --project=my-project \
    --location=ZONE \
    --recommender=RECOMMENDER_ID \
    --etag=ETAG
```

---

## Part 6 — Autoscaling & Scale-to-Zero

### Pay Only for What You Use

```
┌────────────────────────────────────────────────────────────────────┐
│         AUTOSCALING COST SAVINGS                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Scale-to-Zero services (pay nothing when idle):                  │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Cloud Run (min instances = 0)                         │     │
│  │  • Cloud Functions (auto scale to zero)                  │     │
│  │  • GKE + Keda (event-driven pod autoscaling to zero)    │     │
│  │  • Dataflow (auto-scaling streaming pipelines)          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Schedule-based scaling (dev/staging):                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Stop dev VMs at 7 PM, start at 8 AM:                   │     │
│  │  • Cloud Scheduler → Cloud Function → stop/start VMs   │     │
│  │  • Saves ~65% (16 hours/day off)                        │     │
│  │                                                            │     │
│  │  Scale down staging GKE at night:                        │     │
│  │  • Cloud Scheduler → resize node pool to 0             │     │
│  │  • Saves ~60%                                           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Comparison: always-on vs autoscaled (API service):               │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Always-on: 3x n1-standard-4 = $292/month               │     │
│  │  Autoscaled Cloud Run: ~$50/month (same traffic)        │     │
│  │  Savings: 83%                                            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Schedule VM start/stop with Cloud Scheduler
# Start at 8 AM weekdays
gcloud scheduler jobs create http start-dev-vms \
    --schedule="0 8 * * 1-5" \
    --time-zone="America/New_York" \
    --uri="https://compute.googleapis.com/compute/v1/projects/my-project/zones/us-central1-a/instances/dev-server/start" \
    --http-method=POST \
    --oauth-service-account-email=scheduler-sa@my-project.iam.gserviceaccount.com

# Stop at 7 PM weekdays
gcloud scheduler jobs create http stop-dev-vms \
    --schedule="0 19 * * 1-5" \
    --time-zone="America/New_York" \
    --uri="https://compute.googleapis.com/compute/v1/projects/my-project/zones/us-central1-a/instances/dev-server/stop" \
    --http-method=POST \
    --oauth-service-account-email=scheduler-sa@my-project.iam.gserviceaccount.com
```

---

## Part 7 — Storage Cost Optimization

### GCS Lifecycle & Class Optimization

```
┌────────────────────────────────────────────────────────────────────┐
│         STORAGE COST OPTIMIZATION                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Storage class pricing (per GB/month):                            │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Standard:  $0.020  (frequently accessed)                │     │
│  │  Nearline:  $0.010  (accessed < 1x/month)               │     │
│  │  Coldline:  $0.004  (accessed < 1x/quarter)             │     │
│  │  Archive:   $0.0012 (accessed < 1x/year)                │     │
│  │                                                            │     │
│  │  100 TB at Standard = $2,000/month                       │     │
│  │  100 TB at Archive  = $120/month (94% savings)          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Lifecycle policies (automate transitions):                       │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Day 0:   Standard (hot data — active use)              │     │
│  │  Day 30:  → Nearline (warm — less frequent)            │     │
│  │  Day 90:  → Coldline (cold — rare access)              │     │
│  │  Day 365: → Archive (long-term retention)               │     │
│  │  Day 730: → Delete (no longer needed)                   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Autoclass (automatic optimization):                              │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  GCS automatically moves objects between classes        │     │
│  │  based on actual access patterns                         │     │
│  │  No lifecycle rules needed — Google manages it          │     │
│  │  Best for unpredictable access patterns                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Other savings:                                                    │
│  • Compress before upload (gzip, Brotli)                         │
│  • Delete old versions (object versioning cleanup)               │
│  • Delete unused buckets                                          │
│  • Use regional (not multi-region) if single-region access      │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```json
// lifecycle.json — apply with: gsutil lifecycle set lifecycle.json gs://bucket
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
      "condition": {"age": 30, "matchesStorageClass": ["STANDARD"]}
    },
    {
      "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
      "condition": {"age": 90, "matchesStorageClass": ["NEARLINE"]}
    },
    {
      "action": {"type": "SetStorageClass", "storageClass": "ARCHIVE"},
      "condition": {"age": 365, "matchesStorageClass": ["COLDLINE"]}
    },
    {
      "action": {"type": "Delete"},
      "condition": {"age": 730}
    }
  ]
}
```

---

## Part 8 — Database Cost Optimization

### Cloud SQL, Spanner, and More

```
┌────────────────────────────────────────────────────────────────────┐
│         DATABASE COST OPTIMIZATION                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Cloud SQL:                                                        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  1. Right-size instances                                  │     │
│  │     • Review CPU/memory utilization in Cloud Monitoring  │     │
│  │     • Downgrade if < 40% CPU average                    │     │
│  │                                                            │     │
│  │  2. Stop dev/staging outside business hours              │     │
│  │     • Cloud Scheduler → stop/start                      │     │
│  │     • Saves ~65%                                         │     │
│  │                                                            │     │
│  │  3. Use read replicas instead of scaling primary         │     │
│  │     • Read replicas can be smaller machine type          │     │
│  │                                                            │     │
│  │  4. Storage: delete old backups, reduce retained count   │     │
│  │                                                            │     │
│  │  5. CUDs: 25% (1yr) or 52% (3yr) for stable workloads  │     │
│  │                                                            │     │
│  │  6. Enterprise vs Enterprise Plus tier selection         │     │
│  │     • Don't pay for HA features you don't need          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Cloud Spanner:                                                    │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Use autoscaling (min/max nodes)                       │     │
│  │  • Granular instance sizing (100 processing units)      │     │
│  │  • CUDs for predictable workloads                       │     │
│  │  • Single-region cheaper than multi-region              │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Memorystore:                                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Right-size (review memory utilization)                │     │
│  │  • Basic tier (no HA) for caching (data can be rebuilt) │     │
│  │  • Standard tier for sessions (need persistence)        │     │
│  │  • CUDs available for 1yr/3yr                           │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 9 — BigQuery Cost Optimization

### Query & Storage Cost Reduction

```
┌────────────────────────────────────────────────────────────────────┐
│         BIGQUERY COST OPTIMIZATION                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  On-Demand vs Editions:                                            │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  On-Demand: $6.25/TiB scanned (1 TiB free/month)       │     │
│  │  Editions (flat-rate): fixed $/slot capacity            │     │
│  │                                                            │     │
│  │  Rule of thumb:                                          │     │
│  │  < $2000/month in queries → On-Demand                   │     │
│  │  > $2000/month in queries → Editions (with autoscaling) │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Query cost reduction:                                             │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  1. SELECT specific columns (never SELECT *)             │     │
│  │     Bad:  SELECT * FROM dataset.events (10 TB scanned)  │     │
│  │     Good: SELECT user_id, event_type FROM ... (100 GB)  │     │
│  │                                                            │     │
│  │  2. Use partitioned tables                               │     │
│  │     WHERE event_date = '2024-01-15' → scan 1 partition  │     │
│  │     vs scanning entire table                             │     │
│  │                                                            │     │
│  │  3. Use clustered tables                                 │     │
│  │     CLUSTER BY user_id → scan only relevant blocks      │     │
│  │                                                            │     │
│  │  4. Materialize common queries                           │     │
│  │     CREATE MATERIALIZED VIEW → pre-computed results     │     │
│  │                                                            │     │
│  │  5. Use LIMIT for exploration (with caution)            │     │
│  │     Note: LIMIT doesn't reduce data scanned             │     │
│  │     Use --dry_run to preview cost before running        │     │
│  │                                                            │     │
│  │  6. BI Engine (in-memory acceleration)                   │     │
│  │     Cache frequently queried tables in memory            │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Storage cost reduction:                                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • Active storage: $0.02/GB/month                       │     │
│  │  • Long-term (90+ days unmodified): $0.01/GB (auto)    │     │
│  │  • Set table expiration for temporary tables            │     │
│  │  • Delete unused datasets/tables                        │     │
│  │  • Use BigQuery Storage API for efficient reads         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Dry-run to estimate query cost
bq query --dry_run --use_legacy_sql=false \
    "SELECT user_id, event_type FROM dataset.events WHERE date = '2024-01-15'"
# Output: "This query will process 2.3 GB when run."
# Cost: 2.3 GB × $6.25/TiB = ~$0.014

# Set table expiration (7 days for temp tables)
bq update --expiration 604800 dataset.temp_table

# Set default table expiration for dataset
bq update --default_table_expiration 2592000 dataset  # 30 days
```

---

## Part 10 — Network Cost Optimization

### Egress Cost Reduction

```
┌────────────────────────────────────────────────────────────────────┐
│         NETWORK COST OPTIMIZATION                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Egress pricing (data leaving GCP):                               │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Intra-zone (same zone):        FREE                     │     │
│  │  Intra-region (diff zone):      $0.01/GB                │     │
│  │  Inter-region (within US):      $0.01/GB                │     │
│  │  Inter-region (intercontinental):$0.08-0.12/GB          │     │
│  │  Internet egress:               $0.08-0.23/GB           │     │
│  │  Through Cloud CDN:             $0.02-0.08/GB (cheaper) │     │
│  │  Private Google Access:         FREE (to Google APIs)   │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Cost reduction strategies:                                        │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  1. Cloud CDN for cacheable content                      │     │
│  │     → Serves from edge (cheaper egress + lower latency) │     │
│  │     → Up to 75% reduction in origin egress              │     │
│  │                                                            │     │
│  │  2. Keep traffic in same region                          │     │
│  │     → Colocate compute + storage + DB                   │     │
│  │     → Use regional GCS buckets (not multi-region)       │     │
│  │                                                            │     │
│  │  3. Standard tier networking (where latency allows)     │     │
│  │     → 30-50% cheaper than Premium tier                  │     │
│  │     → Uses ISP network instead of Google backbone       │     │
│  │                                                            │     │
│  │  4. Compress responses (gzip, Brotli)                   │     │
│  │     → 60-80% reduction in data transfer                 │     │
│  │                                                            │     │
│  │  5. Private Google Access                                │     │
│  │     → Access Google APIs without public IPs             │     │
│  │     → No egress charges for API traffic                 │     │
│  │                                                            │     │
│  │  6. Minimize cross-region replication                    │     │
│  │     → Only replicate what's needed for DR               │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 11 — GKE & Container Cost Optimization

### Kubernetes Cost Savings

```
┌────────────────────────────────────────────────────────────────────┐
│         GKE COST OPTIMIZATION                                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Use Autopilot (pay per pod, not per node)                     │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Standard: pay for entire node (even if 30% utilized)   │     │
│  │  Autopilot: pay only for pod resource requests          │     │
│  │  Savings: typically 30-50% for mixed workloads          │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  2. Spot node pools (for fault-tolerant workloads)               │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Spot nodes: 60-91% cheaper                              │     │
│  │  Use for: batch jobs, CI/CD, stateless workers          │     │
│  │  Mix: spot nodes (best-effort) + standard (guaranteed)  │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  3. Right-size resource requests & limits                         │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Over-provisioned requests → wasted node capacity       │     │
│  │  Use: Vertical Pod Autoscaler (VPA) recommendations    │     │
│  │  Review: kubectl top pods → compare to requests         │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  4. Cluster autoscaler + HPA                                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  HPA: scale pods based on CPU/memory/custom metrics     │     │
│  │  Cluster autoscaler: add/remove nodes as pods scale     │     │
│  │  Scale to zero for non-critical namespaces at night    │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  5. Use e2 machine types for GKE nodes                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  e2 machines: cheapest general-purpose                   │     │
│  │  e2-standard-4: ~30% cheaper than n1-standard-4        │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  6. Multi-tenant clusters (consolidate workloads)                │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  Instead of: 5 clusters × 3 nodes = 15 nodes            │     │
│  │  Use: 1 cluster × 5 nodes with namespaces               │     │
│  │  Savings: reduce overhead of control plane + base nodes │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 12 — Billing Analysis & Budgets

### Monitoring Costs

```bash
# ═══════════════════════════════════════════════════════════════
# BUDGET ALERTS
# ═══════════════════════════════════════════════════════════════
gcloud billing budgets create \
    --billing-account=BILLING_ACCOUNT_ID \
    --display-name="Monthly Budget" \
    --budget-amount=5000 \
    --threshold-rule=percent=0.5 \
    --threshold-rule=percent=0.9 \
    --threshold-rule=percent=1.0 \
    --notifications-rule-pubsub-topic=projects/my-project/topics/budget-alerts

# ═══════════════════════════════════════════════════════════════
# BILLING EXPORT TO BIGQUERY
# ═══════════════════════════════════════════════════════════════
# Console → Billing → Billing export → BigQuery export
# Enable: Standard usage cost, Detailed usage cost, Pricing

# Query billing data
bq query --use_legacy_sql=false "
SELECT
    service.description AS service,
    SUM(cost) AS total_cost,
    SUM(usage.amount) AS total_usage
FROM \`my-project.billing_dataset.gcp_billing_export_v1_XXXXXX\`
WHERE invoice.month = '202401'
GROUP BY service
ORDER BY total_cost DESC
LIMIT 20
"

# Cost per label (team/environment)
bq query --use_legacy_sql=false "
SELECT
    labels.value AS team,
    SUM(cost) AS total_cost
FROM \`my-project.billing_dataset.gcp_billing_export_v1_XXXXXX\`,
    UNNEST(labels) AS labels
WHERE labels.key = 'team'
  AND invoice.month = '202401'
GROUP BY team
ORDER BY total_cost DESC
"
```

### Essential Labels for Cost Tracking

| Label Key | Example Values | Purpose |
|-----------|---------------|---------|
| `team` | platform, data, ml | Team chargeback |
| `env` | dev, staging, prod | Environment cost split |
| `service` | api, worker, db | Service cost tracking |
| `cost-center` | eng-101, mktg-200 | Finance department |
| `owner` | alice, bob | Accountability |
| `managed-by` | terraform | Automation tracking |

---

## Part 13 — FinOps Practices

### Organizational Cost Management

```
┌────────────────────────────────────────────────────────────────────┐
│         FINOPS PRACTICES                                            │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  FinOps lifecycle:                                                 │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  INFORM → OPTIMIZE → OPERATE                             │     │
│  │                                                            │     │
│  │  INFORM:                                                  │     │
│  │  • Billing export to BigQuery (detailed cost data)       │     │
│  │  • Weekly cost review dashboard (Looker Studio)          │     │
│  │  • Cost allocation by team/service/environment           │     │
│  │  • Anomaly detection (unexpected cost spikes)           │     │
│  │                                                            │     │
│  │  OPTIMIZE:                                                │     │
│  │  • Apply Active Assist recommendations (monthly)        │     │
│  │  • Purchase CUDs for stable workloads                   │     │
│  │  • Right-size VMs, databases, storage                   │     │
│  │  • Implement lifecycle policies                         │     │
│  │  • Enable autoscaling everywhere                        │     │
│  │                                                            │     │
│  │  OPERATE:                                                 │     │
│  │  • Budget alerts at 50%, 90%, 100%                      │     │
│  │  • Automated cost controls (e.g., stop idle resources)  │     │
│  │  • Monthly FinOps review meeting                        │     │
│  │  • Team cost accountability (labels + dashboards)       │     │
│  │  • Architecture review for new services                 │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
│  Cost governance policies:                                         │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  • All resources must have team + env labels            │     │
│  │  • Dev environments auto-stop at 7 PM (scheduler)      │     │
│  │  • No GPU VMs without ML team approval                  │     │
│  │  • CUD review quarterly                                 │     │
│  │  • New project requires cost estimate                   │     │
│  │  • Monthly cost report to engineering leadership       │     │
│  └──────────────────────────────────────────────────────────┘     │
│                                                                      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Part 14 — Terraform & gcloud CLI Reference

### Terraform

```hcl
# ─── Committed Use Discount ──────────────────────────────────
resource "google_compute_region_commitment" "cud" {
  name   = "prod-commitment"
  region = var.region
  plan   = "THIRTY_SIX_MONTH"     # or TWELVE_MONTH
  type   = "COMPUTE_OPTIMIZED"

  resources {
    type   = "VCPU"
    amount = "64"
  }

  resources {
    type   = "MEMORY"
    amount = "256"                  # GB
  }
}

# ─── Budget Alert ─────────────────────────────────────────────
resource "google_billing_budget" "monthly" {
  billing_account = var.billing_account
  display_name    = "Monthly Budget"

  budget_filter {
    projects = ["projects/${var.project_id}"]
  }

  amount {
    specified_amount {
      currency_code = "USD"
      units         = "5000"
    }
  }

  threshold_rules {
    threshold_percent = 0.5
  }
  threshold_rules {
    threshold_percent = 0.9
  }
  threshold_rules {
    threshold_percent = 1.0
  }

  all_updates_rule {
    pubsub_topic = google_pubsub_topic.budget_alerts.id
  }
}

# ─── GCS Lifecycle (cost optimization) ───────────────────────
resource "google_storage_bucket" "data" {
  name     = "my-data-bucket"
  location = var.region
  project  = var.project_id

  autoclass {
    enabled = true               # automatic class management
  }

  lifecycle_rule {
    condition {
      age = 365
    }
    action {
      type          = "SetStorageClass"
      storage_class = "ARCHIVE"
    }
  }

  lifecycle_rule {
    condition {
      age = 730
    }
    action {
      type = "Delete"
    }
  }
}

# ─── Spot VM Instance Template ────────────────────────────────
resource "google_compute_instance_template" "spot" {
  name_prefix  = "spot-worker-"
  machine_type = "e2-standard-4"
  project      = var.project_id

  scheduling {
    preemptible                 = false
    provisioning_model          = "SPOT"
    instance_termination_action = "STOP"
    automatic_restart           = false
  }

  disk {
    source_image = "debian-cloud/debian-12"
    auto_delete  = true
    boot         = true
  }

  network_interface {
    network = google_compute_network.vpc.id
  }

  labels = {
    team = "platform"
    env  = "prod"
  }
}
```

### gcloud CLI Reference

```bash
# ═══════════════════════════════════════════════════════════════
# RECOMMENDATIONS
# ═══════════════════════════════════════════════════════════════
# VM right-sizing
gcloud recommender recommendations list \
    --project=P --location=ZONE \
    --recommender=google.compute.instance.MachineTypeRecommender

# Idle VMs
gcloud recommender recommendations list \
    --project=P --location=ZONE \
    --recommender=google.compute.instance.IdleResourceRecommender

# Idle disks
gcloud recommender recommendations list \
    --project=P --location=ZONE \
    --recommender=google.compute.disk.IdleResourceRecommender

# CUD recommendations
gcloud recommender recommendations list \
    --project=P --location=REGION \
    --recommender=google.compute.commitment.UsageCommitmentRecommender

# ═══════════════════════════════════════════════════════════════
# COMMITMENTS
# ═══════════════════════════════════════════════════════════════
gcloud compute commitments create NAME --region=R \
    --plan=PLAN --resources=vcpu=N,memory=M
gcloud compute commitments list --region=R
gcloud compute commitments describe NAME --region=R

# ═══════════════════════════════════════════════════════════════
# BUDGETS
# ═══════════════════════════════════════════════════════════════
gcloud billing budgets create \
    --billing-account=ACCT --display-name=NAME \
    --budget-amount=AMT --threshold-rule=percent=P

gcloud billing budgets list --billing-account=ACCT
gcloud billing budgets describe BUDGET_ID --billing-account=ACCT

# ═══════════════════════════════════════════════════════════════
# COST ANALYSIS
# ═══════════════════════════════════════════════════════════════
# View billing for project
gcloud billing projects describe PROJECT_ID

# List billing accounts
gcloud billing accounts list
```

---

## Part 15 — Real-World Cost Reduction Playbook

### 30-Day Cost Optimization Sprint

```
┌──────────────────────────────────────────────────────────────────────┐
│     30-DAY COST OPTIMIZATION PLAYBOOK                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Week 1: VISIBILITY                                                  │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Day 1-2:                                                  │        │
│  │  □ Enable billing export to BigQuery                      │        │
│  │  □ Add labels to all resources (team, env, service)      │        │
│  │  □ Set up budget alerts (50%, 90%, 100%)                 │        │
│  │                                                            │        │
│  │  Day 3-5:                                                  │        │
│  │  □ Build Looker Studio cost dashboard                    │        │
│  │  □ Identify top 10 cost drivers                          │        │
│  │  □ Review all Active Assist recommendations              │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Week 2: QUICK WINS (usually 15-30% savings)                        │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  □ Delete idle VMs (Recommender list)                    │        │
│  │  □ Delete unused disks, snapshots, static IPs           │        │
│  │  □ Right-size top 10 over-provisioned VMs               │        │
│  │  □ Stop dev/staging VMs outside business hours          │        │
│  │  □ Apply GCS lifecycle policies to all buckets           │        │
│  │  □ Set BigQuery table expirations for temp tables       │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Week 3: STRUCTURAL CHANGES (additional 10-20%)                     │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  □ Enable Spot VMs for batch/CI workloads               │        │
│  │  □ Configure autoscaling for all compute services       │        │
│  │  □ Switch to e2 machine types where possible            │        │
│  │  □ Enable Cloud CDN for public-facing services          │        │
│  │  □ Review BigQuery queries — add partitioning/clustering│        │
│  │  □ Consolidate GKE clusters (multi-tenant)              │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Week 4: COMMITMENTS (additional 25-57%)                             │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  □ Analyze 90-day usage trends                           │        │
│  │  □ Purchase CUDs for stable compute baseline            │        │
│  │  □ Purchase spend-based CUDs for Cloud SQL / Memorystore│        │
│  │  □ Document all optimizations and savings               │        │
│  │  □ Set up monthly cost review process                   │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Expected total savings: 30-60% of original cloud bill              │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Before/After Example

```
┌──────────────────────────────────────────────────────────────────────┐
│     CASE STUDY: MEDIUM COMPANY ($25K/month → $10K/month)            │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  BEFORE (Monthly: $25,000)                                           │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Compute Engine:   $12,000  (20 VMs, always-on, n1)     │        │
│  │  Cloud SQL:         $4,000  (3 instances, oversized)    │        │
│  │  GCS Storage:       $3,000  (50 TB all Standard)        │        │
│  │  BigQuery:          $2,500  (SELECT *, no partitions)   │        │
│  │  GKE:               $2,000  (Standard, 3 clusters)      │        │
│  │  Network egress:    $1,500  (no CDN, no compression)    │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  AFTER (Monthly: $10,200)                                            │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │  Compute Engine:    $4,800  (right-sized e2, CUDs, Spot)│        │
│  │  Cloud SQL:         $1,800  (right-sized, stop dev, CUD)│        │
│  │  GCS Storage:         $800  (lifecycle: 30 TB archived) │        │
│  │  BigQuery:            $800  (partitioned, SELECT cols)  │        │
│  │  GKE:               $1,200  (Autopilot, 1 cluster, Spot)│        │
│  │  Network egress:      $800  (CDN + compression)         │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                        │
│  Total savings: $14,800/month = $177,600/year (59% reduction)     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

| Optimization | Savings | Effort | Risk |
|-------------|---------|--------|------|
| Delete idle resources | 100% of idle | Low | None |
| Right-size VMs | 20-40% | Low | Low |
| Spot VMs (batch) | 60-91% | Medium | Medium |
| CUDs (1yr) | 37% | Low | Commitment |
| CUDs (3yr) | 57% | Low | Higher commitment |
| SUDs (automatic) | ~30% | None | None |
| GCS lifecycle | 50-80% | Low | None |
| BQ partitioning | 60-90% | Medium | None |
| Autoscaling | 30-50% | Medium | Low |
| Cloud CDN | 20-50% egress | Low | None |
| Dev stop/start | 65% of dev | Low | None |
| GKE Autopilot | 30-50% | Medium | Low |

---

## Console Walkthrough: Cost Management Tools

### Setting Up Budgets & Alerts

```
Step-by-step in Google Cloud Console:

1. Go to Billing → Budgets & alerts
   (Navigation menu → Billing → Budgets & alerts)

2. Click "CREATE BUDGET"

3. Budget scope:
   ┌──────────────────────────────────────────────────────────────┐
   │  Name:           "Monthly Production Budget"                  │
   │  Time range:     Monthly (resets each month)                  │
   │  Projects:       Select specific projects or "All projects"  │
   │  Services:       All services (or filter specific ones)      │
   │  Labels:         Optional — filter by env=prod               │
   └──────────────────────────────────────────────────────────────┘

4. Set budget amount:
   ┌──────────────────────────────────────────────────────────────┐
   │  Budget type:    "Specified amount"                           │
   │  Target amount:  $5,000 (your monthly target)                │
   │                                                                │
   │  Or choose "Last month's spend" to auto-track                │
   └──────────────────────────────────────────────────────────────┘

5. Set alert thresholds:
   ┌──────────────────────────────────────────────────────────────┐
   │  Threshold 1:   50% of budget  → early warning              │
   │  Threshold 2:   90% of budget  → approaching limit          │
   │  Threshold 3:  100% of budget  → budget exceeded            │
   │  Threshold 4:  120% of budget  → significant overrun        │
   │                                                                │
   │  "Actual" alerts:    trigger when spend reaches threshold   │
   │  "Forecasted" alerts: trigger when projected to exceed      │
   └──────────────────────────────────────────────────────────────┘

6. Notifications:
   ┌──────────────────────────────────────────────────────────────┐
   │  ✓ Email alerts to billing admins (default)                 │
   │  ✓ Email alerts to custom recipients (add team leads)       │
   │  Optional: Link Pub/Sub topic for automation                │
   │  (e.g., auto-stop non-production VMs when budget exceeded)  │
   └──────────────────────────────────────────────────────────────┘

7. Click "FINISH" to create the budget
```

### Viewing Active Assist Recommendations

```
Step-by-step in Google Cloud Console:

1. Go to Active Assist → Recommendations Hub
   (Navigation menu → Active Assist → Recommendations)

   Or directly: console.cloud.google.com/active-assist/recommendations

2. View recommendations by category:
   ┌──────────────────────────────────────────────────────────────┐
   │  COST tab shows all cost-saving recommendations:            │
   │                                                                │
   │  • VM right-sizing      — downsize over-provisioned VMs    │
   │  • Idle VM              — VMs with no activity for 14+ days│
   │  • Idle disk            — unattached persistent disks      │
   │  • Idle IP address      — static IPs not attached to any VM│
   │  • CUD recommendations  — commit for predictable workloads │
   │  • Idle Cloud SQL       — databases with no connections    │
   └──────────────────────────────────────────────────────────────┘

3. Click any recommendation to see details:
   ┌──────────────────────────────────────────────────────────────┐
   │  Resource:       my-vm-instance (n1-standard-8)             │
   │  Recommendation: Resize to n1-standard-4                    │
   │  Reason:         Average CPU 12% over last 14 days         │
   │  Estimated saving: $73.50/month                             │
   │  Impact:         Low risk                                   │
   │                                                                │
   │  [APPLY] — applies the change directly                      │
   │  [DISMISS] — hides the recommendation                       │
   └──────────────────────────────────────────────────────────────┘

4. Tip: Check the "COST" tab weekly — new recommendations
   appear as Google collects more usage data over time.
```

### Purchasing Committed Use Discounts (CUDs)

```
Step-by-step in Google Cloud Console:

1. Go to Compute Engine → Committed use discounts
   (Navigation menu → Compute Engine → Committed use discounts)

2. Click "PURCHASE COMMITMENT"

3. For Resource-based CUDs (Compute Engine):
   ┌──────────────────────────────────────────────────────────────┐
   │  Name:       "prod-compute-commitment"                       │
   │  Region:     us-central1 (where your VMs run)               │
   │  Plan:       12 months (37% off) or 36 months (57% off)    │
   │                                                                │
   │  Resources:                                                   │
   │  ┌────────────────────────────────────────────────┐          │
   │  │  vCPUs:    64  (total committed vCPU count)    │          │
   │  │  Memory:   256 GB (total committed memory)     │          │
   │  │  GPUs:     optional (type + count)             │          │
   │  │  Local SSD: optional                            │          │
   │  └────────────────────────────────────────────────┘          │
   │                                                                │
   │  ⚠️  CUDs are non-cancellable — review usage first!         │
   └──────────────────────────────────────────────────────────────┘

4. For Spend-based CUDs (Cloud SQL, Memorystore, etc.):
   ┌──────────────────────────────────────────────────────────────┐
   │  Go to: Billing → Commitments                                │
   │  Select service: Cloud SQL / Memorystore / Cloud Run        │
   │  Commit to $/hour spend for 1 or 3 years                   │
   │  Discount applied automatically to matching usage           │
   └──────────────────────────────────────────────────────────────┘

5. Review CUD utilization:
   Go to Billing → Commitments to see:
   • Active commitments and their utilization %
   • Under-utilized CUDs (you're paying but not using fully)
   • Expiring commitments (plan renewal ahead of time)
```

### Viewing Cost Breakdown by Project/Service/Label

```
Step-by-step in Google Cloud Console:

1. Go to Billing → Cost table (or Billing → Reports)
   (Navigation menu → Billing → Cost table)

2. Cost table — detailed cost breakdown:
   ┌──────────────────────────────────────────────────────────────┐
   │  Filters (top bar):                                           │
   │  • Time range:  Last 30 days / Custom range / Monthly       │
   │  • Projects:    All or specific projects                     │
   │  • Services:    All or specific (Compute, SQL, GCS, etc.)  │
   │  • SKUs:        Detailed per-SKU cost                       │
   │  • Labels:      Filter by team, env, service labels         │
   └──────────────────────────────────────────────────────────────┘

3. Group by different dimensions:
   ┌──────────────────────────────────────────────────────────────┐
   │  Group by: Project                                            │
   │  ┌──────────────────────────────────────────┐               │
   │  │  prod-project      $4,200   (55%)        │               │
   │  │  staging-project     $800   (11%)        │               │
   │  │  dev-project         $600    (8%)        │               │
   │  │  data-project      $2,000   (26%)        │               │
   │  └──────────────────────────────────────────┘               │
   │                                                                │
   │  Group by: Service                                            │
   │  ┌──────────────────────────────────────────┐               │
   │  │  Compute Engine    $3,500   (46%)        │               │
   │  │  Cloud SQL         $1,800   (24%)        │               │
   │  │  Cloud Storage       $900   (12%)        │               │
   │  │  BigQuery            $700    (9%)        │               │
   │  │  Networking          $700    (9%)        │               │
   │  └──────────────────────────────────────────┘               │
   │                                                                │
   │  Group by: Label (e.g., team)                                │
   │  ┌──────────────────────────────────────────┐               │
   │  │  team:platform     $3,000   (39%)        │               │
   │  │  team:data         $2,200   (29%)        │               │
   │  │  team:ml           $1,500   (20%)        │               │
   │  │  team:frontend       $900   (12%)        │               │
   │  └──────────────────────────────────────────┘               │
   └──────────────────────────────────────────────────────────────┘

4. Billing → Reports (visual charts):
   • Line chart of daily/monthly costs over time
   • Compare current month vs previous month
   • Spot cost anomalies and unexpected spikes
   • Download as CSV for further analysis

5. Pro tip: Enable Billing Export to BigQuery for advanced
   analysis, custom dashboards (Looker Studio), and cost
   anomaly detection with SQL queries.
```

---

## What's Next?

Continue to **Chapter 65: Disaster Recovery on GCP** → `65-disaster-recovery.md`
