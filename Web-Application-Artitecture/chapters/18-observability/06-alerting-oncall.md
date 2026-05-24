# Alerting & On-Call — PagerDuty, OpsGenie

> **What you'll learn**: How to set up intelligent alerts that notify the right person at the right time, manage on-call rotations, and build an alerting system that catches real problems without drowning you in noise.

---

## Real-Life Analogy

Think of a **fire alarm system** in a building:

- **Good fire alarm**: Detects actual fires, sounds clearly, tells you WHICH floor and room, connects to the fire department automatically.
- **Bad fire alarm**: Goes off every time someone makes toast, rings at 3 AM for a false alarm, nobody knows what to do when it rings, eventually people ignore it entirely.

Your monitoring alerts work the same way:
- **Good alerts** = fire when something genuinely needs human attention, tell you exactly what's wrong and what to do
- **Bad alerts** = fire constantly for minor issues, wake people up for things that fix themselves, create "alert fatigue" where everyone ignores ALL alerts

The goal: **Every alert should mean "a human needs to act NOW."**

---

## The Alerting Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    ALERTING PIPELINE                              │
│                                                                   │
│  ┌──────────┐   ┌───────────┐   ┌──────────┐   ┌───────────┐  │
│  │  Metrics │──▶│   Alert   │──▶│  Alert   │──▶│Notification│  │
│  │  Source  │   │   Rules   │   │  Router  │   │  Channel   │  │
│  └──────────┘   └───────────┘   └──────────┘   └───────────┘  │
│                                                                   │
│  Prometheus     "error_rate     "critical →    PagerDuty         │
│  Datadog          > 5% for      on-call eng"  Slack              │
│  CloudWatch       5 minutes"   "warning →     Email              │
│                                  #alerts       OpsGenie           │
│                                  channel"      Teams              │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                                                             │  │
│  │  ALERT LIFECYCLE:                                          │  │
│  │                                                             │  │
│  │  [Inactive] → [Pending] → [Firing] → [Acknowledged] →    │  │
│  │                              │          [Resolved]         │  │
│  │                              │                              │  │
│  │                              └─── (auto-resolve if         │  │
│  │                                     condition clears)      │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Alert Severity Levels

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEVERITY LEVELS                                │
│                                                                   │
│  ┌─────────────┬──────────────────────────────────────────────┐ │
│  │  CRITICAL   │  Service is DOWN. Users are affected NOW.     │ │
│  │  (P1/Sev1)  │  Action: Page on-call engineer IMMEDIATELY   │ │
│  │  🔴         │  Example: Payment service 100% errors        │ │
│  ├─────────────┼──────────────────────────────────────────────┤ │
│  │  HIGH       │  Service is degraded. Some users affected.    │ │
│  │  (P2/Sev2)  │  Action: Page during business hours,         │ │
│  │  🟠         │          Slack during off-hours              │ │
│  │             │  Example: p99 latency > 5s for 10 min       │ │
│  ├─────────────┼──────────────────────────────────────────────┤ │
│  │  WARNING    │  Something unusual. Not impacting users yet. │ │
│  │  (P3/Sev3)  │  Action: Slack notification, fix next day    │ │
│  │  🟡         │  Example: Disk 80% full, will fill in 2 days│ │
│  ├─────────────┼──────────────────────────────────────────────┤ │
│  │  INFO       │  Informational. No action needed now.         │ │
│  │  (P4/Sev4)  │  Action: Log only, review in weekly meeting │ │
│  │  🔵         │  Example: Deployment completed successfully  │ │
│  └─────────────┴──────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## On-Call Management

### What is On-Call?

On-call means someone is **designated as the first responder** for production issues during a specific time window.

```
ON-CALL ROTATION (Example: Weekly):

┌─────────────────────────────────────────────────────────────────┐
│  TEAM: Payment Service                                           │
│                                                                   │
│  Week 1: Alice (Primary) + Bob (Secondary)                      │
│  Week 2: Bob (Primary) + Carol (Secondary)                      │
│  Week 3: Carol (Primary) + Dave (Secondary)                     │
│  Week 4: Dave (Primary) + Alice (Secondary)                     │
│                                                                   │
│  What happens when an alert fires:                              │
│                                                                   │
│  [Alert Fires] ──▶ [Page Primary (Alice)]                      │
│                         │                                        │
│                    (no response in 5 min?)                       │
│                         │                                        │
│                         ▼                                        │
│                    [Page Secondary (Bob)]                        │
│                         │                                        │
│                    (no response in 10 min?)                      │
│                         │                                        │
│                         ▼                                        │
│                    [Page Engineering Manager]                    │
│                         │                                        │
│                    (still no response?)                          │
│                         │                                        │
│                         ▼                                        │
│                    [Page VP Engineering] 😱                      │
└─────────────────────────────────────────────────────────────────┘
```

