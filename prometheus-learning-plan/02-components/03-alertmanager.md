# Alertmanager

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc](#kiến-trúc)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Tổng Quan

Alertmanager là thành phần xử lý alerts được gửi từ Prometheus Server. Nó chịu trách nhiệm deduplication, grouping, routing alerts đến các notification channels (email, Slack, PagerDuty, webhook, v.v.).

Các tính năng chính:
- **Grouping**: Gom nhóm alerts tương tự
- **Inhibition**: Ngăn chặn alerts dư thừa
- **Silencing**: Tạm thời tắt alerts
- **Routing**: Định tuyến alerts đến receivers phù hợp

## Kiến Trúc

```
┌─────────────────────────────────────────────────────────┐
│              Prometheus Server                          │
│                                                         │
│  ┌──────────────┐    ┌──────────────┐                  │
│  │ Alert Rules  │───▶│  Evaluation  │                  │
│  └──────────────┘    └──────┬───────┘                  │
└─────────────────────────────┼─────────────────────────┘
                              │ Firing Alerts
                              ▼
┌─────────────────────────────────────────────────────────┐
│                   Alertmanager                          │
│                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐           │
│  │Grouping  │──▶│Inhibition│──▶│ Routing  │           │
│  └──────────┘   └──────────┘   └────┬─────┘           │
│                                      │                 │
│  ┌──────────┐                        │                 │
│  │Silencing │────────────────────────┘                 │
│  └──────────┘                                          │
└─────────────────────────────┬───────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌─────────┐     ┌─────────┐     ┌─────────┐
        │  Email  │     │  Slack  │     │PagerDuty│
        └─────────┘     └─────────┘     └─────────┘
```

### Alert Lifecycle

```
┌──────────┐
│ Inactive │  Alert rule không match
└────┬─────┘
     │ Rule matches
     ▼
┌──────────┐
│ Pending  │  Đợi 'for' duration
└────┬─────┘
     │ Duration exceeded
     ▼
┌──────────┐
│  Firing  │  Gửi đến Alertmanager
└────┬─────┘
     │
     ▼
┌──────────┐
│Resolved  │  Alert không còn match
└──────────┘
```

## Cách Hoạt Động

### 1. Grouping

Gom nhóm alerts tương tự để giảm notification spam.

**Ví dụ**: Nhiều instances của cùng service down:

```
Alert 1: instance=server1, service=api, status=down
Alert 2: instance=server2, service=api, status=down
Alert 3: instance=server3, service=api, status=down

Grouped notification:
"API service down on 3 instances: server1, server2, server3"
```

**Configuration**:
```yaml
route:
  group_by: ['alertname', 'service']
  group_wait: 10s        # Đợi 10s để gom thêm alerts
  group_interval: 10s    # Gửi update mỗi 10s
  repeat_interval: 1h    # Gửi lại sau 1h nếu chưa resolved
```

### 2. Inhibition

Ngăn chặn alerts dư thừa khi có alert quan trọng hơn đang firing.

**Ví dụ**: Nếu toàn bộ datacenter down, không cần alert từng service:

```
Alert 1: datacenter=dc1, status=down (critical)
Alert 2: service=api, datacenter=dc1, status=down (warning)
Alert 3: service=db, datacenter=dc1, status=down (warning)

Inhibition: Alert 1 inhibits Alert 2 và Alert 3
```

**Configuration**:
```yaml
inhibit_rules:
  - source_match:
      severity: 'critical'
      alertname: 'DatacenterDown'
    target_match:
      severity: 'warning'
    equal: ['datacenter']
```

### 3. Silencing

Tạm thời tắt alerts trong khoảng thời gian maintenance.

**Use cases**:
- Planned maintenance
- Known issues đang được fix
- Testing/debugging

**Tạo silence qua API**:
```bash
# Silence alert trong 2 giờ
curl -X POST http://alertmanager:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {
        "name": "alertname",
        "value": "HighCPU",
        "isRegex": false
      },
      {
        "name": "instance",
        "value": "server1",
        "isRegex": false
      }
    ],
    "startsAt": "2024-01-01T10:00:00Z",
    "endsAt": "2024-01-01T12:00:00Z",
    "createdBy": "admin",
    "comment": "Planned maintenance"
  }'
```

### 4. Routing

Định tuyến alerts đến receivers phù hợp dựa trên labels.

```
                    ┌─────────────┐
                    │   Alerts    │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Routing   │
                    │    Tree     │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
  ┌──────────┐      ┌──────────┐      ┌──────────┐
  │  Team A  │      │  Team B  │      │  Team C  │
  │  (Email) │      │ (Slack)  │      │(PagerDuty)│
  └──────────┘      └──────────┘      └──────────┘
```

## Use Cases

### 1. Team-Based Routing

Route alerts đến teams khác nhau:

```yaml
route:
  receiver: 'default'
  routes:
    # Backend team
    - match:
        team: backend
      receiver: backend-team
      routes:
        # Critical alerts to PagerDuty
        - match:
            severity: critical
          receiver: backend-pagerduty
        # Warning alerts to Slack
        - match:
            severity: warning
          receiver: backend-slack
    
    # Frontend team
    - match:
        team: frontend
      receiver: frontend-team
    
    # Database team
    - match:
        component: database
      receiver: database-team
```

### 2. Severity-Based Routing

Route dựa trên mức độ nghiêm trọng:

```yaml
route:
  receiver: 'default'
  routes:
    # Critical: PagerDuty (24/7 on-call)
    - match:
        severity: critical
      receiver: pagerduty
      repeat_interval: 5m
    
    # Warning: Slack (business hours)
    - match:
        severity: warning
      receiver: slack
      repeat_interval: 1h
    
    # Info: Email (daily digest)
    - match:
        severity: info
      receiver: email
      repeat_interval: 24h
```

### 3. Environment-Based Routing

Route khác nhau cho mỗi environment:

```yaml
route:
  receiver: 'default'
  routes:
    # Production: PagerDuty + Slack
    - match:
        env: production
      receiver: production-alerts
      continue: true  # Continue to other routes
    
    # Staging: Slack only
    - match:
        env: staging
      receiver: staging-slack
    
    # Development: Email only
    - match:
        env: development
      receiver: dev-email
```

### 4. Time-Based Routing

Route khác nhau theo thời gian:

```yaml
route:
  receiver: 'default'
  routes:
    # Business hours: Slack
    - match:
        severity: warning
      receiver: slack
      active_time_intervals:
        - business_hours
    
    # After hours: PagerDuty
    - match:
        severity: warning
      receiver: pagerduty
      active_time_intervals:
        - after_hours

time_intervals:
  - name: business_hours
    time_intervals:
      - times:
          - start_time: '09:00'
            end_time: '17:00'
        weekdays: ['monday:friday']
  
  - name: after_hours
    time_intervals:
      - times:
          - start_time: '17:00'
            end_time: '09:00'
        weekdays: ['monday:friday']
      - weekdays: ['saturday', 'sunday']
```

## Cấu Hình

### Cấu Hình Cơ Bản

File `alertmanager.yml`:

```yaml
# Global configuration
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'password'

# Templates
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing tree
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  
  routes:
    - match:
        severity: critical
      receiver: pagerduty
      repeat_interval: 5m

# Inhibition rules
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']

# Receivers
receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'
```

### Email Receiver

```yaml
receivers:
  - name: 'email-team'
    email_configs:
      - to: 'team@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'alertmanager@example.com'
        auth_password: 'password'
        headers:
          Subject: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        html: '{{ template "email.html" . }}'
        require_tls: true
```

### Slack Receiver

```yaml
receivers:
  - name: 'slack-team'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
        username: 'Alertmanager'
        icon_emoji: ':warning:'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
        actions:
          - type: button
            text: 'View in Prometheus'
            url: '{{ .GeneratorURL }}'
          - type: button
            text: 'Silence'
            url: '{{ .SilenceURL }}'
```

### PagerDuty Receiver

```yaml
receivers:
  - name: 'pagerduty-oncall'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_INTEGRATION_KEY'
        severity: '{{ if eq .GroupLabels.severity "critical" }}critical{{ else }}error{{ end }}'
        description: '{{ .GroupLabels.alertname }}: {{ .GroupLabels.instance }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'
          summary: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
```

### Webhook Receiver

```yaml
receivers:
  - name: 'webhook-custom'
    webhook_configs:
      - url: 'http://custom-webhook.example.com/alerts'
        send_resolved: true
        http_config:
          basic_auth:
            username: 'webhook-user'
            password: 'webhook-password'
```

### Custom Templates

File `templates/email.tmpl`:

```go
{{ define "email.html" }}
<!DOCTYPE html>
<html>
<head>
  <style>
    .alert { padding: 10px; margin: 10px 0; border-radius: 5px; }
    .critical { background-color: #ff4444; color: white; }
    .warning { background-color: #ffaa00; color: white; }
    .info { background-color: #4444ff; color: white; }
  </style>
</head>
<body>
  <h2>Alert Summary</h2>
  <p>Status: <strong>{{ .Status | toUpper }}</strong></p>
  <p>Group: {{ .GroupLabels.alertname }}</p>
  
  <h3>Firing Alerts ({{ .Alerts.Firing | len }})</h3>
  {{ range .Alerts.Firing }}
  <div class="alert {{ .Labels.severity }}">
    <h4>{{ .Labels.alertname }}</h4>
    <p><strong>Instance:</strong> {{ .Labels.instance }}</p>
    <p><strong>Summary:</strong> {{ .Annotations.summary }}</p>
    <p><strong>Description:</strong> {{ .Annotations.description }}</p>
    <p><strong>Started:</strong> {{ .StartsAt }}</p>
  </div>
  {{ end }}
  
  <h3>Resolved Alerts ({{ .Alerts.Resolved | len }})</h3>
  {{ range .Alerts.Resolved }}
  <div class="alert info">
    <h4>{{ .Labels.alertname }}</h4>
    <p><strong>Instance:</strong> {{ .Labels.instance }}</p>
    <p><strong>Resolved:</strong> {{ .EndsAt }}</p>
  </div>
  {{ end }}
</body>
</html>
{{ end }}
```

## Best Practices

### 1. Alert Design

**Good alert characteristics**:
- **Actionable**: Alert phải có action rõ ràng
- **Meaningful**: Alert phải có ý nghĩa thực tế
- **Timely**: Alert phải đến đúng lúc
- **Contextual**: Alert phải có đủ context

**Example**:
```yaml
# ❌ Bad: Không actionable
- alert: HighCPU
  expr: cpu_usage > 80
  annotations:
    summary: "CPU is high"

# ✅ Good: Actionable với context
- alert: HighCPU
  expr: cpu_usage > 80
  for: 5m
  labels:
    severity: warning
    component: compute
  annotations:
    summary: "High CPU usage on {{ $labels.instance }}"
    description: "CPU usage is {{ $value }}% (threshold: 80%)"
    runbook_url: "https://wiki.example.com/runbooks/high-cpu"
```

### 2. Grouping Strategy

Group alerts theo logical boundaries:

```yaml
# Group by service và environment
route:
  group_by: ['alertname', 'service', 'env']
  group_wait: 30s
  group_interval: 5m
```

### 3. Inhibition Rules

Prevent alert fatigue:

```yaml
inhibit_rules:
  # Node down inhibits all alerts from that node
  - source_match:
      alertname: 'NodeDown'
    target_match_re:
      alertname: '.*'
    equal: ['instance']
  
  # Cluster down inhibits node alerts
  - source_match:
      alertname: 'ClusterDown'
    target_match:
      alertname: 'NodeDown'
    equal: ['cluster']
```

### 4. High Availability

Chạy multiple Alertmanager instances:

```yaml
# Prometheus configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager1:9093
            - alertmanager2:9093
            - alertmanager3:9093
```

Alertmanager tự động deduplicate alerts giữa các instances.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│Alertmanager │◀───▶│Alertmanager │◀───▶│Alertmanager │
│      1      │     │      2      │     │      3      │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  Receivers   │
                    └──────────────┘
```

### 5. Testing Alerts

Test alert routing:

```bash
# Send test alert
amtool alert add \
  --alertmanager.url=http://localhost:9093 \
  alertname=TestAlert \
  severity=warning \
  instance=test-server

# Check routing
amtool config routes test \
  --config.file=alertmanager.yml \
  severity=critical \
  team=backend
```

### 6. Monitoring Alertmanager

Monitor Alertmanager itself:

```yaml
# Alert on Alertmanager down
- alert: AlertmanagerDown
  expr: up{job="alertmanager"} == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Alertmanager {{ $labels.instance }} is down"

# Alert on notification failures
- alert: AlertmanagerNotificationsFailing
  expr: rate(alertmanager_notifications_failed_total[5m]) > 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Alertmanager notifications failing"
```

### 7. Silence Management

Use silences wisely:

```bash
# List active silences
amtool silence query

# Create silence for maintenance
amtool silence add \
  alertname=HighCPU \
  instance=server1 \
  --duration=2h \
  --author=admin \
  --comment="Planned maintenance"

# Expire silence early
amtool silence expire <silence-id>
```

## Tài Liệu Liên Quan

- [Alerting Rules](../05-alerting/01-alerting-rules.md) - Viết alerting rules
- [Notification Channels](../05-alerting/03-notification-channels.md) - Cấu hình receivers
- [Lab 04 - Alertmanager](../07-labs/lab-04-alertmanager/README.md) - Bài tập thực hành
- [Prometheus Server](./01-prometheus-server.md) - Cấu hình Prometheus

## Tài Liệu Tham Khảo

- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [Alertmanager Notification Templates](https://prometheus.io/docs/alerting/latest/notifications/)
- [Alertmanager API](https://prometheus.io/docs/alerting/latest/management_api/)
