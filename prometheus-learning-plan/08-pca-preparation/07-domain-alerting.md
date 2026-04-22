# Domain 5: Alerting and Dashboarding (14%)

## Tổng Quan

Domain này chiếm 14% kỳ thi PCA. Tập trung vào alerting rules, Alertmanager, notification channels, và Grafana dashboards.

## 1. Alerting Rules

### Syntax

```yaml
groups:
  - name: <group_name>
    interval: <evaluation_interval>  # Optional, overrides global
    rules:
      - alert: <AlertName>
        expr: <PromQL expression>
        for: <duration>              # How long condition must be true
        labels:
          <label_key>: <label_value>
        annotations:
          summary: <short description>
          description: <detailed description>
```

### Example Rules

```yaml
groups:
  - name: availability
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute."

      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) /
          sum(rate(http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}."

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High p95 latency"
          description: "95th percentile latency is {{ $value | humanizeDuration }}."
```

### Alert States

```
Inactive → Pending → Firing → Resolved

Inactive: Condition not met
Pending:  Condition met, waiting for 'for' duration
Firing:   Condition met for 'for' duration → notification sent
Resolved: Condition no longer met → resolve notification sent
```

### Template Variables

```yaml
annotations:
  summary: "Alert on {{ $labels.instance }}"
  description: "Value is {{ $value }}"
  description: "Value is {{ $value | humanize }}"
  description: "Value is {{ $value | humanizePercentage }}"
  description: "Duration is {{ $value | humanizeDuration }}"
```

### Validate Rules

```bash
promtool check rules /etc/prometheus/rules/*.yml
```

## 2. Alertmanager

### Architecture

```
Prometheus → Alertmanager → Receivers (Email, Slack, PagerDuty)
```

### Configuration Structure

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:587'
  slack_api_url: 'https://hooks.slack.com/...'

route:
  receiver: default
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  routes:
    - match:
        severity: critical
      receiver: pager
    - match_re:
        service: ^(db|cache)$
      receiver: database-team

receivers:
  - name: default
    email_configs:
      - to: 'team@example.com'
  - name: pager
    pagerduty_configs:
      - service_key: '<key>'

inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'instance']
```

### Routing

```
Route Tree:
root route (default receiver)
├── severity=critical → pager
├── service=~"db|cache" → database-team
└── (default) → email
```

### Grouping

```yaml
route:
  group_by: ['alertname', 'cluster']  # Group alerts with same values
  group_wait: 30s      # Wait before sending first notification
  group_interval: 5m   # Wait before sending new alerts in group
  repeat_interval: 12h # Wait before resending same notification
```

### Inhibition

Suppress warnings khi critical alert firing:

```yaml
inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'instance']
```

### Silences

```bash
# Create silence via CLI
amtool silence add alertname=HighCPU \
  --duration=2h \
  --author="admin" \
  --comment="Planned maintenance"

# List silences
amtool silence query

# Expire silence
amtool silence expire <id>
```

## 3. Notification Channels

### Email

```yaml
receivers:
  - name: email
    email_configs:
      - to: 'team@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'user@gmail.com'
        auth_password: 'app-password'
        headers:
          Subject: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
```

### Slack

```yaml
receivers:
  - name: slack
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/...'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
```

### PagerDuty

```yaml
receivers:
  - name: pagerduty
    pagerduty_configs:
      - service_key: '<integration-key>'
        description: '{{ .GroupLabels.alertname }}'
```

### Webhook

```yaml
receivers:
  - name: webhook
    webhook_configs:
      - url: 'http://my-service/webhook'
        send_resolved: true
```

## 4. Grafana Integration

### Add Prometheus Data Source

1. Grafana → Configuration → Data Sources
2. Add data source → Prometheus
3. URL: `http://prometheus:9090`
4. Save & Test

### Dashboard Panels

**Time Series Panel**:
```promql
# CPU usage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Stat Panel** (single value):
```promql
# Current error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100
```

**Gauge Panel**:
```promql
# Memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

**Table Panel**:
```promql
# Top endpoints by request count
topk(10, sum by (endpoint) (rate(http_requests_total[5m])))
```

### Variables

```
# Dashboard variable: instance
Query: label_values(up, instance)

# Usage in panel
rate(http_requests_total{instance="$instance"}[5m])
```

### Alerting in Grafana

Grafana có thể tạo alerts từ panels:

```yaml
# Alert rule in Grafana
condition: avg() OF query(A, 5m, now) IS ABOVE 80
```

## 5. Best Practices

### Alert Design

1. **Alert on symptoms, not causes**: Alert on user-facing issues
2. **Meaningful `for` duration**: Avoid flapping (minimum 1-2 minutes)
3. **Clear annotations**: Include runbook links
4. **Severity levels**: critical, warning, info
5. **Test alerts**: Regularly verify notifications work

### Dashboard Design

1. **Overview first**: High-level metrics at top
2. **Drill-down**: Detailed metrics below
3. **Consistent time ranges**: Use dashboard-level time range
4. **Meaningful titles**: Clear panel names
5. **Documentation**: Add panel descriptions

### Alertmanager Best Practices

1. **Group related alerts**: Reduce notification noise
2. **Use inhibition**: Prevent duplicate notifications
3. **Set appropriate repeat_interval**: Avoid alert fatigue
4. **Test routing**: Verify alerts reach correct receivers
5. **Use silences**: For planned maintenance

## 6. Key Concepts Summary

| Concept | Key Points |
|---------|-----------|
| Alert States | inactive → pending → firing → resolved |
| `for` duration | How long condition must be true before firing |
| Grouping | Batch related alerts into single notification |
| Inhibition | Suppress lower-severity alerts |
| Silences | Mute alerts for maintenance windows |
| Grafana | Visualization and dashboarding tool |

## 7. Practice Questions

**Q1**: Sự khác biệt giữa `group_wait`, `group_interval`, và `repeat_interval` là gì?

<details>
<summary>Đáp án</summary>

`group_wait`: Thời gian chờ trước khi gửi notification đầu tiên cho một group (để batch alerts). `group_interval`: Thời gian chờ trước khi gửi notification cho alerts mới trong group đã có. `repeat_interval`: Thời gian chờ trước khi gửi lại notification cho alert đang firing (tránh spam).

</details>

**Q2**: Khi nào alert ở trạng thái "pending"?

<details>
<summary>Đáp án</summary>

Alert ở trạng thái "pending" khi PromQL expression trả về true nhưng chưa đủ thời gian `for` duration. Ví dụ: `for: 5m` → alert pending trong 5 phút đầu, sau đó chuyển sang firing. Pending state giúp tránh flapping alerts từ transient issues.

</details>

**Q3**: Tại sao cần `honor_labels: true` khi scrape Pushgateway?

<details>
<summary>Đáp án</summary>

Pushgateway lưu metrics với labels từ batch jobs (job, instance). Nếu không có `honor_labels: true`, Prometheus sẽ override những labels này bằng labels từ scrape config, làm mất thông tin về job gốc. `honor_labels: true` giữ nguyên labels từ pushed metrics.

</details>

## Tiếp Theo

- [Practice Questions](./08-practice-questions.md)
- [Tài liệu tham khảo: Alerting](../05-alerting/)
