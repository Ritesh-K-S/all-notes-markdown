# Chapter 57: AWS Migration Strategies

---

## Table of Contents

- [Overview](#overview)
- [Part 1: The 7 R's of Migration](#part-1-the-7-rs-of-migration)
- [Part 2: AWS Migration Hub](#part-2-aws-migration-hub)
- [Part 3: Application Migration Service (MGN)](#part-3-application-migration-service-mgn)
- [Part 4: Database Migration Service (DMS)](#part-4-database-migration-service-dms)
- [Part 5: Portal Walkthrough — DMS](#part-5-portal-walkthrough--dms)
- [Part 6: Terraform & CLI Examples](#part-6-terraform--cli-examples)
- [Part 7: Real-World Patterns](#part-7-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is Cloud Migration? Why Does It Matter?

Most companies didn't start in the cloud — they have servers, databases, and applications running in their own data centers ("on-premises"). **Cloud migration** is the process of moving these workloads to AWS.

**Why migrate?**
- 💰 Reduce costs (no more buying/maintaining hardware)
- 🚀 Scale up/down instantly (no more ordering servers 6 weeks in advance)
- 🌍 Go global faster (deploy in any region in minutes)
- 🔒 Better security (AWS invests billions in security infrastructure)

**The migration journey has 4 phases:**
1. **Assess**: Discover what you have, estimate costs, build a business case
2. **Mobilize**: Plan the migration, set up AWS environment, train the team
3. **Migrate**: Actually move workloads using the right strategy for each app
4. **Modernize**: Optimize for cloud-native (serverless, containers, managed services)

AWS provides a suite of migration services to move workloads from on-premises or other clouds to AWS. Understanding migration strategies and tools is critical for planning and executing successful cloud migrations.

```
What you'll learn:
├── 7 R's of migration strategies
├── AWS Migration Hub (central tracking)
├── Application Migration Service (MGN) — lift-and-shift
├── Database Migration Service (DMS) — database migration
├── Schema Conversion Tool (SCT) — heterogeneous migrations
└── Migration planning best practices
```

---

## Part 1: The 7 R's of Migration

```
┌─────────────────────────────────────────────────────────────────────┐
│           THE 7 R's OF MIGRATION                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Strategy         │ Description              │ Effort │ Example       │
│ ─────────────────┼──────────────────────────┼────────┼───────────────│
│ 1. Rehost        │ Lift-and-shift           │ Low    │ EC2 migration │
│   (lift & shift) │ Move as-is to cloud      │        │               │
│                   │                          │        │               │
│ 2. Replatform    │ Lift-tinker-and-shift    │ Low-Med│ Move DB to   │
│   (lift & tinker)│ Minor optimizations       │        │ RDS           │
│                   │                          │        │               │
│ 3. Repurchase    │ Drop-and-shop            │ Med    │ Move CRM to  │
│                   │ Switch to SaaS            │        │ Salesforce    │
│                   │                          │        │               │
│ 4. Refactor      │ Re-architect             │ High   │ Monolith to  │
│   (re-architect) │ Redesign for cloud-native│        │ microservices │
│                   │                          │        │               │
│ 5. Retire        │ Decommission             │ None   │ Remove unused │
│                   │ Turn off unneeded apps    │        │ servers       │
│                   │                          │        │               │
│ 6. Retain        │ Keep as-is               │ None   │ Legacy apps  │
│   (revisit)      │ Not ready to migrate      │        │ with deps     │
│                   │                          │        │               │
│ 7. Relocate      │ VMware Cloud on AWS      │ Low    │ Move VMs to  │
│                   │ Hypervisor-level move     │        │ VMC on AWS    │
│                                                                       │
│ ⚡ Most migrations start with Rehost, then optimize over time     │
│ ⚠️ Assess ALL applications before choosing strategies              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: AWS Migration Hub

```
Console → AWS Migration Hub

┌─────────────────────────────────────────────────────────────────────┐
│           AWS MIGRATION HUB                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Central place to track migration progress across AWS tools         │
│                                                                       │
│ Discovery:                                                           │
│ ├── Application Discovery Service (ADS)                           │
│ │   ├── Agentless discovery: VMware vCenter connector           │
│ │   │   → Collects VM inventory, CPU, memory, disk              │
│ │   ├── Agent-based discovery: Install on each server            │
│ │   │   → Detailed: processes, network connections, perf data  │
│ │   └── Import: Upload existing inventory spreadsheet           │
│ │                                                                  │
│ ├── Migration Hub Strategy Recommendations                       │
│ │   → Analyze source code and config for migration strategy    │
│ │                                                                  │
│ └── Migration Hub Orchestrator                                    │
│     → Automate multi-step migration workflows                   │
│                                                                       │
│ Dashboard:                                                           │
│ ├── Servers discovered: [count]                                   │
│ ├── Applications: Group servers into applications               │
│ ├── Migration status: Not started / In-progress / Complete     │
│ └── Home Region: Select once (cannot change)                    │
│                                                                       │
│ ⚡ Free to use — only pay for migration tools used              │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Application Migration Service (MGN)

```
Console → AWS Application Migration Service

┌─────────────────────────────────────────────────────────────────────┐
│           APPLICATION MIGRATION SERVICE (MGN)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Automated lift-and-shift (rehost) for servers                │
│ Replaces: CloudEndure Migration, Server Migration Service (SMS)  │
│                                                                       │
│ How it works:                                                        │
│ 1. Install AWS Replication Agent on source server                 │
│ 2. Agent continuously replicates data to AWS (block-level)       │
│ 3. Launch test instances to validate                               │
│ 4. Cutover → launch production instances                          │
│                                                                       │
│ ┌──────────┐     continuous       ┌───────────────┐               │
│ │ On-Prem   │ ─── replication ──→ │ Staging Area   │               │
│ │ Server    │     (block-level)   │ (EBS volumes)  │               │
│ └──────────┘                      └───────┬───────┘               │
│                                           │ launch                  │
│                                    ┌──────▼──────┐                 │
│                                    │ Target EC2   │                 │
│                                    │ Instance     │                 │
│                                    └─────────────┘                 │
│                                                                       │
│ Setup Walkthrough:                                                   │
│ Console → MGN → Get started                                        │
│ ├── Initialize service (creates IAM roles)                        │
│ ├── Replication template:                                          │
│ │   ├── Staging area subnet: [select VPC subnet ▼]             │
│ │   ├── Replication Server type: [t3.small ▼]                  │
│ │   ├── EBS volume type: [gp3 ▼] → for replicated data        │
│ │   ├── EBS encryption: [Default key ▼]                        │
│ │   ├── Data routing: Private IP / Public IP                   │
│ │   └── Throttle bandwidth: [0 (unlimited) ▼]                 │
│ ├── Source servers → Add server (install agent)                 │
│ ├── Launch template:                                               │
│ │   ├── Instance type right-sizing: [Basic ▼]                  │
│ │   ├── OS licensing: [BYOL / License-included ▼]             │
│ │   ├── Target subnet: [select ▼]                              │
│ │   └── Security groups: [select ▼]                            │
│ ├── Test → Launch test instance → Validate                      │
│ └── Cutover → Launch cutover instance → Finalize               │
│                                                                       │
│ ⚡ Supports: Windows, Linux, most x86 OS versions               │
│ ⚡ Minimal downtime (continuous replication until cutover)       │
│ ⚡ Free for 90 days per source server, then $0.042/hr           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Database Migration Service (DMS)

```
Console → AWS DMS

┌─────────────────────────────────────────────────────────────────────┐
│           DATABASE MIGRATION SERVICE (DMS)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What: Migrate databases to AWS with minimal downtime               │
│                                                                       │
│ Migration types:                                                      │
│ ├── Homogeneous: Same engine (MySQL → RDS MySQL)               │
│ │   → DMS only, no conversion needed                            │
│ ├── Heterogeneous: Different engine (Oracle → Aurora PostgreSQL)│
│ │   → SCT (convert schema) + DMS (migrate data)               │
│ └── Continuous replication: Ongoing CDC (change data capture)  │
│                                                                       │
│ Supported sources:                                                   │
│ ├── On-premises: Oracle, SQL Server, MySQL, PostgreSQL, etc.  │
│ ├── EC2 databases                                                 │
│ ├── RDS (all engines)                                             │
│ ├── Aurora                                                        │
│ ├── S3 (as source or target)                                    │
│ ├── MongoDB, DocumentDB                                          │
│ └── Kinesis, Kafka (as target)                                  │
│                                                                       │
│ Architecture:                                                        │
│ ┌──────────┐     ┌─────────────────┐     ┌───────────┐          │
│ │ Source DB  │ ──→ │ DMS Replication  │ ──→ │ Target DB  │          │
│ │ (on-prem)  │     │ Instance         │     │ (RDS)      │          │
│ └──────────┘     └─────────────────┘     └───────────┘          │
│                   EC2 running DMS engine                             │
│                                                                       │
│ Schema Conversion Tool (SCT):                                       │
│ ├── Desktop application (download from AWS)                      │
│ ├── Converts schema objects: tables, views, stored procedures  │
│ ├── Action items: flags code that can't auto-convert           │
│ ├── Assessment report: compatibility summary                   │
│ └── DMS Schema Conversion (managed alternative in console)    │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: Portal Walkthrough — DMS

```
Console → DMS → Create replication instance

┌─────────────────────────────────────────────────────────────────┐
│ Create Replication Instance                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Name: [my-dms-instance]                                        │
│ → Identifier for the replication instance                     │
│                                                                   │
│ Description: [Migration from on-prem Oracle to Aurora]       │
│                                                                   │
│ Instance class: [dms.r5.large ▼]                              │
│ → CPU and memory for replication                              │
│ → dms.t3.medium (dev), dms.r5.xlarge (production)           │
│                                                                   │
│ Engine version: [3.5.2 ▼]                                     │
│ → Latest recommended                                          │
│                                                                   │
│ High availability: ● Multi-AZ ○ Single-AZ                   │
│ → Multi-AZ for production (automatic failover)               │
│                                                                   │
│ Allocated storage: [100] GB                                    │
│ → Staging area for in-flight data                             │
│                                                                   │
│ VPC: [my-vpc ▼]                                               │
│ Replication subnet group: [my-subnet-group ▼]                │
│ → Subnets where instance can be placed                       │
│                                                                   │
│ Publicly accessible: ○ Yes ● No                              │
│ → No for security; use VPN/DX for on-prem                   │
│                                                                   │
│                         [Create]                                │
└─────────────────────────────────────────────────────────────────┘

Console → DMS → Endpoints → Create endpoint

┌─────────────────────────────────────────────────────────────────┐
│ Create Endpoint                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Endpoint type: ● Source endpoint ○ Target endpoint            │
│                                                                   │
│ Endpoint identifier: [oracle-source]                          │
│                                                                   │
│ Source engine: [Oracle ▼]                                      │
│ → oracle, mysql, postgresql, sqlserver, mongodb, s3, etc.    │
│                                                                   │
│ Access to endpoint database:                                   │
│ ● Provide access information manually                         │
│ ○ AWS Secrets Manager                                          │
│                                                                   │
│ Server name: [oracle.onprem.example.com]                     │
│ Port: [1521]                                                   │
│ SSL mode: [verify-ca ▼] → none/require/verify-ca/verify-full│
│ User name: [dms_user]                                         │
│ Password: [********]                                          │
│ Database name: [ORCL]                                         │
│                                                                   │
│ Test endpoint connection:                                      │
│ ├── Replication instance: [my-dms-instance ▼]               │
│ └── [Run test] → Status: successful ✓                       │
│                                                                   │
│                         [Create endpoint]                      │
└─────────────────────────────────────────────────────────────────┘

Console → DMS → Database migration tasks → Create task

┌─────────────────────────────────────────────────────────────────┐
│ Create Database Migration Task                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Task identifier: [oracle-to-aurora-migration]                 │
│                                                                   │
│ Replication instance: [my-dms-instance ▼]                    │
│ Source endpoint: [oracle-source ▼]                            │
│ Target endpoint: [aurora-target ▼]                            │
│                                                                   │
│ Migration type:                                                │
│ ● Migrate existing data                                       │
│ ○ Migrate existing data and replicate ongoing changes (CDC)  │
│ ○ Replicate data changes only                                 │
│ → CDC = Change Data Capture for minimal downtime             │
│                                                                   │
│ Target table preparation mode:                                 │
│ ● Do nothing    → tables must exist                          │
│ ○ Drop tables   → recreate tables                            │
│ ○ Truncate      → clear data, keep tables                    │
│                                                                   │
│ LOB column settings:                                           │
│ ● Limited LOB mode  Max LOB size: [32] KB                    │
│ ○ Full LOB mode     → slower, handles unlimited size         │
│ ○ Inline LOB mode                                             │
│                                                                   │
│ Enable validation: ☑ → compare source/target data            │
│ Enable CloudWatch logs: ☑                                     │
│                                                                   │
│ Table mappings:                                                │
│ ● JSON editor                                                  │
│ ○ Guided UI                                                    │
│ Schema: [%] → wildcard for all schemas                       │
│ Table: [%]  → wildcard for all tables                        │
│ Action: Include / Exclude                                      │
│                                                                   │
│                         [Create task]                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Terraform & CLI Examples

```hcl
# DMS Replication Instance
resource "aws_dms_replication_instance" "main" {
  replication_instance_id    = "my-dms-instance"
  replication_instance_class = "dms.r5.large"
  allocated_storage          = 100
  multi_az                   = true
  publicly_accessible        = false

  vpc_security_group_ids = [aws_security_group.dms.id]
  replication_subnet_group_id = aws_dms_replication_subnet_group.main.id
}

# DMS Source Endpoint
resource "aws_dms_endpoint" "source" {
  endpoint_id   = "oracle-source"
  endpoint_type = "source"
  engine_name   = "oracle"
  server_name   = "oracle.onprem.example.com"
  port          = 1521
  database_name = "ORCL"
  username      = "dms_user"
  password      = var.source_db_password
  ssl_mode      = "verify-ca"
}

# DMS Target Endpoint
resource "aws_dms_endpoint" "target" {
  endpoint_id   = "aurora-target"
  endpoint_type = "target"
  engine_name   = "aurora-postgresql"
  server_name   = aws_rds_cluster.aurora.endpoint
  port          = 5432
  database_name = "mydb"
  username      = "admin"
  password      = var.target_db_password
}

# DMS Replication Task
resource "aws_dms_replication_task" "migration" {
  replication_task_id      = "oracle-to-aurora"
  replication_instance_arn = aws_dms_replication_instance.main.replication_instance_arn
  source_endpoint_arn      = aws_dms_endpoint.source.endpoint_arn
  target_endpoint_arn      = aws_dms_endpoint.target.endpoint_arn
  migration_type           = "full-load-and-cdc"

  table_mappings = jsonencode({
    rules = [{
      rule-type = "selection"
      rule-id   = "1"
      rule-name = "all-tables"
      object-locator = {
        schema-name = "%"
        table-name  = "%"
      }
      rule-action = "include"
    }]
  })
}
```

```bash
# Create replication instance
aws dms create-replication-instance \
  --replication-instance-identifier my-dms-instance \
  --replication-instance-class dms.r5.large \
  --allocated-storage 100 \
  --multi-az

# Test endpoint connection
aws dms test-connection \
  --replication-instance-arn arn:aws:dms:... \
  --endpoint-arn arn:aws:dms:...

# Start replication task
aws dms start-replication-task \
  --replication-task-arn arn:aws:dms:... \
  --start-replication-task-type start-replication

# Monitor task
aws dms describe-replication-tasks \
  --filters Name=replication-task-id,Values=oracle-to-aurora
```

---

## Part 7: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD PATTERNS                                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Large-Scale Rehost Migration                             │
│ Discovery (ADS agent) → Group into applications                    │
│ → MGN replication → Test waves → Cutover waves                   │
│ → Decommission on-prem servers                                     │
│                                                                       │
│ Pattern 2: Database Migration (Heterogeneous)                      │
│ SCT (convert Oracle schema → PostgreSQL)                          │
│ → Fix action items (manual conversion)                            │
│ → DMS full-load + CDC                                              │
│ → Validate data → Cutover applications                           │
│                                                                       │
│ Pattern 3: Phased Migration                                          │
│ Phase 1: Rehost (quick wins, lift-and-shift)                     │
│ Phase 2: Replatform (move DBs to RDS, use S3)                   │
│ Phase 3: Refactor (re-architect for cloud-native)                │
│ → Optimize costs and performance at each phase                   │
│                                                                       │
│ ⚡ Best practices:                                                    │
│ 1. Always run Discovery first — know what you have              │
│ 2. Group servers into migration waves (10-50 servers/wave)     │
│ 3. Use DMS with CDC for near-zero-downtime DB migration        │
│ 4. Test, test, test before cutover                               │
│ 5. Plan rollback strategy for each wave                         │
│ 6. Use Migration Hub for central visibility                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
Migration Quick Reference:
├── 7 R's: Rehost, Replatform, Repurchase, Refactor, Retire, Retain, Relocate
├── Migration Hub: Central tracking dashboard (free)
├── Application Discovery Service: Discover on-prem servers
│   ├── Agentless: VMware vCenter (basic inventory)
│   └── Agent-based: Detailed (processes, network, performance)
├── MGN (Application Migration Service): Lift-and-shift servers
│   ├── Continuous block-level replication
│   ├── Test → Cutover workflow
│   └── Free 90 days per server
├── DMS (Database Migration Service): Migrate databases
│   ├── Homogeneous: Same engine (DMS only)
│   ├── Heterogeneous: Different engine (SCT + DMS)
│   ├── CDC: Change Data Capture (near-zero downtime)
│   └── Supports: Oracle, SQL Server, MySQL, PostgreSQL, MongoDB, S3...
├── SCT (Schema Conversion Tool): Convert schemas between engines
├── ⚡ Start with Rehost, optimize later
└── ⚡ Always test before cutover, plan rollback
```

---

## What's Next?

In **Chapter 58: AWS Snow Family & DataSync**, we'll cover Snowball Edge, Snowcone, Snowmobile for offline data transfer, plus DataSync and Transfer Family for online data migration.