### Escalation Policies

```
ESCALATION EXAMPLE (PagerDuty/OpsGenie):

Minute 0:  Page primary on-call via phone + SMS + push notification
Minute 5:  If not acknowledged → page secondary on-call
Minute 10: If not acknowledged → page team lead
Minute 15: If not acknowledged → page engineering director
Minute 30: If not resolved → create incident, add to bridge call
```

---

## How It Works Internally

### Alert Evaluation Flow (Prometheus + Alertmanager)

```
┌─────────────────────────────────────────────────────────────────┐
│               PROMETHEUS ALERTING FLOW                            │
│                                                                   │
│  Step 1: RULE EVALUATION (every 15 seconds)                     │
│  ─────────────────────────────────────────                      │
│  Prometheus evaluates:                                           │
│    rate(http_requests_total{status="500"}[5m])                  │
│    / rate(http_requests_total[5m]) > 0.05                       │
│                                                                   │
│  Result: 0.08 (8% error rate) > 0.05 threshold → ALERT!       │
│                                                                   │
│  Step 2: PENDING (wait for "for" duration)                      │
│  ─────────────────────────────────────────                      │
│  Rule says: for: 5m                                              │
│  → Alert stays "pending" for 5 minutes                          │
│  → If condition clears before 5m, alert is cancelled            │
│  → This prevents alerting on brief spikes                       │
│                                                                   │
│  Step 3: FIRING (send to Alertmanager)                          │
│  ─────────────────────────────────────────                      │
│  After 5 minutes of continuous violation:                        │
│  → Alert state changes to "firing"                              │
│  → Sent to Alertmanager                                          │
│                                                                   │
│  Step 4: ALERTMANAGER processes                                  │
│  ─────────────────────────────────────────                      │
│  • Deduplication: Same alert from 3 Prometheus? Send once.      │
│  • Grouping: 50 alerts from same service? Send one summary.     │
│  • Silencing: Planned maintenance? Suppress alerts.              │
│  • Routing: Critical → PagerDuty, Warning → Slack              │
│  • Inhibition: If cluster is down, don't alert per-service     │
│                                                                   │
│  Step 5: NOTIFICATION                                            │
│  ─────────────────────────────────────────                      │
│  → PagerDuty/OpsGenie receives                                  │
│  → Pages on-call engineer                                        │
│  → Engineer acknowledges, investigates, resolves                │
└─────────────────────────────────────────────────────────────────┘
```

### Alertmanager Routing

```yaml
# alertmanager.yml - Smart routing of alerts
global:
  resolve_timeout: 5m

route:
  # Default: send to Slack
  receiver: 'slack-notifications'
  group_by: ['alertname', 'service']
  group_wait: 30s        # Wait 30s to group similar alerts
  group_interval: 5m     # Re-send group every 5 min if still firing
  repeat_interval: 4h    # Don't repeat same alert more than every 4h
  
  routes:
    # Critical alerts → PagerDuty (wake people up!)
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      repeat_interval: 15m
    
    # Warning alerts → Slack channel
    - match:
        severity: warning
      receiver: 'slack-warnings'
    
    # Database alerts → DBA team
    - match:
        component: database
      receiver: 'dba-team-pagerduty'

receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: '<PAGERDUTY_SERVICE_KEY>'
        severity: critical
  
  - name: 'slack-notifications'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/...'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ .CommonAnnotations.summary }}'
  
  - name: 'slack-warnings'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/...'
        channel: '#alerts-low-priority'

# Silence alerts during maintenance
inhibit_rules:
  - source_match:
      alertname: 'ClusterDown'
    target_match_re:
      alertname: '.+'  # If cluster is down, suppress all other alerts
    equal: ['cluster']
```

---

## Code Examples

### Python: Custom Alert Manager Integration

