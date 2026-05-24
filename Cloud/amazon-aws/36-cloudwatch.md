# Chapter 36: CloudWatch

---

## Table of Contents

- [Overview](#overview)
- [Part 1: CloudWatch Fundamentals](#part-1-cloudwatch-fundamentals)
- [Part 2: CloudWatch Metrics](#part-2-cloudwatch-metrics)
- [Part 3: CloudWatch Alarms (Full Portal Walkthrough)](#part-3-cloudwatch-alarms-full-portal-walkthrough)
- [Part 4: CloudWatch Logs](#part-4-cloudwatch-logs)
- [Part 5: CloudWatch Dashboards](#part-5-cloudwatch-dashboards)
- [Part 6: CloudWatch Synthetics & RUM](#part-6-cloudwatch-synthetics--rum)
- [Part 7: CloudWatch Agent & Custom Metrics](#part-7-cloudwatch-agent--custom-metrics)
- [Part 8: Terraform & CLI Examples](#part-8-terraform--cli-examples)
- [Quick Reference](#quick-reference)
- [What's Next?](#whats-next)

---

## Overview

### What is CloudWatch? Why Do We Need Monitoring?

Imagine driving a car without a dashboard — no speedometer, no fuel gauge, no engine temperature light. You wouldn't know if you're going too fast, running out of gas, or about to overheat until something breaks.

**CloudWatch is the dashboard for your AWS infrastructure.** It shows you:
- 📊 **Metrics**: How much CPU is your EC2 using? How many errors is your API returning?
- 📝 **Logs**: What messages is your application printing? Any errors or warnings?
- 🚨 **Alarms**: "Alert me if CPU goes above 80%" or "Alert me if there are more than 10 errors in 5 minutes"
- 🤖 **Auto-actions**: "If CPU > 80% for 5 minutes, add more servers automatically"

**Without CloudWatch**, you won't know your app is down until users start complaining. **With CloudWatch**, you get alerted instantly and can even auto-fix problems.

Amazon CloudWatch is the monitoring and observability service for AWS. It collects metrics, logs, and events from AWS resources and applications, and lets you set alarms, create dashboards, and take automated actions.

```
What you'll learn:
├── CloudWatch Fundamentals (metrics, namespaces, dimensions)
├── Metrics (default vs custom, statistics, periods)
├── Alarms (full portal walkthrough)
├── Logs (log groups, insights, subscriptions)
├── Dashboards (visualization)
├── Synthetics & RUM (proactive monitoring)
├── CloudWatch Agent (OS-level metrics)
└── Terraform & CLI examples
```

---

## Part 1: CloudWatch Fundamentals

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUDWATCH ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ ┌──────────────────────────────────────────────────────────┐      │
│ │                    CloudWatch                             │      │
│ │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │      │
│ │  │ Metrics  │  │  Logs    │  │  Alarms  │  │Dashboard│ │      │
│ │  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │      │
│ │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │      │
│ │  │Synthetics│  │Events/EB │  │ Insights │  │  RUM    │ │      │
│ │  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │      │
│ └──────────────────────────────────────────────────────────┘      │
│       ▲              ▲              ▲              ▲               │
│  EC2, RDS,      Lambda,       Custom apps,    API calls          │
│  ALB, etc.      ECS, etc.     CloudWatch Agent                   │
│                                                                       │
│ Key concepts:                                                        │
│ ├── Namespace: Container for metrics (AWS/EC2, AWS/RDS, etc.)   │
│ ├── Metric: Time-ordered data points (CPUUtilization, etc.)     │
│ ├── Dimension: Key-value pair to identify a metric instance     │
│ │   (InstanceId=i-123, LoadBalancer=my-alb)                    │
│ ├── Statistic: Aggregation (Average, Sum, Min, Max, p99)       │
│ ├── Period: Time granularity (60s, 300s, 3600s)                │
│ └── Retention: 15 months (1-sec data kept 3 hours, etc.)       │
│                                                                       │
│ Data retention:                                                      │
│ ├── 1-second: Available for 3 hours                              │
│ ├── 60-second: Available for 15 days                             │
│ ├── 5-minute: Available for 63 days                              │
│ └── 1-hour: Available for 455 days (15 months)                  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: CloudWatch Metrics

```
┌─────────────────────────────────────────────────────────────────────┐
│           KEY DEFAULT METRICS BY SERVICE                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ EC2 (5-min default, 1-min with detailed monitoring):                │
│ ├── CPUUtilization (%)                                            │
│ ├── NetworkIn / NetworkOut (bytes)                                │
│ ├── DiskReadOps / DiskWriteOps                                   │
│ ├── StatusCheckFailed (instance + system)                        │
│ └── ⚠️ NO memory or disk space metrics by default               │
│     → Need CloudWatch Agent for RAM, disk, processes            │
│                                                                       │
│ ALB:                                                                 │
│ ├── RequestCount                                                  │
│ ├── TargetResponseTime                                            │
│ ├── HTTPCode_Target_2XX/4XX/5XX_Count                            │
│ ├── HealthyHostCount / UnHealthyHostCount                        │
│ └── ActiveConnectionCount                                         │
│                                                                       │
│ RDS:                                                                 │
│ ├── CPUUtilization, FreeableMemory                               │
│ ├── ReadIOPS / WriteIOPS                                          │
│ ├── DatabaseConnections                                           │
│ ├── FreeStorageSpace                                              │
│ └── ReplicaLag                                                    │
│                                                                       │
│ Lambda:                                                              │
│ ├── Invocations, Errors, Throttles                               │
│ ├── Duration (ms)                                                 │
│ ├── ConcurrentExecutions                                          │
│ └── IteratorAge (for stream-based invocations)                   │
│                                                                       │
│ DynamoDB:                                                            │
│ ├── ConsumedReadCapacityUnits / ConsumedWriteCapacityUnits      │
│ ├── ThrottledRequests                                             │
│ ├── SuccessfulRequestLatency                                     │
│ └── ReturnedItemCount                                             │
│                                                                       │
│ Custom Metrics:                                                      │
│ aws cloudwatch put-metric-data \                                   │
│   --namespace "MyApp" \                                             │
│   --metric-name "ActiveUsers" \                                    │
│   --value 42 \                                                      │
│   --dimensions "Environment=prod,Service=web"                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 3: CloudWatch Alarms (Full Portal Walkthrough)

```
Console → CloudWatch → Alarms → Create alarm

┌─────────────────────────────────────────────────────────────────┐
│           STEP 1: SPECIFY METRIC AND CONDITIONS                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Select metric ──                                             │
│ Browse: EC2 → Per-Instance Metrics                             │
│ Search: [CPUUtilization]                                       │
│                                                                   │
│ Select metric:                                                  │
│ ☑ InstanceId=i-0abc123  CPUUtilization                        │
│                                                                   │
│ ── Metric ──                                                    │
│ Metric name: CPUUtilization                                    │
│ InstanceId: i-0abc123                                          │
│                                                                   │
│ Statistic: [Average ▼]                                        │
│   ├── Average (⚡ most common for CPU)                        │
│   ├── Sum (use for counts like errors)                        │
│   ├── Minimum / Maximum                                       │
│   ├── SampleCount                                              │
│   └── Percentile: p99, p95, p90                               │
│                                                                   │
│ Period: [5 minutes ▼]                                         │
│   ├── 1 minute (detailed monitoring or custom metrics)       │
│   ├── 5 minutes (⚡ default)                                  │
│   └── 15 minutes, 1 hour, 6 hours, 1 day                    │
│                                                                   │
│ ── Conditions ──                                                │
│ Threshold type:                                                │
│ ● Static  ○ Anomaly detection                                 │
│                                                                   │
│ → Static: Fixed threshold (e.g., CPU > 80%)                  │
│ → Anomaly detection: ML-based band (detects unusual)         │
│                                                                   │
│ Whenever CPUUtilization is...                                  │
│ ● Greater than  [80]                                          │
│ ○ Greater/Equal than  [threshold]                             │
│ ○ Lower/Equal than  [threshold]                               │
│ ○ Lower than  [threshold]                                     │
│                                                                   │
│ Datapoints to alarm: [3] out of [5]                           │
│ → 3 out of 5 consecutive periods must breach threshold       │
│ → Prevents false alarms from short spikes                    │
│ → ⚡ 3/5 is a good production default                        │
│                                                                   │
│ Missing data treatment:                                        │
│ ● Treat missing data as missing (⚡ recommended)              │
│ ○ Treat as breaching                                          │
│ ○ Treat as not breaching                                      │
│ ○ Treat as within threshold                                   │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 2: CONFIGURE ACTIONS                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ ── Alarm state trigger ──                                      │
│ In alarm:                                                       │
│ ☑ Send notification to SNS topic                              │
│   Topic: [prod-alerts ▼] or Create new topic                  │
│   Email: [ops-team@company.com]                                │
│                                                                   │
│ ☑ EC2 action (for EC2 alarms only):                           │
│   ○ Stop  ○ Terminate  ○ Reboot  ● Recover                  │
│   → Recover: Migrates instance to new host (same IP/EBS)    │
│                                                                   │
│ ☑ Auto Scaling action:                                        │
│   Scaling policy: [scale-up-policy ▼]                         │
│                                                                   │
│ ☑ Systems Manager action:                                     │
│   → Run SSM Automation document                              │
│                                                                   │
│ OK state:                                                       │
│ ☑ Send notification when alarm returns to OK                  │
│   Topic: [prod-alerts ▼]                                      │
│                                                                   │
│ Insufficient data:                                              │
│ ☐ Send notification (usually leave unchecked)                 │
│                                                                   │
│                    [Next]                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           STEP 3: NAME AND DESCRIPTION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Alarm name: [prod-web-cpu-high]                                │
│ Alarm description: [CPU > 80% for 15 min on prod web server] │
│                                                                   │
│                    [Next → Review → Create alarm]               │
└─────────────────────────────────────────────────────────────────┘

Composite Alarms:
Console → CloudWatch → Alarms → Create composite alarm

┌─────────────────────────────────────────────────────────────────┐
│ Combine multiple alarms with AND / OR logic:                    │
│                                                                   │
│ ALARM("prod-cpu-high") AND ALARM("prod-5xx-high")             │
│ → Only triggers if BOTH conditions are true                   │
│ → Reduces alert noise                                         │
│                                                                   │
│ ⚡ Use composite alarms for complex alerting rules              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 4: CloudWatch Logs

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUDWATCH LOGS                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Hierarchy:                                                           │
│ ├── Log Group: Container (e.g., /aws/lambda/my-function)        │
│ │   ├── Retention: 1 day to 10 years (or never expire)         │
│ │   ├── KMS encryption                                           │
│ │   └── Metric filters                                           │
│ ├── Log Stream: Sequence from one source (container, instance) │
│ └── Log Event: Single log line with timestamp                   │
│                                                                       │
│ Sources (automatic):                                                 │
│ ├── Lambda → /aws/lambda/{function-name}                        │
│ ├── API Gateway → /aws/apigateway/{api-name}                   │
│ ├── ECS → /ecs/{service-name}                                   │
│ ├── VPC Flow Logs → custom group                                │
│ └── Route 53 DNS queries → /aws/route53/{zone-id}              │
│                                                                       │
│ Sources (agent required):                                            │
│ ├── EC2 instance logs → install CloudWatch Agent                │
│ └── On-premises servers → install CW Agent                     │
│                                                                       │
│ ── Logs Insights (query language) ──                               │
│ Console → CloudWatch → Logs → Logs Insights                       │
│                                                                       │
│ fields @timestamp, @message                                        │
│ | filter @message like /ERROR/                                     │
│ | sort @timestamp desc                                              │
│ | limit 50                                                          │
│                                                                       │
│ # Top 10 error messages                                             │
│ fields @message                                                      │
│ | filter @message like /ERROR/                                     │
│ | stats count(*) as errorCount by @message                        │
│ | sort errorCount desc                                              │
│ | limit 10                                                          │
│                                                                       │
│ # Lambda duration statistics                                        │
│ filter @type = "REPORT"                                             │
│ | stats avg(@duration), max(@duration), min(@duration)             │
│   by bin(1h)                                                        │
│                                                                       │
│ ── Metric Filters ──                                                │
│ Create a CloudWatch metric from log pattern:                        │
│ Console → Log group → Metric filters → Create                     │
│ Filter pattern: [ERROR]                                              │
│ Metric namespace: MyApp                                              │
│ Metric name: ErrorCount                                              │
│ Metric value: 1                                                      │
│ → Count errors and alarm on them                                  │
│                                                                       │
│ ── Subscription Filters ──                                          │
│ Stream logs to:                                                      │
│ ├── Lambda (real-time processing)                                │
│ ├── Kinesis Data Firehose (→ S3, Redshift, Elasticsearch)      │
│ ├── Kinesis Data Streams (custom processing)                    │
│ └── Cross-account log sharing                                    │
│                                                                       │
│ ── Log Export ──                                                    │
│ Export to S3: Console → Log group → Actions → Export data to S3  │
│ → For long-term archival (cheaper than CW retention)             │
│ → ⚡ Use S3 Lifecycle to move to Glacier after 90 days           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 5: CloudWatch Dashboards

```
Console → CloudWatch → Dashboards → Create dashboard

┌─────────────────────────────────────────────────────────────────┐
│           DASHBOARDS                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Dashboard name: [prod-overview]                                │
│                                                                   │
│ Add widget:                                                     │
│ ├── Line: Time series (CPU over time)                         │
│ ├── Stacked area: Cumulative metrics                          │
│ ├── Number: Single value (current error count)                │
│ ├── Gauge: Percentage display (CPU utilization)               │
│ ├── Bar: Comparison bar chart                                 │
│ ├── Pie: Distribution chart                                    │
│ ├── Text: Markdown notes                                       │
│ ├── Logs table: Query results                                 │
│ ├── Alarm status: Show alarm states                           │
│ └── Explorer: Dynamic resource group metrics                 │
│                                                                   │
│ Widget settings:                                                │
│ ├── Title: [Web Server CPU]                                   │
│ ├── Metrics: Select from any namespace/metric                │
│ ├── Statistic: Average, Sum, etc.                             │
│ ├── Period: 1 min to 1 day                                    │
│ ├── Y-axis: Auto or fixed range                               │
│ ├── Annotations: Horizontal line at threshold (80%)          │
│ └── Auto-refresh: 10s, 1m, 2m, 5m, 15m                      │
│                                                                   │
│ Cross-account dashboards:                                      │
│ → Enable cross-account observability                          │
│ → Monitoring account sees metrics from source accounts       │
│ → ⚡ Centralized monitoring for Organizations                 │
│                                                                   │
│ Pricing: $3/month per dashboard (50 metrics free)             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Part 6: CloudWatch Synthetics & RUM

```
┌─────────────────────────────────────────────────────────────────────┐
│           SYNTHETICS (CANARIES)                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Console → CloudWatch → Synthetics → Create canary                  │
│                                                                       │
│ What: Automated scripts that test your endpoints on a schedule.  │
│ Like a robot checking your website/API every 5 minutes.           │
│                                                                       │
│ Blueprint:                                                           │
│ ├── Heartbeat monitoring: Simple URL health check               │
│ ├── API canary: Test REST API endpoints                         │
│ ├── Broken link checker: Find broken links on page             │
│ ├── Visual monitoring: Screenshot comparison (detects UI change)│
│ ├── Canary recorder: Record browser actions (Chrome extension) │
│ └── Custom script: Node.js or Python                            │
│                                                                       │
│ Name: [prod-website-check]                                         │
│ URL: [https://www.myapp.com]                                       │
│ Schedule: Every [5] minutes                                        │
│ Data retention: [31] days                                           │
│ S3 bucket for artifacts: [cw-synthetics-results ▼]                │
│                                                                       │
│ → Screenshots, HAR files, logs stored in S3                      │
│ → Creates CloudWatch metrics (SuccessPercent, Duration)          │
│ → Set alarms on canary failure                                    │
│ → ⚡ Catch outages before customers report them                  │
│                                                                       │
│ ── RUM (Real User Monitoring) ──                                   │
│ Console → CloudWatch → RUM → Add app monitor                      │
│                                                                       │
│ What: JavaScript snippet in your web app that reports real user  │
│ performance data back to CloudWatch.                               │
│                                                                       │
│ Captures:                                                            │
│ ├── Page load time, largest contentful paint                    │
│ ├── JavaScript errors                                            │
│ ├── HTTP errors (4xx, 5xx)                                       │
│ ├── Session performance                                          │
│ └── User journey (page navigation)                               │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7: CloudWatch Agent & Custom Metrics

```
┌─────────────────────────────────────────────────────────────────────┐
│           CLOUDWATCH AGENT                                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│ Why: EC2 default metrics don't include memory, disk, processes.  │
│ The CloudWatch Agent runs on EC2 and collects OS-level data.      │
│                                                                       │
│ Install:                                                             │
│ # Amazon Linux 2                                                    │
│ sudo yum install amazon-cloudwatch-agent                           │
│                                                                       │
│ # Run wizard to generate config                                    │
│ sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-      │
│   agent-config-wizard                                               │
│                                                                       │
│ # Store config in SSM Parameter Store                               │
│ aws ssm put-parameter --name "CWAgentConfig" --type String \     │
│   --value file://config.json                                       │
│                                                                       │
│ # Start agent with SSM config                                      │
│ sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-      │
│   agent-ctl -a fetch-config -m ec2 \                               │
│   -c ssm:CWAgentConfig -s                                          │
│                                                                       │
│ Agent collects:                                                      │
│ ├── Memory: mem_used_percent (⚡ most important)                │
│ ├── Disk: disk_used_percent, disk_free                          │
│ ├── CPU: cpu_usage_user, cpu_usage_system (per-core)           │
│ ├── Network: netstat_tcp_established                            │
│ ├── Processes: processes_total, processes_running               │
│ └── Custom logs: Tail log files → CloudWatch Logs              │
│                                                                       │
│ ⚡ At scale: Use SSM to install/configure CW Agent across fleet  │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 7.5: Additional CloudWatch Features

### Container Insights (ECS/EKS Monitoring)

Container Insights provides **automatic container-level metrics** for ECS and EKS clusters without any application code changes.

```
What you get:
├── CPU/Memory utilization per container, task, service, cluster
├── Network I/O per container
├── Storage utilization
├── Running task count per service
├── Automatic CloudWatch dashboards
└── Integration with CloudWatch Logs for container logs

Enable for ECS:
Console → ECS → Account Settings → CloudWatch Container Insights → Enable

Enable for EKS:
Install the CloudWatch agent as a DaemonSet on your cluster
(AWS provides a one-click setup via CloudWatch → Insights → Container Insights)
```

> 💡 Container Insights costs ~$0.30 per task per month. Worth it for production, consider disabling in dev.

### Contributor Insights

Contributor Insights identifies your **top-N contributors** for any CloudWatch Logs data. Perfect for finding:
- Top 10 most accessed API endpoints
- Top IPs generating errors
- Busiest DynamoDB partition keys (hot partitions)

```
Console → CloudWatch → Insights → Contributor Insights → Create rule

  Log group: /aws/apigateway/prod
  Contribution field: $.ip
  Filter: $.status = 500
  
  → Shows: Top 10 IPs causing 500 errors in real-time
```

### Application Insights

Application Insights **auto-detects** common application problems and correlates metrics across services. It's designed for .NET and SQL Server workloads but works with any technology.

```
Console → CloudWatch → Insights → Application Insights → Add application
  → Auto-discovers your EC2, RDS, ELB resources
  → Creates monitoring dashboards and alerts automatically
  → Detects anomalies using ML
```

---

## Part 8: Terraform & CLI Examples

```hcl
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "prod-web-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "CPU > 80% for 15 minutes"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]

  dimensions = {
    InstanceId = aws_instance.web.id
  }

  datapoints_to_alarm = 3
  treat_missing_data  = "missing"
}

resource "aws_cloudwatch_log_group" "app" {
  name              = "/app/web-server"
  retention_in_days = 30
  kms_key_id        = aws_kms_key.logs.arn
}

resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "prod-overview"
  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          metrics = [["AWS/EC2", "CPUUtilization", "InstanceId", "i-123"]]
          period  = 300
          stat    = "Average"
          title   = "EC2 CPU"
        }
      }
    ]
  })
}
```

```bash
# List metrics
aws cloudwatch list-metrics --namespace AWS/EC2

# Get metric statistics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Average

# Put custom metric
aws cloudwatch put-metric-data \
  --namespace MyApp \
  --metric-name OrderCount \
  --value 42 \
  --unit Count

# Set alarm state (testing)
aws cloudwatch set-alarm-state \
  --alarm-name prod-web-cpu-high \
  --state-value ALARM \
  --state-reason "Testing alarm"
```

---

## Quick Reference

```
CloudWatch Quick Reference:
├── Metrics: Default (5 min) or Detailed (1 min, extra cost)
├── ⚠️ EC2: No memory/disk by default → need CW Agent
├── Alarms: Threshold or anomaly detection
├── Alarms → Actions: SNS, Auto Scaling, EC2, SSM
├── Composite alarms: Combine with AND/OR logic
├── Logs: Log Groups → Log Streams → Log Events
├── Logs Insights: SQL-like query language
├── Metric filters: Create metrics from log patterns
├── Dashboards: Visualization ($3/month each)
├── Synthetics: Automated endpoint testing (canaries)
├── RUM: Real user monitoring (JS snippet)
├── Agent: OS-level metrics (memory, disk)
└── Retention: 1 day to 10 years (or never expire)
```

---

## What's Next?

In **Chapter 37: CloudTrail**, we'll cover API audit logging — tracking every API call made in your AWS account for security, compliance, and troubleshooting.
