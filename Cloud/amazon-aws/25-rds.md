# Chapter 25: RDS - Relational Database Service

---

## Table of Contents

- [Overview](#overview)
- [Part 1: RDS Fundamentals](#part-1-rds-fundamentals)
- [Part 2: Creating an RDS Instance (Full Portal Walkthrough)](#part-2-creating-an-rds-instance-full-portal-walkthrough)
- [Part 3: Multi-AZ Deployments](#part-3-multi-az-deployments)
- [Part 4: Read Replicas](#part-4-read-replicas)
- [Part 5: Backups & Snapshots](#part-5-backups--snapshots)
- [Part 6: Parameter Groups & Option Groups](#part-6-parameter-groups--option-groups)
- [Part 7: Security](#part-7-security)
- [Part 8: Monitoring & Performance Insights](#part-8-monitoring--performance-insights)
- [Part 9: RDS Proxy](#part-9-rds-proxy)
- [Part 10: Terraform & CLI Examples](#part-10-terraform--cli-examples)
- [Part 11: Real-World Patterns](#part-11-real-world-patterns)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is RDS? Why Do We Need It?

If you've ever set up a MySQL or PostgreSQL database on a server, you know the pain: installing the software, configuring it, setting up backups, applying security patches, handling disk space, setting up replication for high availability... it's a lot of work.

**Amazon RDS (Relational Database Service)** is like **hiring a DBA (Database Administrator) who works 24/7 for free.** You tell AWS which database engine you want (MySQL, PostgreSQL, etc.), and AWS handles everything else — installation, patching, backups, failover, and scaling.

**Why not just run a database on EC2?**
- On EC2: You install, patch, back up, and manage everything yourself
- On RDS: AWS does it all — you just write SQL queries and build your app
- RDS is slightly more expensive than a raw EC2 instance, but the time and risk saved is worth it for most teams

**Simple real-world examples:**
- 🛒 An e-commerce site stores product catalog and orders in RDS MySQL
- 👤 A SaaS app stores user accounts and settings in RDS PostgreSQL
- 📊 A reporting system runs on RDS with read replicas to handle heavy queries

Amazon RDS is a managed relational database service that handles provisioning, patching, backups, and failover. You choose the engine (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server), and AWS handles the operational heavy-lifting.

```
What you'll learn:
├── RDS Fundamentals
│   ├── Managed vs self-managed databases
│   ├── Supported engines
│   └── Pricing model
├── Creating an RDS Instance (Full Portal Walkthrough)
│   ├── Engine selection
│   ├── Templates (Production, Dev/Test, Free tier)
│   ├── Instance configuration
│   ├── Storage (gp3, io2, magnetic)
│   ├── Connectivity (VPC, subnet group, public access, SG)
│   ├── Authentication
│   ├── Monitoring
│   └── Maintenance & backups
├── Multi-AZ (high availability)
├── Read Replicas (read scaling)
├── Backups & Snapshots
├── Parameter Groups & Option Groups
├── Security (encryption, IAM auth, SSL)
├── Performance Insights
├── RDS Proxy
├── Terraform & CLI examples
└── Real-world patterns
```

---

## Part 1: RDS Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           RDS CORE CONCEPTS                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ What RDS manages for you:                                           │
│ ├── Provisioning (EC2 + EBS under the hood)                      │
│ ├── OS and engine patching                                        │
│ ├── Automated backups (point-in-time recovery)                   │
│ ├── Multi-AZ failover (high availability)                        │
│ ├── Read replicas (read scaling)                                  │
│ ├── Monitoring (CloudWatch, Performance Insights)                │
│ ├── Storage auto-scaling                                          │
│ └── Encryption at rest + in transit                               │
│                                                                       │
│ What YOU still manage:                                               │
│ ├── Schema design and optimization                                │
│ ├── Query performance tuning                                      │
│ ├── Application-level connection management                      │
│ ├── Engine-specific parameter tuning                              │
│ └── ⚠️ No SSH access (it's managed — you can't log into the OS)  │
│                                                                       │
│ Supported engines:                                                   │
│ ┌──────────────────┬──────────────────────────────────────────────┐│
│ │ Engine           │ Notes                                        ││
│ ├──────────────────┼──────────────────────────────────────────────┤│
│ │ MySQL            │ Most popular open-source. Versions 5.7, 8.0 ││
│ │ PostgreSQL       │ Feature-rich. PostGIS, JSONB. Versions 13-16││
│ │ MariaDB          │ MySQL fork. Community-driven.                ││
│ │ Oracle           │ Enterprise. Bring-your-own-license or paid. ││
│ │ SQL Server       │ Microsoft. Express (free) to Enterprise.    ││
│ │ ⚡ Aurora        │ AWS-built. MySQL/PG compatible. 5x faster.  ││
│ │                  │ Separate chapter (26).                       ││
│ └──────────────────┴──────────────────────────────────────────────┘│
│                                                                       │
│ Instance classes:                                                    │
│ ├── db.t3/t4g: Burstable (dev/test, light prod). Cheapest.      │
│ ├── db.m6g/m7g: General purpose (most production workloads).    │
│ ├── db.r6g/r7g: Memory optimized (large datasets, caching).    │
│ └── db.x2g: Extra memory (SAP HANA, large in-memory DBs).      │
│                                                                       │
│ Pricing (us-east-1, MySQL):                                        │
│ ├── db.t3.micro: ~$0.017/hr ($12/month) — free tier eligible    │
│ ├── db.m6g.large: ~$0.152/hr ($110/month)                       │
│ ├── db.r6g.large: ~$0.192/hr ($138/month)                       │
│ ├── Storage: gp3 $0.08/GB/month, io2 $0.125/GB/month           │
│ ├── Multi-AZ: 2x instance cost                                   │
│ ├── Backups: Free up to DB size, then $0.095/GB/month           │
│ └── Free tier: db.t3.micro, 20 GB gp2, 20 GB backups, 12 months│
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Creating an RDS Instance (Full Portal Walkthrough)

```
Console → RDS → Databases → Create database

┌─────────────────────────────────────────────────────────────────┐
│           CREATE DATABASE                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Choose a database creation method ──                        │
│ ● Standard create                                               │
│ ○ Easy create                                                    │
│ → Standard: Full control over all configuration options       │
│ → Easy: Simplified, uses AWS best-practice defaults           │
│                                                                   │
│ ── Engine options ──                                            │
│ ○ Aurora (MySQL Compatible)                                    │
│ ○ Aurora (PostgreSQL Compatible)                               │
│ ● MySQL                                                         │
│ ○ MariaDB                                                       │
│ ○ PostgreSQL                                                    │
│ ○ Oracle                                                        │
│ ○ Microsoft SQL Server                                         │
│                                                                   │
│ Engine version: [MySQL 8.0.35 ▼]                               │
│ → Choose the latest stable version for new projects           │
│ → Check application compatibility before upgrading            │
│                                                                   │
│ ── Templates ──                                                 │
│ ○ Production (Multi-AZ, gp3, high availability)               │
│ ● Dev/Test (Single-AZ, smaller instance)                      │
│ ○ Free tier (db.t3.micro, 20 GB, single-AZ)                  │
│                                                                   │
│ → Production: Pre-selects Multi-AZ, gp3, larger instances    │
│ → Dev/Test: Single-AZ, smaller defaults. Good for staging.   │
│ → Free tier: Smallest possible. Learning/experimentation.    │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Settings ──                                                  │
│                                                                   │
│ DB instance identifier: [prod-mysql-01]                        │
│ → Unique name in this region. Used in the endpoint DNS.       │
│ → Example endpoint: prod-mysql-01.abc123.us-east-1.rds.amazonaws.com │
│                                                                   │
│ Master username: [admin]                                       │
│ → Default admin user for the database                         │
│ → Cannot be changed after creation                            │
│                                                                   │
│ Credentials management:                                        │
│   ● Self managed (you set the password)                        │
│   ○ AWS Secrets Manager (auto-rotate, recommended for prod)   │
│                                                                   │
│ Master password: [********]                                    │
│ Confirm password: [********]                                   │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Instance configuration ──                                   │
│                                                                   │
│ DB instance class:                                              │
│   ○ Standard classes (m classes)                               │
│   ○ Memory optimized classes (r, x classes)                   │
│   ● Burstable classes (t classes)                              │
│                                                                   │
│ Instance type: [db.t3.medium ▼]                                │
│   ├── db.t3.micro: 2 vCPU, 1 GB (free tier)                 │
│   ├── db.t3.small: 2 vCPU, 2 GB                              │
│   ├── db.t3.medium: 2 vCPU, 4 GB ← good start for dev      │
│   ├── db.m6g.large: 2 vCPU, 8 GB ← production start        │
│   └── db.r6g.xlarge: 4 vCPU, 32 GB ← memory-intensive      │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Storage ──                                                   │
│                                                                   │
│ Storage type: [General Purpose SSD (gp3) ▼]                   │
│   ├── gp3: Best price/performance. Configurable IOPS.        │
│   ├── io1/io2: Provisioned IOPS. Critical databases.         │
│   └── Magnetic: Legacy. ⚠️ Don't use for new databases.      │
│                                                                   │
│ Allocated storage (GiB): [100]                                │
│ → Minimum: 20 GiB (free tier). Max: 65,536 GiB.             │
│                                                                   │
│ Provisioned IOPS: [3000]  (gp3/io1/io2 only)                 │
│ Storage throughput: [125] MB/s  (gp3 only)                    │
│                                                                   │
│ Storage autoscaling:                                           │
│ ☑ Enable storage autoscaling                                  │
│ Maximum storage threshold: [500] GiB                          │
│ → Automatically increases storage when running low           │
│ → Increases by 10% or 10 GB (whichever is more)             │
│ → ⚡ Always enable for production                             │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Availability & durability ──                                │
│                                                                   │
│ Multi-AZ deployment:                                           │
│   ○ Create a standby instance (Multi-AZ DB instance)          │
│   ○ Create a readable standby (Multi-AZ DB cluster) — NEW    │
│   ● Do not create a standby instance                           │
│                                                                   │
│ → Multi-AZ DB instance: Synchronous replication to standby   │
│   in another AZ. Automatic failover (60-120 seconds).         │
│   Standby is NOT readable. 2x cost.                           │
│ → Multi-AZ DB cluster: Two readable standbys. Faster         │
│   failover (~35 seconds). Transaction commit on 2/3 nodes.   │
│ → No standby: Single AZ. Dev/test only. No auto-failover.   │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Connectivity ──                                              │
│                                                                   │
│ Compute resource:                                               │
│   ● Don't connect to an EC2 compute resource                  │
│   ○ Connect to an EC2 compute resource                        │
│                                                                   │
│ VPC: [vpc-main ▼]                                              │
│                                                                   │
│ DB subnet group: [default-vpc-abc123 ▼]                       │
│ → Group of subnets where RDS can place instances             │
│ → Must span at least 2 AZs                                   │
│ → ⚡ Use private subnets only                                 │
│                                                                   │
│ Public access: ● No ○ Yes                                     │
│ → No: Only accessible from within VPC (recommended)          │
│ → Yes: Assigns public IP. ⚠️ Security risk for production    │
│                                                                   │
│ VPC security group:                                             │
│   ● Choose existing                                             │
│   ○ Create new                                                  │
│ Existing: [sg-rds-mysql ▼]                                    │
│ → Allow MySQL port 3306 from application security group      │
│ → PostgreSQL: 5432, SQL Server: 1433, Oracle: 1521           │
│                                                                   │
│ Availability Zone: [No preference ▼]                           │
│ → "No preference" = AWS chooses (recommended)                 │
│ → Or pick specific AZ to co-locate with app servers          │
│                                                                   │
│ Database port: [3306]                                          │
│ → Default for MySQL. Can customize.                           │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Database authentication ──                                  │
│                                                                   │
│ ● Password authentication                                      │
│ ○ Password and IAM database authentication                    │
│ ○ Password and Kerberos authentication                        │
│                                                                   │
│ → Password: Standard username/password (most common)         │
│ → IAM: Generate auth tokens via IAM. No password in app.    │
│   ⚡ Recommended for production (rotate without app changes)  │
│ → Kerberos: Active Directory integration (Oracle, SQL Server)│
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Monitoring ──                                                │
│                                                                   │
│ ☑ Turn on Performance Insights                                │
│ Retention: [7 days (free) ▼]  (or 2 years paid)             │
│ → Visual database performance monitoring                      │
│ → Shows: wait events, SQL queries, top consumers             │
│ → ⚡ Always enable — free for 7 days retention                │
│                                                                   │
│ Enhanced Monitoring:                                            │
│ ☑ Enable Enhanced Monitoring                                  │
│ Granularity: [60 seconds ▼]                                   │
│ Monitoring Role: [rds-monitoring-role ▼]                      │
│ → OS-level metrics: CPU, memory, disk I/O, processes        │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Additional configuration ──                                 │
│                                                                   │
│ Initial database name: [myapp]                                │
│ → Creates this database automatically on launch              │
│ → Leave blank = no database created (you create later)       │
│                                                                   │
│ DB parameter group: [default.mysql8.0 ▼]                      │
│ → Engine configuration (max_connections, innodb settings)    │
│ → Create custom parameter group for production tuning        │
│                                                                   │
│ Option group: [default:mysql-8-0 ▼]                           │
│ → Engine-specific features (e.g., Oracle TDE, MySQL memcached│
│                                                                   │
│ ── Backup ──                                                    │
│ ☑ Enable automated backups                                    │
│ Backup retention period: [7 ▼] days (max: 35)                │
│ Backup window: [03:00 - 04:00 UTC ▼]                         │
│ ☑ Copy tags to snapshots                                      │
│                                                                   │
│ → Automated backups: Daily full backup + transaction logs    │
│ → Point-in-time recovery: Restore to any second within       │
│   retention period                                             │
│ → ⚡ Set retention to 7+ days for production                  │
│                                                                   │
│ ── Encryption ──                                                │
│ ☑ Enable encryption                                           │
│ KMS key: [aws/rds (default) ▼]                                │
│ → ⚡ Always enable for production                             │
│ → Cannot be changed after creation                            │
│                                                                   │
│ ── Log exports ──                                               │
│ ☑ Audit log                                                    │
│ ☑ Error log                                                    │
│ ☑ General log                                                  │
│ ☑ Slow query log                                               │
│ → Exports to CloudWatch Logs for analysis                    │
│                                                                   │
│ ── Maintenance ──                                               │
│ ☑ Enable auto minor version upgrade                           │
│ Maintenance window: [Sun 04:00 - Sun 04:30 ▼]                │
│ → Minor version upgrades applied automatically               │
│ → Major version upgrades require manual action               │
│                                                                   │
│ ── Deletion protection ──                                      │
│ ☑ Enable deletion protection                                  │
│ → Prevents accidental deletion via console/CLI               │
│ → Must uncheck before deleting the database                  │
│ → ⚡ Always enable for production                             │
│                                                                   │
│                    [Create database]                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 3: Multi-AZ Deployments

```
┌─────────────────────────────────────────────────────────────────────┐
│           MULTI-AZ DEPLOYMENTS                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Multi-AZ DB Instance (Classic):                                     │
│ ┌─────────────────┐         ┌─────────────────┐                    │
│ │ AZ-a            │         │ AZ-b            │                    │
│ │ ┌─────────────┐ │ sync    │ ┌─────────────┐ │                    │
│ │ │ Primary     │─┼────────►│ │ Standby     │ │                    │
│ │ │ (read/write)│ │ replica │ │ (NOT usable)│ │                    │
│ │ └─────────────┘ │         │ └─────────────┘ │                    │
│ └─────────────────┘         └─────────────────┘                    │
│        ↑                           ↑                               │
│   app connects              auto-failover                         │
│   (DNS endpoint)            (60-120 seconds)                      │
│                                                                       │
│ → Standby receives synchronous replication                       │
│ → NOT readable (pure standby for failover only)                 │
│ → Failover: DNS flips to standby (same endpoint)               │
│ → Triggers: AZ failure, instance failure, maintenance           │
│ → 2x cost (you pay for both instances)                          │
│                                                                       │
│ Multi-AZ DB Cluster (New — recommended):                           │
│ ┌──────────┐    ┌──────────┐    ┌──────────┐                      │
│ │ AZ-a     │    │ AZ-b     │    │ AZ-c     │                      │
│ │ ┌──────┐ │    │ ┌──────┐ │    │ ┌──────┐ │                      │
│ │ │Writer│ │───►│ │Reader│ │───►│ │Reader│ │                      │
│ │ └──────┘ │sync│ └──────┘ │sync│ └──────┘ │                      │
│ └──────────┘    └──────────┘    └──────────┘                      │
│                                                                       │
│ → 2 reader instances (ARE readable for read queries)            │
│ → Writer endpoint + Reader endpoint                              │
│ → Faster failover (~35 seconds vs 60-120)                       │
│ → Transaction log-based replication (more efficient)            │
│ → Available for MySQL 8.0.28+, PostgreSQL 13.4+                │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: Read Replicas

```
Console → RDS → Select database → Actions → Create read replica

┌─────────────────────────────────────────────────────────────────┐
│           CREATE READ REPLICA                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ DB instance identifier: [prod-mysql-01-replica-us-west]       │
│ → Each replica gets its own endpoint                          │
│                                                                   │
│ AWS Region: [US West (Oregon) ▼]                               │
│ → Same region: Low-latency read scaling                       │
│ → Cross-region: DR, geo-distributed reads                     │
│                                                                   │
│ Instance class: [db.r6g.large ▼]                               │
│ → Can be different from primary                               │
│ → Replica can be larger (for heavy read workloads)           │
│                                                                   │
│ Multi-AZ: ☑ Create replica in a different zone               │
│ → Replica itself can be Multi-AZ!                             │
│                                                                   │
│ Storage: Inherited from primary (can upgrade)                 │
│                                                                   │
│ Network:                                                        │
│   VPC: [vpc-west ▼]                                            │
│   Subnet group: [rds-private-west ▼]                          │
│   Public access: ○ Yes ● No                                   │
│   Security group: [sg-rds-replica ▼]                          │
│                                                                   │
│ Replication:                                                    │
│ ├── Asynchronous replication (eventual consistency)           │
│ ├── Replica lag: Typically <1 second (same region)           │
│ ├── Cross-region: Higher lag (seconds to minutes)            │
│ └── ⚠️ Reads from replica may be slightly stale!             │
│                                                                   │
│                    [Create read replica]                        │
│                                                                   │
│ Read Replica Key Facts:                                        │
│ ├── Up to 15 read replicas (Aurora) or 5 (other engines)    │
│ ├── Can be promoted to standalone DB (breaks replication)    │
│ ├── Replica can have read replicas (daisy chain)             │
│ ├── Same-region: No data transfer cost                        │
│ ├── Cross-region: Data transfer charges apply                │
│ └── Use cases: Reporting, analytics, read-heavy apps         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Application usage pattern:
├── Write queries → Primary endpoint
├── Read queries → Reader endpoint (or direct replica endpoints)
├── Connection pooling: Route reads to replicas, writes to primary
└── RDS Proxy: Manages connection routing automatically
```

---

## Part 5: Backups & Snapshots

```
┌─────────────────────────────────────────────────────────────────────┐
│           RDS BACKUPS                                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Automated Backups:                                                   │
│ ├── Daily full snapshot + transaction logs (every 5 min)         │
│ ├── Point-in-time recovery to any second within retention       │
│ ├── Retention: 1-35 days (0 = disabled, NOT recommended)        │
│ ├── Backup window: Choose low-traffic period                     │
│ ├── Free storage up to DB instance size                          │
│ ├── Stored in S3 (managed by AWS, not visible in your S3)       │
│ └── ⚡ Deleted when you delete the DB instance                   │
│                                                                       │
│ Manual Snapshots:                                                    │
│ ├── User-initiated (on-demand)                                   │
│ ├── Persist even after DB deletion                               │
│ ├── Can share with other AWS accounts                            │
│ ├── Can copy to other regions                                     │
│ ├── Used for: Before major changes, migration, archival         │
│ └── RDS → Databases → Select → Actions → Take snapshot          │
│                                                                       │
│ Point-in-Time Recovery:                                             │
│ RDS → Databases → Select → Actions → Restore to point in time   │
│   ├── Restore time: [Latest restorable time ▼]                  │
│   │   or: Custom → [2024-01-15 14:30:00 UTC]                   │
│   ├── DB instance identifier: [prod-mysql-01-restored]          │
│   ├── Creates a NEW instance (doesn't overwrite existing)       │
│   └── ⚡ Essential for accidental data corruption/deletion       │
│                                                                       │
│ Restore from Snapshot:                                              │
│ RDS → Snapshots → Select → Actions → Restore snapshot            │
│   ├── Creates a NEW instance from snapshot data                  │
│   ├── Can change: instance class, storage, VPC, Multi-AZ       │
│   └── Endpoint changes (new instance = new endpoint)            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 6: Parameter Groups & Option Groups

```
Console → RDS → Parameter groups → Create parameter group

┌─────────────────────────────────────────────────────────────────┐
│           PARAMETER GROUP                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Parameter group family: [mysql8.0 ▼]                           │
│ Group name: [prod-mysql-8-params]                              │
│ Description: [Production MySQL 8.0 tuning]                    │
│                                                                   │
│ Common parameters to tune:                                     │
│ ├── max_connections: Default 150. Increase for high-traffic.  │
│ ├── innodb_buffer_pool_size: % of memory for caching.         │
│ ├── slow_query_log: Enable for query optimization.            │
│ ├── long_query_time: Threshold for slow query (seconds).     │
│ ├── character_set_server: utf8mb4 for full Unicode.          │
│ ├── time_zone: Set to your app's timezone.                   │
│ └── log_bin_trust_function_creators: 1 (for replication).    │
│                                                                   │
│ Static vs Dynamic parameters:                                  │
│ ├── Dynamic: Applied immediately (no restart needed)          │
│ └── Static: Requires DB instance reboot to apply              │
│                                                                   │
│ ⚡ Best practice: Create custom parameter group for production  │
│ → Never modify the default parameter group directly           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Option Groups (engine-specific features):
├── MySQL: memcached plugin, audit plugin
├── Oracle: TDE, Oracle Application Express (APEX)
├── SQL Server: TDE, native backup/restore, SSRS
└── Less commonly modified than parameter groups
```

---

## Part 7: Security

```
┌─────────────────────────────────────────────────────────────────────┐
│           RDS SECURITY                                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Network security:                                                    │
│ ├── Deploy in private subnets (no public access)                 │
│ ├── Security groups: Allow DB port from app SG only             │
│ ├── No direct internet access                                    │
│ └── VPC endpoint for management API (optional)                   │
│                                                                       │
│ Encryption at rest:                                                  │
│ ├── AES-256 via AWS KMS                                          │
│ ├── Encrypts: storage, backups, snapshots, replicas             │
│ ├── ⚠️ Must enable at creation (cannot encrypt existing DB)      │
│ ├── To encrypt existing: snapshot → copy with encryption →      │
│ │   restore from encrypted snapshot                              │
│ └── aws/rds managed key (free) or custom CMK                    │
│                                                                       │
│ Encryption in transit:                                               │
│ ├── SSL/TLS connections supported                                │
│ ├── Force SSL: rds.force_ssl = 1 (in parameter group)          │
│ ├── Download CA certificate from AWS                             │
│ ├── Connection string: mysql --ssl-ca=rds-combined-ca-bundle.pem│
│ └── ⚡ Always force SSL for production                           │
│                                                                       │
│ IAM Database Authentication:                                        │
│ ├── Generate temporary auth token (15-minute expiry)            │
│ ├── No password stored in application                            │
│ ├── IAM policy controls who can connect                         │
│ ├── Works with MySQL, PostgreSQL, MariaDB                       │
│ └── Token: aws rds generate-db-auth-token --hostname ... --port │
│                                                                       │
│ Secrets Manager integration:                                        │
│ ├── Store DB credentials in Secrets Manager                     │
│ ├── Automatic rotation (every 30/60/90 days)                   │
│ ├── Application retrieves credentials at runtime                │
│ └── ⚡ Best practice for production credential management        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 8: Monitoring & Performance Insights

```
Console → RDS → Performance Insights

┌─────────────────────────────────────────────────────────────────┐
│           PERFORMANCE INSIGHTS                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ What: Visual dashboard for database performance analysis.      │
│                                                                   │
│ Dashboard shows:                                                │
│ ├── Database load (Average Active Sessions - AAS)             │
│ ├── Top wait events (what the DB is waiting on)               │
│ │   ├── CPU: Query using too much CPU                        │
│ │   ├── IO: Disk reads/writes (index missing?)               │
│ │   ├── Lock: Row/table locks (concurrent writes)            │
│ │   └── Network: Waiting for client/replica                  │
│ ├── Top SQL queries (ranked by load)                          │
│ ├── Top hosts, users, databases                               │
│ └── Counter metrics (buffer pool hit ratio, etc.)             │
│                                                                   │
│ How to use:                                                     │
│ 1. Enable Performance Insights (at DB creation or modify)     │
│ 2. Go to RDS → Performance Insights                           │
│ 3. Select your database                                        │
│ 4. Look for: DB Load > vCPUs = performance bottleneck!       │
│ 5. Click wait event → see which SQL queries are responsible  │
│ 6. Optimize the query or add indexes                           │
│                                                                   │
│ Pricing:                                                        │
│ ├── Free tier: 7 days retention                               │
│ ├── Long-term: 2 years retention ($__ per vCPU per month)    │
│                                                                   │
│ CloudWatch metrics for RDS:                                    │
│ ├── CPUUtilization, FreeableMemory                            │
│ ├── ReadIOPS, WriteIOPS                                        │
│ ├── ReadLatency, WriteLatency                                  │
│ ├── DatabaseConnections                                        │
│ ├── FreeStorageSpace                                           │
│ └── ReplicaLag (for read replicas)                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 9: RDS Proxy

```
Console → RDS → Proxies → Create proxy

┌─────────────────────────────────────────────────────────────────┐
│           CREATE RDS PROXY                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Proxy identifier: [prod-mysql-proxy]                           │
│                                                                   │
│ Engine family: ● MySQL ○ PostgreSQL ○ SQL Server               │
│                                                                   │
│ ☑ Require Transport Layer Security (TLS)                      │
│                                                                   │
│ Database: [prod-mysql-01 ▼]                                    │
│                                                                   │
│ Connection pool maximum: [100] (% of max_connections)         │
│                                                                   │
│ Secrets Manager secret: [prod-mysql-credentials ▼]            │
│ IAM role: [rds-proxy-role ▼]                                  │
│                                                                   │
│ VPC: [vpc-main ▼]                                              │
│ Subnets: ☑ subnet-priv-1a  ☑ subnet-priv-1b                  │
│ Security group: [sg-rds-proxy ▼]                              │
│                                                                   │
│ Why RDS Proxy?                                                  │
│ ├── Connection pooling: Reuse DB connections efficiently     │
│ ├── Lambda: Prevents connection exhaustion from Lambda       │
│ │   (100s of concurrent Lambdas = 100s of DB connections)   │
│ ├── Failover: 66% faster failover than direct connection    │
│ ├── IAM auth: Enforce IAM authentication centrally           │
│ └── ⚡ Essential for Lambda + RDS architectures              │
│                                                                   │
│ Application connects to: proxy endpoint instead of DB endpoint│
│ proxy-prod-mysql.proxy-abc123.us-east-1.rds.amazonaws.com    │
│                                                                   │
│                    [Create proxy]                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 9.5: Blue/Green Deployments

### What is a Blue/Green Deployment for RDS?

RDS Blue/Green Deployments let you make **major changes** (engine upgrades, schema changes, parameter changes) with **near-zero downtime** by creating a staging copy, applying changes there, then switching traffic.

```
Before switchover:
┌─────────────────────┐     ┌─────────────────────┐
│  BLUE (Production)  │     │  GREEN (Staging)     │
│  MySQL 5.7          │ ──→ │  MySQL 8.0           │
│  Current traffic    │ rep │  Changes applied     │
│  ●                  │     │  ○ No traffic yet    │
└─────────────────────┘     └─────────────────────┘

After switchover (< 1 minute):
┌─────────────────────┐     ┌─────────────────────┐
│  OLD BLUE           │     │  NEW PRODUCTION      │
│  MySQL 5.7          │     │  MySQL 8.0           │
│  ○ (delete later)   │     │  ● All traffic       │
└─────────────────────┘     └─────────────────────┘
```

### When to Use Blue/Green Deployments

```
✅ Major version upgrades (MySQL 5.7 → 8.0)
✅ Schema changes (add columns, change indexes)
✅ Parameter group changes
✅ Switching storage types
❌ Not needed for minor version upgrades (use maintenance window)
❌ Not supported for read replicas as source
```

### Console Walkthrough

```
Console → RDS → Databases → Select DB → Actions → Create Blue/Green Deployment

  ┌─────────────────────────────────────────────────────────┐
  │ Create Blue/Green Deployment                             │
  │                                                          │
  │ Blue/Green deployment identifier: mysql-upgrade          │
  │ Blue source: prod-mysql-01                              │
  │                                                          │
  │ Green database settings:                                 │
  │   DB engine version:    8.0.35                          │
  │   DB parameter group:   mysql-8-params                  │
  │                                                          │
  │ [Create Blue/Green Deployment]                           │
  └─────────────────────────────────────────────────────────┘

After green is ready (may take 30+ minutes):
  Actions → Switch over → Timeout: 300 seconds → [Switch over]
```

> ⚠️ During switchover, writes are briefly blocked (typically <1 minute). Plan for a maintenance window, even though it's fast.

---

## Part 10: Terraform & CLI Examples

```hcl
# RDS MySQL instance
resource "aws_db_instance" "mysql" {
  identifier     = "prod-mysql-01"
  engine         = "mysql"
  engine_version = "8.0.35"
  instance_class = "db.m6g.large"

  allocated_storage     = 100
  max_allocated_storage = 500
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = "myapp"
  username = "admin"
  password = var.db_password  # Use Secrets Manager in production

  multi_az               = true
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false

  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Sun:04:00-Sun:04:30"

  performance_insights_enabled = true
  deletion_protection          = true
  skip_final_snapshot          = false
  final_snapshot_identifier    = "prod-mysql-01-final"

  parameter_group_name = aws_db_parameter_group.mysql.name

  tags = { Environment = "prod" }
}

resource "aws_db_subnet_group" "main" {
  name       = "main"
  subnet_ids = [aws_subnet.private_a.id, aws_subnet.private_b.id]
}

resource "aws_db_parameter_group" "mysql" {
  name   = "prod-mysql-8"
  family = "mysql8.0"

  parameter {
    name  = "character_set_server"
    value = "utf8mb4"
  }

  parameter {
    name  = "slow_query_log"
    value = "1"
  }
}
```

```bash
# Create RDS instance
aws rds create-db-instance \
  --db-instance-identifier prod-mysql-01 \
  --engine mysql \
  --engine-version 8.0.35 \
  --db-instance-class db.m6g.large \
  --allocated-storage 100 \
  --master-username admin \
  --master-user-password "$DB_PASSWORD" \
  --multi-az \
  --storage-encrypted \
  --no-publicly-accessible \
  --vpc-security-group-ids sg-0abc123 \
  --db-subnet-group-name main

# Create read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-mysql-01-replica \
  --source-db-instance-identifier prod-mysql-01 \
  --db-instance-class db.r6g.large

# Create manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier prod-mysql-01 \
  --db-snapshot-identifier prod-mysql-01-pre-migration

# Restore to point in time
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-mysql-01 \
  --target-db-instance-identifier prod-mysql-01-restored \
  --restore-time "2024-01-15T14:30:00Z"

# Modify instance (e.g., scale up)
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-01 \
  --db-instance-class db.m6g.xlarge \
  --apply-immediately
```

---

## Part 11: Real-World Patterns

```
┌─────────────────────────────────────────────────────────────────────┐
│           REAL-WORLD RDS PATTERNS                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Pattern 1: Production Web Application                               │
│ ├── MySQL 8.0 on db.m6g.large, Multi-AZ                         │
│ ├── gp3 storage, 200 GB, autoscaling to 1 TB                    │
│ ├── 1 read replica for reporting queries                         │
│ ├── Secrets Manager for credentials (auto-rotation)             │
│ ├── RDS Proxy for connection pooling                             │
│ ├── Performance Insights enabled                                 │
│ ├── Automated backups, 14-day retention                          │
│ └── Deletion protection enabled                                   │
│                                                                       │
│ Pattern 2: Lambda + RDS                                             │
│ ├── RDS Proxy (essential — prevents connection exhaustion)       │
│ ├── Lambda connects to proxy endpoint                            │
│ ├── IAM database authentication via proxy                       │
│ ├── Lambda in VPC with access to RDS subnet                    │
│ └── Secrets Manager for initial credentials                      │
│                                                                       │
│ Pattern 3: Cross-Region DR                                          │
│ ├── Primary: us-east-1, Multi-AZ                                │
│ ├── Cross-region read replica in eu-west-1                       │
│ ├── Promote replica to standalone on failover                    │
│ ├── Application DNS (Route 53) points to active region          │
│ └── RPO: seconds, RTO: minutes                                   │
│                                                                       │
│ Pattern 4: Cost Optimization                                        │
│ ├── Reserved Instances for stable production DBs (up to 60% off)│
│ ├── Burstable (t3) for dev/test (avoid for production)          │
│ ├── Graviton instances (db.m6g) for 20% cost reduction         │
│ ├── Stop dev/test instances outside business hours               │
│ └── Right-size based on CloudWatch CPU/memory metrics            │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting: Common RDS Issues

### "Can't connect to RDS instance"

```
1. ☐ Is the RDS instance status "Available"?
   Console → RDS → Databases → check Status

2. ☐ Does the RDS security group allow inbound on the DB port?
   MySQL: 3306, PostgreSQL: 5432, SQL Server: 1433, Oracle: 1521
   Source: Your EC2 security group or IP address

3. ☐ Are the EC2 and RDS in the same VPC?
   Cross-VPC connections require VPC Peering + correct routing

4. ☐ Is "Publicly accessible" set correctly?
   For EC2 → RDS in same VPC: Set to No (recommended)
   For local development: Set to Yes + Security Group allows your IP

5. ☐ Check the endpoint URL (not the instance name!):
   Console → RDS → DB instance → Connectivity & security → Endpoint
   Example: mydb.abc123xyz.us-east-1.rds.amazonaws.com
```

### "RDS instance ran out of storage"

```
Symptom: Database is in "storage-full" state, connections fail

Fix:
1. Enable Storage Autoscaling (should have been on from the start!)
   Console → RDS → Modify → Enable storage autoscaling
   Set maximum storage threshold (e.g., 500 GB)

2. For immediate fix: Modify instance → increase Allocated storage
   (Takes effect immediately, may cause brief I/O pause)

Prevention:
- Enable Storage Autoscaling on creation
- Set CloudWatch alarm on FreeStorageSpace < 10 GB
```

### Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| No Multi-AZ for production | Single AZ failure = downtime | Enable Multi-AZ (2x cost, but essential) |
| Publicly accessible for prod | Exposed to internet attacks | Set to No, access via VPC only |
| No automated backups | Can't do PITR after data loss | Enable with 7-35 day retention |
| Stopped instance auto-restarts | Surprise billing after 7 days | Delete dev instances, use snapshots |
| No deletion protection | Accidental delete = disaster | Enable deletion protection on production |

---

## Quick Reference

```
RDS Quick Reference:
├── Engines: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server (+ Aurora)
├── Multi-AZ: Sync replication, auto-failover (60-120s or 35s cluster)
├── Read Replicas: Async, up to 5 (15 for Aurora), cross-region
├── Backups: Automated daily + transaction logs, 1-35 day retention
├── Point-in-time recovery: Any second within retention period
├── Encryption: AES-256 KMS at rest, SSL/TLS in transit
├── IAM auth: Temp tokens, no passwords in application code
├── RDS Proxy: Connection pooling, essential for Lambda
├── Performance Insights: Free 7-day, visual query analysis
├── Storage autoscaling: Grows automatically, set max threshold
└── ⚡ Always: Multi-AZ, encryption, automated backups, deletion protection
```

---

## What's Next?

In **Chapter 26: Aurora**, we'll cover Amazon Aurora — AWS's purpose-built relational database with MySQL/PostgreSQL compatibility, 5x performance, and features like Serverless v2, Global Database, and cloning.