```python
# alert_webhook_handler.py
# Handle incoming alerts from Prometheus Alertmanager via webhook

from flask import Flask, request, jsonify
import requests
from datetime import datetime

app = Flask(__name__)

# Alertmanager sends alerts to this webhook
@app.route('/webhook/alerts', methods=['POST'])
def handle_alert():
    """Process alerts from Alertmanager and route them."""
    alerts = request.json.get('alerts', [])
    
    for alert in alerts:
        alert_name = alert['labels'].get('alertname', 'Unknown')
        severity = alert['labels'].get('severity', 'info')
        status = alert['status']  # "firing" or "resolved"
        service = alert['labels'].get('service', 'unknown')
        summary = alert['annotations'].get('summary', 'No summary')
        
        print(f"[{status.upper()}] {alert_name} ({severity}) - {summary}")
        
        if status == 'firing':
            if severity == 'critical':
                # Page on-call immediately
                page_oncall(alert_name, summary, service)
                # Create incident ticket
                create_incident(alert_name, summary, service)
            elif severity == 'warning':
                # Send to Slack
                send_slack_alert(alert_name, summary, service)
        
        elif status == 'resolved':
            # Notify that alert is resolved
            send_slack_resolved(alert_name, service)
    
    return jsonify({"status": "processed"}), 200

def page_oncall(alert_name: str, summary: str, service: str):
    """Send page via PagerDuty API."""
    requests.post(
        "https://events.pagerduty.com/v2/enqueue",
        json={
            "routing_key": "YOUR_PAGERDUTY_ROUTING_KEY",
            "event_action": "trigger",
            "payload": {
                "summary": f"[{service}] {alert_name}: {summary}",
                "severity": "critical",
                "source": service,
                "timestamp": datetime.utcnow().isoformat(),
                "custom_details": {
                    "runbook": f"https://wiki.company.com/runbooks/{alert_name}",
                    "dashboard": f"https://grafana.company.com/d/{service}"
                }
            }
        }
    )

def send_slack_alert(alert_name: str, summary: str, service: str):
    """Send alert to Slack channel."""
    requests.post(
        "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK",
        json={
            "text": f"⚠️ *{alert_name}* ({service})\n{summary}\n"
                    f"<https://grafana.company.com/d/{service}|View Dashboard>"
        }
    )

if __name__ == '__main__':
    app.run(port=9095)
```

### Java: PagerDuty Integration for Custom Alerting

```java
// AlertService.java - Custom alerting with PagerDuty Events API v2

import com.fasterxml.jackson.databind.ObjectMapper;
import java.net.http.*;
import java.time.Instant;
import java.util.Map;

public class AlertService {
    
    private static final String PAGERDUTY_URL = "https://events.pagerduty.com/v2/enqueue";
    private final String routingKey;
    private final HttpClient httpClient;
    private final ObjectMapper objectMapper;
    
    public AlertService(String routingKey) {
        this.routingKey = routingKey;
        this.httpClient = HttpClient.newHttpClient();
        this.objectMapper = new ObjectMapper();
    }
    
    /**
     * Triggers a PagerDuty incident for critical alerts.
     */
    public void triggerAlert(String service, String alertName, 
                            String summary, String severity) {
        var payload = Map.of(
            "routing_key", routingKey,
            "event_action", "trigger",
            "dedup_key", service + "-" + alertName, // Prevents duplicate incidents
            "payload", Map.of(
                "summary", String.format("[%s] %s: %s", service, alertName, summary),
                "severity", severity, // critical, error, warning, info
                "source", service,
                "timestamp", Instant.now().toString(),
                "custom_details", Map.of(
                    "runbook_url", "https://wiki.company.com/runbooks/" + alertName,
                    "dashboard_url", "https://grafana.company.com/d/" + service,
                    "service", service
                )
            )
        );
        
        try {
            String body = objectMapper.writeValueAsString(payload);
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(PAGERDUTY_URL))
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(body))
                .build();
            
            HttpResponse<String> response = httpClient.send(
                request, HttpResponse.BodyHandlers.ofString());
            
            if (response.statusCode() != 202) {
                System.err.println("PagerDuty alert failed: " + response.body());
            }
        } catch (Exception e) {
            System.err.println("Failed to send PagerDuty alert: " + e.getMessage());
        }
    }
    
    /**
     * Resolves a previously triggered incident.
     */
    public void resolveAlert(String service, String alertName) {
        var payload = Map.of(
            "routing_key", routingKey,
            "event_action", "resolve",
            "dedup_key", service + "-" + alertName
        );
        // ... send HTTP request same as above
    }
}

// Usage in health check monitoring:
// AlertService alerts = new AlertService("YOUR_ROUTING_KEY");
// if (errorRate > 0.05) {
//     alerts.triggerAlert("payment-service", "HighErrorRate", 
//         "Error rate is 8%, threshold 5%", "critical");
// }
```

