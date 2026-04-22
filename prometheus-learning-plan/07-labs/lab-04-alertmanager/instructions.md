# Lab 04 - Alertmanager Instructions

## Mục Tiêu

Trong lab này, bạn sẽ cấu hình alerting với Alertmanager. Bạn sẽ tạo alerting rules, configure routing, và test notification delivery.

## Yêu Cầu

- Lab 01-03 đã hoàn thành
- Prometheus running
- Email account hoặc Slack webhook (cho notifications)

## Phần 1: Install Alertmanager

### Bước 1: Download Alertmanager

```bash
cd /tmp
AM_VERSION="0.27.0"
wget https://github.com/prometheus/alertmanager/releases/download/v${AM_VERSION}/alertmanager-${AM_VERSION}.linux-amd64.tar.gz

# Extract
tar -xzf alertmanager-${AM_VERSION}.linux-amd64.tar.gz
cd alertmanager-${AM_VERSION}.linux-amd64
```

### Bước 2: Install Binaries

```bash
# Copy binaries
sudo cp alertmanager amtool /usr/local/bin/

# Create directories
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager

# Set ownership
sudo chown -R prometheus:prometheus /etc/alertmanager /var/lib/alertmanager
sudo chown prometheus:prometheus /usr/local/bin/alertmanager /usr/local/bin/amtool
```

### Bước 3: Create Configuration

Tạo `/etc/alertmanager/alertmanager.yml`:

```bash
sudo vim /etc/alertmanager/alertmanager.yml
```

Nội dung cơ bản:

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'default'

receivers:
  - name: 'default'
    # Will configure later

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

Set ownership:

```bash
sudo chown prometheus:prometheus /etc/alertmanager/alertmanager.yml
```

### Bước 4: Create Systemd Service

```bash
sudo vim /etc/systemd/system/alertmanager.service
```

Nội dung:

```ini
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Bước 5: Start Alertmanager

```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
sudo systemctl status alertmanager
```

### Bước 6: Verify Alertmanager UI

Truy cập: `http://<server-ip>:9093`

Bạn sẽ thấy Alertmanager web interface.

## Phần 2: Create Alerting Rules

### Bước 1: Create Alert Rules File

```bash
sudo vim /etc/prometheus/rules/alerts.yml
```

Nội dung:

```yaml
groups:
  - name: lab_alerts
    interval: 30s
    rules:
      # Alert when target is down
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute."

      # Alert on high CPU usage
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanize }}% on {{ $labels.instance }}."

      # Alert on high memory usage
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanize }}% on {{ $labels.instance }}."

      # Alert on high error rate
      - alert: HighErrorRate
        expr: sum(rate(app_requests_total{status="500"}[5m])) / sum(rate(app_requests_total[5m])) > 0.05
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)."

      # Alert on high latency
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(app_request_duration_seconds_bucket[5m])) > 0.5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High request latency"
          description: "95th percentile latency is {{ $value | humanizeDuration }}."

      # Alert on disk space
      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Only {{ $value | humanize }}% disk space remaining on {{ $labels.mountpoint }}."
```

Set ownership:

```bash
sudo chown prometheus:prometheus /etc/prometheus/rules/alerts.yml
```

### Bước 2: Update Prometheus Config

Edit `/etc/prometheus/prometheus.yml`:

```bash
sudo vim /etc/prometheus/prometheus.yml
```

Add alerting section:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

# Load rules
rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:
  # ... existing configs ...
```

### Bước 3: Validate Rules

```bash
sudo /usr/local/bin/promtool check rules /etc/prometheus/rules/alerts.yml
```

**Expected**: `SUCCESS: ... rules found`

### Bước 4: Reload Prometheus

```bash
curl -X POST http://localhost:9090/-/reload
```

### Bước 5: Verify Rules Loaded

1. Truy cập Prometheus UI: `http://<server-ip>:9090`
2. Navigate to **Status** → **Rules**
3. Verify alerts are loaded

## Phần 3: Configure Notification Channels

### Option A: Email Notifications

Edit `/etc/alertmanager/alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'your-email@gmail.com'
  smtp_auth_username: 'your-email@gmail.com'
  smtp_auth_password: 'your-app-password'

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'email-notifications'
  routes:
    - match:
        severity: critical
      receiver: 'critical-email'

receivers:
  - name: 'email-notifications'
    email_configs:
      - to: 'your-email@gmail.com'
        headers:
          Subject: '[Prometheus] {{ .GroupLabels.alertname }}'

  - name: 'critical-email'
    email_configs:
      - to: 'your-email@gmail.com'
        headers:
          Subject: '[CRITICAL] {{ .GroupLabels.alertname }}'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

**Note**: For Gmail, use App Password, not regular password.

### Option B: Slack Notifications

```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'slack-notifications'

receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

### Option C: Webhook (for testing)

```yaml
receivers:
  - name: 'webhook'
    webhook_configs:
      - url: 'http://localhost:5001/webhook'
        send_resolved: true
```

### Reload Alertmanager

```bash
# Validate config
sudo /usr/local/bin/amtool check-config /etc/alertmanager/alertmanager.yml

# Reload
sudo systemctl reload alertmanager
```

## Phần 4: Test Alerts

### Test 1: Trigger InstanceDown Alert

