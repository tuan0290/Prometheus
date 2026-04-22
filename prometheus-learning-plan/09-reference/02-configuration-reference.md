# Configuration Reference

## prometheus.yml

### Global Section

```yaml
global:
  scrape_interval: 15s        # How often to scrape (default: 1m)
  scrape_timeout: 10s         # Timeout per scrape (default: 10s)
  evaluation_interval: 15s    # How often to evaluate rules (default: 1m)
  external_labels:            # Labels added to all time series
    cluster: 'prod'
    region: 'us-east-1'
```

### Alerting Section

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']
      timeout: 10s
      api_version: v2
```

### Rule Files

```yaml
rule_files:
  - '/etc/prometheus/rules/*.yml'
  - '/etc/prometheus/alerts/*.yml'
```

### Scrape Config

```yaml
scrape_configs:
  - job_name: 'my_job'
    scrape_interval: 30s          # Override global
    scrape_timeout: 10s
    metrics_path: '/metrics'      # Default: /metrics
    scheme: 'http'                # http or https
    honor_labels: false           # Override conflicting labels
    honor_timestamps: false       # Use scrape time
    
    params:                       # URL parameters
      module: [http_2xx]
    
    static_configs:
      - targets: ['host:port']
        labels:
          env: production
    
    relabel_configs: []           # Relabeling before scrape
    metric_relabel_configs: []    # Relabeling after scrape
    
    tls_config:
      ca_file: '/path/to/ca.pem'
      cert_file: '/path/to/cert.pem'
      key_file: '/path/to/key.pem'
      insecure_skip_verify: false
    
    basic_auth:
      username: 'user'
      password: 'pass'
```

### Service Discovery Configs

```yaml
# File-based
file_sd_configs:
  - files: ['/etc/prometheus/targets/*.json']
    refresh_interval: 5m

# Kubernetes
kubernetes_sd_configs:
  - role: pod|node|service|endpoints|ingress
    namespaces:
      names: ['default', 'monitoring']

# Consul
consul_sd_configs:
  - server: 'localhost:8500'
    services: ['web', 'api']
    tags: ['production']

# EC2
ec2_sd_configs:
  - region: 'us-east-1'
    port: 9100
    filters:
      - name: 'tag:Environment'
        values: ['production']
```

### Relabel Config

```yaml
relabel_configs:
  - source_labels: [__address__]
    target_label: __param_target
  
  - source_labels: [__param_target]
    target_label: instance
  
  - target_label: __address__
    replacement: 'blackbox:9115'
  
  - source_labels: [env]
    regex: 'production'
    action: keep          # keep|drop|replace|labelmap|labeldrop|labelkeep
  
  - source_labels: [__address__]
    regex: '([^:]+):.*'
    target_label: hostname
    replacement: '${1}'
```

### Remote Write/Read

```yaml
remote_write:
  - url: 'http://remote-storage:9201/write'
    queue_config:
      max_samples_per_send: 1000
      max_shards: 200
      capacity: 2500

remote_read:
  - url: 'http://remote-storage:9201/read'
    read_recent: true
```

## alertmanager.yml

### Global

```yaml
global:
  resolve_timeout: 5m
  
  # SMTP
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'user@gmail.com'
  smtp_auth_password: 'password'
  smtp_require_tls: true
  
  # Slack
  slack_api_url: 'https://hooks.slack.com/services/...'
  
  # PagerDuty
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'
```

### Route

```yaml
route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  
  routes:
    - match:
        severity: critical
      receiver: 'pager'
      continue: false       # Stop routing after match
    
    - match_re:
        service: '^(db|cache)$'
      receiver: 'database-team'
    
    - matchers:             # New syntax (v0.22+)
        - alertname = "HighCPU"
        - severity =~ "warning|critical"
      receiver: 'ops-team'
```

### Receivers

```yaml
receivers:
  - name: 'email'
    email_configs:
      - to: 'team@example.com'
        send_resolved: true
        headers:
          Subject: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
        html: '{{ template "email.html" . }}'
  
  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/...'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
  
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: '<key>'
        description: '{{ .GroupLabels.alertname }}'
        severity: '{{ .CommonLabels.severity }}'
  
  - name: 'webhook'
    webhook_configs:
      - url: 'http://my-service/webhook'
        send_resolved: true
        max_alerts: 0       # 0 = unlimited
```

### Inhibit Rules

```yaml
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
```

### Time Intervals

```yaml
time_intervals:
  - name: business_hours
    time_intervals:
      - times:
          - start_time: '09:00'
            end_time: '17:00'
        weekdays: ['monday:friday']
        months: ['january:december']
```

## Command Line Flags

### Prometheus

```bash
# Prometheus 3.x - các flags --storage.tsdb.retention.* đã deprecated
# Nên cấu hình retention trong prometheus.yml thay vì CLI flags
prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle \
  --web.enable-admin-api \
  --log.level=info
```

> **Lưu ý Prometheus 3.x**: `--storage.tsdb.retention.time` và `--storage.tsdb.retention.size` đã bị deprecated. Cấu hình retention trong `prometheus.yml`:
> ```yaml
> global:
>   # Retention mặc định: 15 ngày
>   # Cấu hình qua config file thay vì CLI flags
> storage:
>   tsdb:
>     retention:
>       time: 15d
>       size: 10GB
> ```

### Alertmanager

```bash
alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address=0.0.0.0:9093 \
  --log.level=info
```

### node_exporter

```bash
node_exporter \
  --web.listen-address=0.0.0.0:9100 \
  --collector.disable-defaults \
  --collector.cpu \
  --collector.meminfo \
  --collector.filesystem
```

## API Endpoints

### Prometheus API

```bash
# Query
GET /api/v1/query?query=up&time=<timestamp>

# Range query
GET /api/v1/query_range?query=up&start=<ts>&end=<ts>&step=15s

# Targets
GET /api/v1/targets

# Rules
GET /api/v1/rules

# Alerts
GET /api/v1/alerts

# Reload config
POST /-/reload

# Health check
GET /-/healthy
GET /-/ready
```

### Alertmanager API

```bash
# Get alerts
GET /api/v2/alerts

# Create silence
POST /api/v2/silences

# Get silences
GET /api/v2/silences

# Delete silence
DELETE /api/v2/silence/<id>

# Status
GET /api/v2/status

# Reload config
POST /-/reload
```