---

## Alerting Best Practices

### The Alerting Golden Rules

```
┌─────────────────────────────────────────────────────────────────┐
│               ALERTING GOLDEN RULES                               │
│                                                                   │
│  1. EVERY ALERT MUST BE ACTIONABLE                              │
│     If you get paged, there must be something you can DO.       │
│     Bad: "CPU at 82%" (so what?)                                │
│     Good: "Error rate > 5% — users affected, check payment svc"│
│                                                                   │
│  2. ALERT ON SYMPTOMS, NOT CAUSES                               │
│     Bad: "High CPU" (might not affect users)                    │
│     Good: "Response time > 2s" (users ARE affected)             │
│                                                                   │
│  3. EVERY ALERT NEEDS A RUNBOOK                                 │
│     Include a link to: "When this fires, do THIS"               │
│     At 3 AM, you want step-by-step instructions                 │
│                                                                   │
│  4. SET APPROPRIATE THRESHOLDS                                   │
│     Too sensitive = too many alerts = alert fatigue              │
│     Too loose = miss real problems                              │
│     Start loose, tighten based on real incidents                │
│                                                                   │
│  5. USE "FOR" DURATION TO AVOID FLAPPING                        │
│     "Error rate > 5% FOR 5 MINUTES"                             │
│     Brief spikes that self-resolve shouldn't page anyone        │
│                                                                   │
│  6. PAGE ONLY FOR CRITICAL (USER-FACING IMPACT)                 │
│     Everything else → Slack/email during business hours          │
└─────────────────────────────────────────────────────────────────┘
```

### Alert Fatigue: The Silent Killer

```
THE ALERT FATIGUE DEATH SPIRAL:

Step 1: Team sets up 200 alerts "just in case"
        ↓
Step 2: 50 alerts fire daily (most are noise)
        ↓
Step 3: Engineers start ignoring alerts
        ↓
Step 4: Real critical alert fires
        ↓
Step 5: Nobody notices → 2-hour outage
        ↓
Step 6: "We need MORE alerts!" → Go to Step 1

THE FIX:
  • Audit alerts quarterly
  • Delete alerts nobody acts on
  • Target: < 2 pages per on-call shift
  • Every alert fired should result in action OR be deleted
```

---

## Real-World Example

### How Google Manages On-Call (SRE Practices)

```
┌─────────────────────────────────────────────────────────────────┐
│              GOOGLE SRE ON-CALL PRACTICES                         │
│                                                                   │
│  Rules:                                                          │
│  • Maximum 2 incidents per 12-hour shift (target)              │
│  • 25% of on-call time spent on operational work (max)         │
│  • Remaining 75% on engineering improvements                    │
│  • If too many pages → team must fix root causes               │
│  • 8-12 hour shifts (never 24-hour)                            │
│                                                                   │
│  Escalation:                                                    │
│  Primary on-call ──(5 min)──▶ Secondary ──(10 min)──▶ Manager │
│                                                                   │
│  Post-Incident:                                                 │
│  • Blameless postmortem for every Sev1/Sev2                    │
│  • Action items tracked to completion                          │
│  • "What broke?" + "How to prevent next time?"                │
│                                                                   │
│  Compensation:                                                  │
│  • Extra time off after on-call shifts                         │
│  • On-call bonus pay                                           │
│  • Team rotation ensures fair distribution                     │
└─────────────────────────────────────────────────────────────────┘
```

### How PagerDuty Works Under the Hood