Stop node_exporter:

```bash
sudo systemctl stop node_exporter
```

Wait 1-2 minutes, then check:

1. Prometheus UI → **Alerts** tab
2. Alert should show as **PENDING** then **FIRING**
3. Alertmanager UI → Should show active alert
4. Check notification channel (email/Slack)

Restore:

```bash
sudo systemctl start node_exporter
```

### Test 2: Trigger HighErrorRate Alert

Generate errors:

```bash
for i in {1..100}; do
    curl -s http://localhost:8080/error > /dev/null
    sleep 0.1
done
```

Wait 1-2 minutes, check alert fires.

### Test 3: Trigger HighCPUUsage Alert

Generate CPU load:

```bash
# Install stress tool
sudo yum install -y stress  # CentOS/RHEL
# or
sudo apt install -y stress  # Ubuntu/Debian

# Generate load
stress --cpu 4 --timeout 300s
```

Wait 2-3 minutes, check alert fires.

### Test 4: Manual Alert Testing

Use amtool:

```bash
# Send test alert
amtool alert add test_alert \
  alertname=TestAlert \
  severity=warning \
  instance=localhost:9090 \
  --annotation=summary="This is a test alert" \
  --annotation=description="Testing Alertmanager configuration"
```

Check Alertmanager UI and notifications.

## Phần 5: Advanced Routing

### Configure Multiple Routes

Edit `/etc/alertmanager/alertmanager.yml`:

```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h

  routes:
    # Critical alerts to pager
    - match:
        severity: critical
      receiver: 'pager'
      continue: true

    # Database alerts to DB team
    - match_re:
        alertname: '.*Database.*'
      receiver: 'database-team'

    # High priority during business hours
    - match:
        severity: warning
      receiver: 'email'
      active_time_intervals:
        - business_hours

time_intervals:
  - name: business_hours
    time_intervals:
      - times:
          - start_time: '09:00'
            end_time: '17:00'
        weekdays: ['monday:friday']

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'

  - name: 'pager'
    email_configs:
      - to: 'oncall@example.com'

  - name: 'database-team'
    email_configs:
      - to: 'dba@example.com'

  - name: 'email'
    email_configs:
      - to: 'alerts@example.com'
```

## Phần 6: Silences

### Create Silence via UI

1. Truy cập Alertmanager UI: `http://<server-ip>:9093`
2. Click **Silences** tab
3. Click **New Silence**
4. Fill in:
   - Matchers: `alertname=HighCPUUsage`
   - Duration: 2h
   - Creator: your-name
   - Comment: "Planned maintenance"
5. Click **Create**

### Create Silence via CLI

```bash
# Silence specific alert
amtool silence add alertname=HighCPUUsage \
  --duration=2h \
  --author="admin" \
  --comment="Planned maintenance"

# List silences
amtool silence query

# Expire silence
amtool silence expire <silence-id>
```

## Phần 7: Alert Grouping và Inhibition

### Grouping Example

Alerts với same `alertname` và `severity` sẽ được grouped:

```yaml
route:
  group_by: ['alertname', 'severity']
  group_wait: 30s      # Wait before sending first notification
  group_interval: 5m   # Wait before sending batch of new alerts
  repeat_interval: 12h # Wait before resending notification
```

### Inhibition Example

Prevent warning alerts khi critical alert firing:

```yaml
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

## Troubleshooting

### Issue 1: Alerts Not Firing

**Giải pháp**:
```bash
# Check rules loaded
curl http://localhost:9090/api/v1/rules | jq .

# Check alert state
curl http://localhost:9090/api/v1/alerts | jq .

# Validate rules
sudo /usr/local/bin/promtool check rules /etc/prometheus/rules/alerts.yml

# Check Prometheus logs
sudo journalctl -u prometheus -f
```

### Issue 2: Notifications Not Sent

**Giải pháp**:
```bash
# Check Alertmanager logs
sudo journalctl -u alertmanager -f

# Test SMTP connection
telnet smtp.gmail.com 587

# Validate Alertmanager config
sudo /usr/local/bin/amtool check-config /etc/alertmanager/alertmanager.yml

# Check Alertmanager status
curl http://localhost:9093/api/v1/status | jq .
```

### Issue 3: Too Many Notifications

**Giải pháp**:
- Increase `repeat_interval`
- Use grouping effectively
- Add inhibition rules
- Use silences for known issues

## Best Practices

1. **Alert on Symptoms, Not Causes**: Alert on user-facing issues
2. **Use Severity Levels**: critical, warning, info
3. **Meaningful Annotations**: Include context and runbook links
4. **Test Alerts Regularly**: Ensure notifications work
5. **Use Silences**: For planned maintenance
6. **Group Related Alerts**: Reduce notification noise
7. **Set Appropriate `for` Duration**: Avoid flapping alerts

## Tiếp Theo

Sau khi hoàn thành lab này, bạn đã có:
- ✅ Alertmanager installed và configured
- ✅ Alert rules created và tested
- ✅ Notification channels configured
- ✅ Understanding of routing và grouping
- ✅ Knowledge of silences và inhibition

Tiếp tục với: [Verification](./verification.md) để kiểm tra kết quả, sau đó chuyển sang [Lab 05 - Exporters](../lab-05-exporters/README.md)
