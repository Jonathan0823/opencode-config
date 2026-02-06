# Alerting

## Prometheus Alertmanager

### Configuration

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'password'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    - match:
        severity: warning
      receiver: 'slack-warnings'

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'
        subject: 'Alert: {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          {{ end }}

  - name: 'slack-warnings'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'
        channel: '#alerts'
        title: 'Warning: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: '<integration-key>'
        description: '{{ .GroupLabels.alertname }}'
        severity: critical

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

## Prometheus Rules

### Recording Rules

```yaml
# recording-rules.yml
groups:
  - name: app_rules
    interval: 30s
    rules:
      - record: job:http_requests_per_second
        expr: |
          sum by (job) (
            rate(http_requests_total[5m])
          )
      
      - record: job:http_error_rate
        expr: |
          sum by (job) (
            rate(http_requests_total{status_code=~"5.."}[5m])
          ) / 
          sum by (job) (
            rate(http_requests_total[5m])
          )
      
      - record: job:http_latency_p99
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (
              rate(http_request_duration_seconds_bucket[5m])
            )
          )
```

### Alerting Rules

```yaml
# alerting-rules.yml
groups:
  - name: app_alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: job:http_error_rate > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.job }}"
      
      # High latency
      - alert: HighLatency
        expr: job:http_latency_p99 > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "p99 latency is {{ $value }}s for {{ $labels.job }}"
      
      # Low request rate
      - alert: LowRequestRate
        expr: job:http_requests_per_second < 10
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Low request rate"
          description: "Request rate is {{ $value }}/s for {{ $labels.job }}"
      
      # Service down
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
          description: "Service {{ $labels.instance }} is down"
      
      # Disk space
      - alert: DiskSpaceLow
        expr: |
          (
            node_filesystem_avail_bytes{mountpoint="/"} /
            node_filesystem_size_bytes{mountpoint="/"}
          ) < 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space is low"
          description: "Disk usage is above 90% on {{ $labels.instance }}"
      
      # Memory usage
      - alert: HighMemoryUsage
        expr: |
          (
            node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
          ) / node_memory_MemTotal_bytes > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}"
```

## SLOs and SLIs

### Definitions

```yaml
# Service Level Indicators (SLIs)
# - Request latency
# - Error rate
# - System throughput
# - Availability

# Service Level Objectives (SLOs)
slos:
  availability:
    target: 99.9  # 99.9% uptime
    window: 30d
  
  latency:
    target: 95    # 95% of requests under 200ms
    threshold: 200ms
    window: 7d
  
  error_rate:
    target: 0.1   # 0.1% error rate
    window: 7d
```

### Burn Rate Alerts

```yaml
# burn-rate-alerts.yml
groups:
  - name: slo_alerts
    rules:
      # Fast burn (2% budget in 1 hour)
      - alert: ErrorBudgetFastBurn
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[1h]))
            /
            sum(rate(http_requests_total[1h]))
          ) > (14.4 * 0.001)  # 14.4 * SLO
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Fast error budget burn"
          description: "Error budget is burning fast"
      
      # Slow burn (5% budget in 6 hours)
      - alert: ErrorBudgetSlowBurn
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[6h]))
            /
            sum(rate(http_requests_total[6h]))
          ) > (6 * 0.001)  # 6 * SLO
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Slow error budget burn"
          description: "Error budget is burning slowly"
```

## PagerDuty Integration

```yaml
# pagerduty-service.yml
receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: '<service-key>'
        description: '{{ template "pagerduty.default.description" . }}'
        severity: '{{ .CommonLabels.severity }}'
        details:
          firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
          resolved: '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'
        links:
          - href: '{{ template "pagerduty.default.dashboard_url" . }}'
            text: 'Dashboard'
```

## Alert Templates

```html
<!-- alert-template.tmpl -->
{{ define "slack.default.title" }}{{ .Status | toUpper }}: {{ .GroupLabels.alertname }}{{ end }}

{{ define "slack.default.text" }}
{{ range .Alerts }}
*Alert:* {{ .Annotations.summary }}
*Description:* {{ .Annotations.description }}
*Severity:* {{ .Labels.severity }}
*Started:* {{ .StartsAt }}

{{ if eq .Status "firing" }}
*Runbook:* {{ .Annotations.runbook_url }}
*Dashboard:* {{ .Annotations.dashboard_url }}
{{ end }}
{{ end }}
{{ end }}
```

## On-Call Rotation

```yaml
# PagerDuty schedule
schedule:
  name: "Engineering On-Call"
  timezone: "America/New_York"
  
  layers:
    - name: "Primary"
      rotation:
        type: "weekly"
        start: "2024-01-01T09:00:00-05:00"
        users:
          - "user1@example.com"
          - "user2@example.com"
          - "user3@example.com"
    
    - name: "Secondary"
      rotation:
        type: "weekly"
        start: "2024-01-01T09:00:00-05:00"
        handoff_time: "09:00"
        users:
          - "user4@example.com"
          - "user5@example.com"

  escalation_policy:
    - target: "Primary"
      timeout: 15m
    - target: "Secondary"
      timeout: 15m
    - target: "Manager"
      timeout: 30m
```