```
┌─────────────────────────────────────────────────────────────────┐
│                PAGERDUTY FLOW                                     │
│                                                                   │
│  [Your Monitoring] ──event──▶ [PagerDuty]                       │
│                                    │                              │
│                              ┌─────┼─────┐                       │
│                              │     │     │                        │
│                              ▼     ▼     ▼                        │
│                           Phone  SMS   Push                      │
│                           Call         Notification              │
│                              │     │     │                        │
│                              └─────┼─────┘                       │
│                                    │                              │
│                                    ▼                              │
│                        ┌─── Engineer Response ───┐               │
│                        │                          │               │
│                        ▼                          ▼               │
│                   [Acknowledge]            [No Response]          │
│                   (I'm on it!)            (5 min timeout)        │
│                        │                          │               │
│                        ▼                          ▼               │
│                   [Investigate]           [Escalate to           │
│                   [Resolve]               Secondary]              │
│                                                                   │
│  Features:                                                       │
│  • Multi-channel: phone, SMS, push, email, Slack                │
│  • Smart routing based on schedules                             │
│  • Escalation policies (auto-escalate if no ack)               │
│  • Analytics (MTTA, MTTR, pages per person)                     │
│  • Postmortem management                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Infrastructure Example: Complete Alerting Setup

```yaml
# Prometheus alert rules (what to alert on)
# alert_rules.yml
groups:
  - name: critical_alerts
    rules:
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Service {{ $labels.job }} is DOWN"
          runbook: "https://wiki.company.com/runbooks/service-down"
          dashboard: "https://grafana.company.com/d/service-health"
      
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.service }} error rate: {{ $value | humanizePercentage }}"
          runbook: "https://wiki.company.com/runbooks/high-error-rate"
      
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.service }} p99 latency > 3s"

  - name: capacity_alerts
    rules:
      - alert: DiskSpaceRunningOut
        expr: |
          predict_linear(node_filesystem_avail_bytes[6h], 24*60*60) < 0
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Disk will be full in ~24 hours on {{ $labels.instance }}"
      
      - alert: MemoryPressure
        expr: |
          (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
          / node_memory_MemTotal_bytes > 0.9
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage > 90% on {{ $labels.instance }}"
```

---

## Common Mistakes / Pitfalls

| Mistake | Impact | Fix |
|---------|--------|-----|
| **Alerting on every metric threshold** | 100+ alerts/day, alert fatigue | Only alert on user-facing impact |
| **No "for" duration** | Brief spikes trigger pages at 3 AM | Set `for: 5m` minimum for most alerts |
| **No runbook linked** | Engineer at 3 AM doesn't know what to do | Every alert MUST link to instructions |
| **Same person always on-call** | Burnout, single point of failure | Fair rotation, limit shifts to 1 week |
| **No escalation policy** | Alert gets missed if person is asleep | Auto-escalate after 5 min no-ack |
| **Alerting on causes not symptoms** | High CPU alert but users are fine | Alert on error rate, latency (user impact) |
| **Never deleting old alerts** | Noise accumulates over time | Quarterly alert audit — delete unused alerts |
| **No postmortems** | Same problems recur repeatedly | Blameless postmortem after every Sev1 |

---

## When to Use / When NOT to Use

### Page the On-Call Engineer (Phone/SMS) When:
- ✅ Users are currently impacted (errors, downtime)
- ✅ Data loss is occurring or imminent
- ✅ Revenue is being lost
- ✅ Security breach detected
- ✅ The issue won't self-resolve

### Send to Slack/Email (Don't Page) When:
- ✅ Issue is not user-facing yet (disk filling up)
- ✅ Problem will likely self-resolve (brief spike)
- ✅ Can wait until business hours
- ✅ Informational (deployment completed, scaling event)

### Don't Alert At All When:
- ❌ It's expected behavior (planned maintenance)
- ❌ No one would take action
- ❌ It's a metric for a dashboard only
- ❌ It happens every day and nobody cares

---

## Key Takeaways

1. **Every alert must be actionable** — if getting paged doesn't require immediate human action, it shouldn't be a page.

2. **Alert on symptoms (user impact), not causes** — "Error rate > 5%" beats "CPU > 80%" because it tells you users are affected.

3. **Alert fatigue kills** — too many false alarms means real alerts get ignored. Target < 2 pages per on-call shift.

4. **Every alert needs a runbook** — at 3 AM, an engineer needs clear "do this first, then this" instructions.

5. **On-call must be fair** — rotate weekly, compensate with time off or pay, limit to 8-12 hour shifts.

6. **Use the "for" duration** — require the condition to persist (e.g., 5 minutes) before firing to avoid alerting on brief spikes.

7. **Postmortems drive improvement** — after every major incident, identify what went wrong and add preventive measures. Blameless always.

---

## What's Next?

With alerting in place, you know WHEN things go wrong. But how do you define what "wrong" means quantitatively? Next, we'll cover **SLIs, SLOs, and SLAs** — the formal way to define and measure reliability.

**Next: [07-sli-slo-sla.md](./07-sli-slo-sla.md)** — SLIs, SLOs & SLAs — Defining Reliability
