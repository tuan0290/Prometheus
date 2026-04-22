# Notification Channels

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc Notification](#kiến-trúc-notification)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Email Configuration](#email-configuration)
- [Slack Integration](#slack-integration)
- [PagerDuty Integration](#pagerduty-integration)
- [Webhook Configuration](#webhook-configuration)
- [Notification Templates](#notification-templates)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

## Tổng Quan

Notification channels (receivers) trong Alertmanager là các kênh gửi thông báo khi alert firing. Alertmanager hỗ trợ nhiều loại receiver khác nhau, từ email đơn giản đến các hệ thống on-call phức tạp như PagerDuty.

**Các receiver được hỗ trợ:**
- Email (SMTP)
- Slack
- PagerDuty
- Webhook (HTTP)
- OpsGenie
- VictorOps / Splunk On-Call
- Telegram
- Microsoft Teams (qua webhook)
- SNS (Amazon Simple Notification Service)

## Kiến Trúc Notification

```
┌─────────────────────────────────────────────────────────────┐
│                      Alertmanager                           │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Routing Tree                        │  │
│  │                                                       │  │
│  │  severity=critical ──────────────▶ PagerDuty         │  │
│  │  severity=warning  ──────────────▶ Slack             │  │
│  │  team=dba          ──────────────▶ Email             │  │
│  │  default           ──────────────▶ Webhook           │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   Notifier Pool                       │  │
│  │                                                       │  │
│  │  EmailNotifier  SlackNotifier  PagerDutyNotifier      │  │
│  │  WebhookNotifier  OpsGenieNotifier  ...               │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
      Email           Slack        PagerDuty      Webhook
```

## Cách Hoạt Động

### Notification Flow

1. Alert được route đến receiver phù hợp
2. Alertmanager render template với alert data
3. Gửi notification đến external service
4. Retry nếu gửi thất bại (với backoff)
5. Log kết quả gửi

### Retry Logic

```
Lần 1: Gửi ngay
Lần 2: Sau 10s (nếu thất bại)
Lần 3: Sau 20s
Lần 4: Sau 40s
...
Tối đa: 5 lần retry
```

## Email Configuration

### Cấu Hình SMTP Global

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'app-password'  # Dùng App Password cho Gmail
  smtp_require_tls: true
```

### Receiver Email Cơ Bản

```yaml
receivers:
  - name: 'email-team'
    email_configs:
      - to: 'team@example.com'
        # Gửi email khi alert resolved
        send_resolved: true
        # Subject tùy chỉnh
        headers:
          Subject: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
```

### Email Với Nhiều Người Nhận

```yaml
receivers:
  - name: 'email-oncall'
    email_configs:
      - to: 'oncall@example.com, backup@example.com'
        send_resolved: true
        html: '{{ template "email.html" . }}'
        text: '{{ template "email.text" . }}'
```

### Cấu Hình Gmail

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'your-email@gmail.com'
  smtp_auth_username: 'your-email@gmail.com'
  # Tạo App Password tại: https://myaccount.google.com/apppasswords
  smtp_auth_password: 'xxxx-xxxx-xxxx-xxxx'
  smtp_require_tls: true
```

### Cấu Hình Office 365

```yaml
global:
  smtp_smarthost: 'smtp.office365.com:587'
  smtp_from: 'alerts@company.com'
  smtp_auth_username: 'alerts@company.com'
  smtp_auth_password: 'password'
  smtp_require_tls: true
```

## Slack Integration

### Tạo Slack Webhook

1. Truy cập [Slack API](https://api.slack.com/apps)
2. Tạo App mới hoặc chọn app có sẵn
3. Vào "Incoming Webhooks" → Enable
4. Click "Add New Webhook to Workspace"
5. Chọn channel → Copy webhook URL

### Cấu Hình Slack Cơ Bản

```yaml
global:
  slack_api_url: '<YOUR_SLACK_WEBHOOK_URL>'

receivers:
  - name: 'slack-alerts'
    slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ template "slack.default.title" . }}'
        text: '{{ template "slack.default.text" . }}'
```

### Slack Với Custom Formatting

```yaml
receivers:
  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        title: |
          [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}]
          {{ .GroupLabels.SortedPairs.Values | join " " }}
          {{ if gt (len .CommonLabels) (len .GroupLabels) }}
            ({{ with .CommonLabels.Remove .GroupLabels.Names }}{{ .Values | join " " }}{{ end }})
          {{ end }}
        text: >-
          {{ range .Alerts -}}
          *Alert:* {{ .Annotations.title }}{{ if .Labels.severity }} - `{{ .Labels.severity }}`{{ end }}

          *Description:* {{ .Annotations.description }}

          *Details:*
            {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
            {{ end }}
          {{ end }}
        actions:
          - type: button
            text: 'Runbook :green_book:'
            url: '{{ (index .Alerts 0).Annotations.runbook_url }}'
          - type: button
            text: 'Dashboard :grafana:'
            url: '{{ (index .Alerts 0).Annotations.dashboard_url }}'
```

### Slack Với Nhiều Channels

```yaml
receivers:
  - name: 'slack-platform'
    slack_configs:
      - channel: '#platform-alerts'
        api_url: 'https://hooks.slack.com/services/xxx'

  - name: 'slack-backend'
    slack_configs:
      - channel: '#backend-alerts'
        api_url: 'https://hooks.slack.com/services/yyy'
```

## PagerDuty Integration

### Tạo PagerDuty Integration

1. Đăng nhập PagerDuty
2. Vào Services → Service Directory
3. Tạo Service mới hoặc chọn service có sẵn
4. Vào Integrations → Add Integration
5. Chọn "Prometheus" hoặc "Events API v2"
6. Copy Integration Key

### Cấu Hình PagerDuty Cơ Bản

```yaml
receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: '<integration-key>'
        send_resolved: true
        description: '{{ template "pagerduty.default.description" . }}'
        severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
```

### PagerDuty Với Custom Details

```yaml
receivers:
  - name: 'pagerduty-production'
    pagerduty_configs:
      - routing_key: '<integration-key>'
        send_resolved: true
        severity: 'critical'
        description: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'
          num_firing: '{{ .Alerts.Firing | len }}'
          instance: '{{ .CommonLabels.instance }}'
          cluster: '{{ .CommonLabels.cluster }}'
        links:
          - href: '{{ (index .Alerts 0).Annotations.runbook_url }}'
            text: 'Runbook'
          - href: '{{ (index .Alerts 0).Annotations.dashboard_url }}'
            text: 'Dashboard'
```

### PagerDuty Với Service Key (v1)

```yaml
receivers:
  - name: 'pagerduty-v1'
    pagerduty_configs:
      - service_key: '<service-key>'  # Events API v1
        send_resolved: true
```

## Webhook Configuration

### Webhook Cơ Bản

Webhook gửi HTTP POST request với JSON payload đến URL bất kỳ:

```yaml
receivers:
  - name: 'webhook-custom'
    webhook_configs:
      - url: 'http://my-service:8080/alerts'
        send_resolved: true
        http_config:
          bearer_token: 'my-secret-token'
```

### Webhook Payload Format

Alertmanager gửi JSON với format sau:

```json
{
  "version": "4",
  "groupKey": "{}:{alertname=\"HighCPU\"}",
  "truncatedAlerts": 0,
  "status": "firing",
  "receiver": "webhook-custom",
  "groupLabels": {
    "alertname": "HighCPU"
  },
  "commonLabels": {
    "alertname": "HighCPU",
    "severity": "warning"
  },
  "commonAnnotations": {
    "summary": "CPU usage cao"
  },
  "externalURL": "http://alertmanager:9093",
  "alerts": [
    {
      "status": "firing",
      "labels": {
        "alertname": "HighCPU",
        "instance": "server1:9100",
        "severity": "warning"
      },
      "annotations": {
        "summary": "CPU usage cao trên server1",
        "description": "CPU usage là 85%"
      },
      "startsAt": "2024-01-01T00:00:00Z",
      "endsAt": "0001-01-01T00:00:00Z",
      "generatorURL": "http://prometheus:9090/graph?..."
    }
  ]
}
```

### Webhook Với Authentication

```yaml
receivers:
  - name: 'webhook-secure'
    webhook_configs:
      - url: 'https://api.example.com/webhooks/alerts'
        send_resolved: true
        http_config:
          # Basic auth
          basic_auth:
            username: 'alertmanager'
            password: 'secret'
          # Hoặc Bearer token
          bearer_token: 'eyJhbGciOiJIUzI1NiIs...'
          # Hoặc TLS
          tls_config:
            ca_file: '/etc/ssl/certs/ca.crt'
            cert_file: '/etc/ssl/certs/client.crt'
            key_file: '/etc/ssl/private/client.key'
```

### Ví Dụ Webhook Handler (Python)

```python
from flask import Flask, request, jsonify
import json

app = Flask(__name__)

@app.route('/alerts', methods=['POST'])
def handle_alert():
    data = request.get_json()
    
    status = data.get('status')
    alerts = data.get('alerts', [])
    
    for alert in alerts:
        alert_name = alert['labels'].get('alertname')
        severity = alert['labels'].get('severity')
        summary = alert['annotations'].get('summary', '')
        
        print(f"[{status.upper()}] {alert_name} ({severity}): {summary}")
        
        # Xử lý alert theo logic của bạn
        if status == 'firing' and severity == 'critical':
            create_incident(alert)
        elif status == 'resolved':
            resolve_incident(alert)
    
    return jsonify({'status': 'ok'}), 200

def create_incident(alert):
    # Logic tạo incident
    pass

def resolve_incident(alert):
    # Logic resolve incident
    pass

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

## Notification Templates

### Template Basics

Alertmanager sử dụng Go templates để format notifications:

```
{{ .Status }}           → "firing" hoặc "resolved"
{{ .Alerts }}           → Danh sách tất cả alerts
{{ .Alerts.Firing }}    → Chỉ firing alerts
{{ .Alerts.Resolved }}  → Chỉ resolved alerts
{{ .GroupLabels }}      → Labels dùng để group
{{ .CommonLabels }}     → Labels chung của tất cả alerts
{{ .CommonAnnotations }}→ Annotations chung
{{ .ExternalURL }}      → URL của Alertmanager
```

### Template File

```
{{/* /etc/alertmanager/templates/default.tmpl */}}

{{ define "email.subject" }}
[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }} - {{ .CommonLabels.instance }}
{{ end }}

{{ define "email.html" }}
<!DOCTYPE html>
<html>
<body>
<h2>{{ .Status | toUpper }}: {{ .GroupLabels.alertname }}</h2>

<table>
  <tr><th>Alert</th><th>Severity</th><th>Description</th></tr>
  {{ range .Alerts }}
  <tr>
    <td>{{ .Labels.alertname }}</td>
    <td>{{ .Labels.severity }}</td>
    <td>{{ .Annotations.description }}</td>
  </tr>
  {{ end }}
</table>

<p>
  <a href="{{ .ExternalURL }}">Alertmanager</a>
</p>
</body>
</html>
{{ end }}

{{ define "slack.title" }}
[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}
{{ end }}

{{ define "slack.text" }}
{{ range .Alerts }}
*{{ .Labels.alertname }}* ({{ .Labels.severity }})
{{ .Annotations.description }}
{{ end }}
{{ end }}
```

### Sử Dụng Template Trong Config

```yaml
templates:
  - '/etc/alertmanager/templates/*.tmpl'

receivers:
  - name: 'email-team'
    email_configs:
      - to: 'team@example.com'
        html: '{{ template "email.html" . }}'
        headers:
          Subject: '{{ template "email.subject" . }}'

  - name: 'slack-team'
    slack_configs:
      - channel: '#alerts'
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
```

### Template Functions Hữu Ích

```
{{ .Status | toUpper }}           → "FIRING"
{{ $value | printf "%.2f" }}      → "85.23"
{{ .Labels.SortedPairs }}         → Labels được sort
{{ range .Alerts }}...{{ end }}   → Lặp qua alerts
{{ if eq .Status "firing" }}...{{ end }} → Điều kiện
{{ len .Alerts.Firing }}          → Số lượng firing alerts
```

## Use Cases

### Email: Thông Báo Cho Team

- Alerts không khẩn cấp cần review
- Daily/weekly summary reports
- Thông báo cho management
- Audit trail

### Slack: Collaboration và Visibility

- Real-time alerts cho team
- Thảo luận và phối hợp xử lý
- Alerts ở nhiều mức severity
- Integration với ChatOps workflow

### PagerDuty: On-Call Management

- Critical alerts cần response ngay
- On-call rotation management
- Escalation policies
- Incident tracking và post-mortems

### Webhook: Custom Integration

- Tích hợp với hệ thống nội bộ
- ITSM tools (ServiceNow, Jira)
- Custom notification logic
- Audit logging

## Cấu Hình

### Cấu Hình Đầy Đủ Với Nhiều Channels

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'app-password'
  smtp_require_tls: true
  slack_api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  receiver: 'slack-default'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      group_wait: 10s
      repeat_interval: 1h
      continue: true  # Cũng gửi Slack

    - match:
        severity: critical
      receiver: 'slack-critical'

    - match:
        severity: warning
      receiver: 'slack-warnings'

    - match:
        team: dba
      receiver: 'email-dba'

receivers:
  - name: 'slack-default'
    slack_configs:
      - channel: '#alerts'
        send_resolved: true

  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
        color: 'danger'
        title: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *{{ .Annotations.summary }}*
          {{ .Annotations.description }}
          {{ end }}

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warning'
        send_resolved: true
        color: 'warning'
        title: '⚠️ WARNING: {{ .GroupLabels.alertname }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: 'your-pagerduty-key'
        send_resolved: true
        severity: 'critical'

  - name: 'email-dba'
    email_configs:
      - to: 'dba@example.com'
        send_resolved: true
        html: '{{ template "email.html" . }}'
```

## Best Practices

### 1. Phân Tầng Notifications

```
Critical → PagerDuty (on-call) + Slack #critical
Warning  → Slack #warnings
Info     → Slack #info (hoặc không notify)
```

### 2. Luôn Bật send_resolved

```yaml
email_configs:
  - to: 'team@example.com'
    send_resolved: true  # Quan trọng! Biết khi nào alert hết
```

### 3. Sử Dụng Templates Nhất Quán

- Tạo template files riêng biệt
- Dùng chung templates cho cùng loại receiver
- Include runbook URL và dashboard URL

### 4. Test Notifications

```bash
# Gửi test alert
curl -X POST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {
      "alertname": "TestAlert",
      "severity": "warning",
      "instance": "test-server"
    },
    "annotations": {
      "summary": "Test notification",
      "description": "This is a test alert"
    },
    "generatorURL": "http://prometheus:9090"
  }]'
```

### 5. Bảo Mật Credentials

```bash
# Dùng environment variables
export SLACK_WEBHOOK_URL="https://hooks.slack.com/..."
export PAGERDUTY_KEY="your-key"

# Trong alertmanager.yml
global:
  slack_api_url: "${SLACK_WEBHOOK_URL}"
```

### 6. Monitor Alertmanager Itself

```yaml
# Alert khi Alertmanager không gửi được notifications
- alert: AlertmanagerNotificationsFailing
  expr: |
    rate(alertmanager_notifications_failed_total[5m]) > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Alertmanager không gửi được notifications"
```

## Tài Liệu Liên Quan

- [Alerting Rules](./01-alerting-rules.md) - Viết alerting rules
- [Alertmanager Config](./02-alertmanager-config.md) - Routing và grouping
- [Grafana Integration](./04-grafana-integration.md) - Alerting trong Grafana
- [Lab 04 - Alertmanager](../07-labs/lab-04-alertmanager/README.md) - Bài tập thực hành

## Tài Liệu Tham Khảo

- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/) - Official docs
- [Notification Templates](https://prometheus.io/docs/alerting/latest/notifications/) - Template reference
- [Slack Incoming Webhooks](https://api.slack.com/messaging/webhooks) - Slack webhook setup
- [PagerDuty Events API](https://developer.pagerduty.com/docs/events-api-v2/overview/) - PagerDuty integration
