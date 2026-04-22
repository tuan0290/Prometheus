# Lab 05 - Exporters Instructions

## Mục Tiêu

Trong lab này, bạn sẽ cài đặt và cấu hình multiple exporters để monitor infrastructure. Bạn sẽ setup blackbox_exporter (HTTP/TCP monitoring) và postgres_exporter (database monitoring).

## Yêu Cầu

- Lab 01-04 đã hoàn thành
- Prometheus running
- PostgreSQL installed (hoặc sẽ cài trong lab này)

## Phần 1: Install blackbox_exporter

### Bước 1: Download blackbox_exporter

```bash
cd /tmp
BB_VERSION="0.25.0"
wget https://github.com/prometheus/blackbox_exporter/releases/download/v${BB_VERSION}/blackbox_exporter-${BB_VERSION}.linux-amd64.tar.gz

# Extract
tar -xzf blackbox_exporter-${BB_VERSION}.linux-amd64.tar.gz
cd blackbox_exporter-${BB_VERSION}.linux-amd64
```

### Bước 2: Install Binary

```bash
# Copy binary
sudo cp blackbox_exporter /usr/local/bin/

# Create config directory
sudo mkdir -p /etc/blackbox_exporter

# Set ownership
sudo chown prometheus:prometheus /usr/local/bin/blackbox_exporter
sudo chown -R prometheus:prometheus /etc/blackbox_exporter
```

### Bước 3: Create Configuration

Tạo `/etc/blackbox_exporter/blackbox.yml`:

```bash
sudo vim /etc/blackbox_exporter/blackbox.yml
```

Nội dung:

```yaml
modules:
  # HTTP 2xx check
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: []  # Defaults to 2xx
      method: GET
      follow_redirects: true
      preferred_ip_protocol: "ip4"

  # HTTP POST check
  http_post_2xx:
    prober: http
    timeout: 5s
    http:
      method: POST
      headers:
        Content-Type: application/json
      body: '{"test": "data"}'

  # HTTPS with certificate validation
  http_2xx_with_tls:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: []
      method: GET
      tls_config:
        insecure_skip_verify: false

  # TCP connection check
  tcp_connect:
    prober: tcp
    timeout: 5s

  # ICMP ping check
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: "ip4"

  # DNS check
  dns_check:
    prober: dns
    timeout: 5s
    dns:
      query_name: "example.com"
      query_type: "A"
```

Set ownership:

```bash
sudo chown prometheus:prometheus /etc/blackbox_exporter/blackbox.yml
```

### Bước 4: Create Systemd Service

```bash
sudo vim /etc/systemd/system/blackbox_exporter.service
```

Nội dung:

```ini
[Unit]
Description=Blackbox Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/blackbox_exporter \
  --config.file=/etc/blackbox_exporter/blackbox.yml

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Bước 5: Start blackbox_exporter

```bash
sudo systemctl daemon-reload
sudo systemctl enable blackbox_exporter
sudo systemctl start blackbox_exporter
sudo systemctl status blackbox_exporter
```

### Bước 6: Test blackbox_exporter

```bash
# Check metrics endpoint
curl http://localhost:9115/metrics

# Test HTTP probe
curl 'http://localhost:9115/probe?target=https://prometheus.io&module=http_2xx'
```

## Phần 2: Configure Prometheus for blackbox_exporter

### Bước 1: Update Prometheus Config

Edit `/etc/prometheus/prometheus.yml`:

```bash
sudo vim /etc/prometheus/prometheus.yml
```

Add scrape config:

```yaml
scrape_configs:
  # ... existing configs ...

  # Blackbox exporter - HTTP monitoring
  - job_name: 'blackbox_http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://prometheus.io
          - https://grafana.com
          - http://localhost:9090
          - http://localhost:8080
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115

  # Blackbox exporter - TCP monitoring
  - job_name: 'blackbox_tcp'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
          - localhost:9090
          - localhost:9093
          - localhost:5432
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115

  # Blackbox exporter itself
  - job_name: 'blackbox_exporter'
    static_configs:
      - targets: ['localhost:9115']
```

### Bước 2: Reload Prometheus

```bash
# Validate config
sudo /usr/local/bin/promtool check config /etc/prometheus/prometheus.yml

# Reload
curl -X POST http://localhost:9090/-/reload
```

### Bước 3: Verify Targets

Trong Prometheus UI:
1. Navigate to **Status** → **Targets**
2. Verify `blackbox_http`, `blackbox_tcp`, `blackbox_exporter` jobs
3. All targets should be UP

## Phần 3: Install PostgreSQL

### Bước 1: Install PostgreSQL

```bash
# CentOS/RHEL
sudo yum install -y postgresql-server postgresql-contrib

# Ubuntu/Debian
sudo apt update
sudo apt install -y postgresql postgresql-contrib
```

### Bước 2: Initialize và Start PostgreSQL

```bash
# CentOS/RHEL
sudo postgresql-setup --initdb
sudo systemctl enable postgresql
sudo systemctl start postgresql

# Ubuntu/Debian
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

### Bước 3: Configure PostgreSQL Authentication

Edit pg_hba.conf:

```bash
# CentOS/RHEL
sudo vim /var/lib/pgsql/data/pg_hba.conf

# Ubuntu/Debian
sudo vim /etc/postgresql/*/main/pg_hba.conf
```

Change local connections to md5:

```
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
```

Restart PostgreSQL:

```bash
sudo systemctl restart postgresql
```

### Bước 4: Create Monitoring User

```bash
# Switch to postgres user
sudo -u postgres psql

# In psql:
CREATE USER prometheus_monitor WITH PASSWORD 'prometheus123';
GRANT pg_monitor TO prometheus_monitor;
\q
```

### Bước 5: Test Connection

```bash
psql -U prometheus_monitor -h localhost -d postgres
# Password: prometheus123
```

## Phần 4: Install postgres_exporter

### Bước 1: Download postgres_exporter

```bash
cd /tmp
PGE_VERSION="0.15.0"
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v${PGE_VERSION}/postgres_exporter-${PGE_VERSION}.linux-amd64.tar.gz

# Extract
tar -xzf postgres_exporter-${PGE_VERSION}.linux-amd64.tar.gz
cd postgres_exporter-${PGE_VERSION}.linux-amd64
```

### Bước 2: Install Binary

```bash
sudo cp postgres_exporter /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/postgres_exporter
```

### Bước 3: Create Systemd Service

```bash
sudo vim /etc/systemd/system/postgres_exporter.service
```

Nội dung:

```ini
[Unit]
Description=PostgreSQL Exporter
Wants=network-online.target
After=network-online.target postgresql.service

[Service]
User=prometheus
Group=prometheus
Type=simple
Environment="DATA_SOURCE_NAME=postgresql://prometheus_monitor:prometheus123@localhost:5432/postgres?sslmode=disable"
ExecStart=/usr/local/bin/postgres_exporter

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Bước 4: Start postgres_exporter

```bash
sudo systemctl daemon-reload
sudo systemctl enable postgres_exporter
sudo systemctl start postgres_exporter
sudo systemctl status postgres_exporter
```

### Bước 5: Test postgres_exporter

```bash
curl http://localhost:9187/metrics | grep pg_
```

## Phần 5: Configure Prometheus for postgres_exporter

### Bước 1: Update Prometheus Config

Edit `/etc/prometheus/prometheus.yml`:

```bash
sudo vim /etc/prometheus/prometheus.yml
```

Add scrape config:

```yaml
scrape_configs:
  # ... existing configs ...

  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']
        labels:
          database: 'postgres'
```

### Bước 2: Reload Prometheus

```bash
curl -X POST http://localhost:9090/-/reload
```

### Bước 3: Verify Target

Trong Prometheus UI, verify `postgres` job is UP.

## Phần 6: Query Exporter Metrics

### blackbox_exporter Queries

```promql
# HTTP probe success
probe_success{job="blackbox_http"}

# HTTP response time
probe_http_duration_seconds{job="blackbox_http"}

# SSL certificate expiry
probe_ssl_earliest_cert_expiry{job="blackbox_http"}

# HTTP status code
probe_http_status_code{job="blackbox_http"}

# TCP probe success
probe_success{job="blackbox_tcp"}

# Probe duration
probe_duration_seconds
```

### postgres_exporter Queries

```promql
# PostgreSQL up
pg_up

# Database size
pg_database_size_bytes

# Active connections
pg_stat_activity_count

# Transaction rate
rate(pg_stat_database_xact_commit[5m])

# Locks
pg_locks_count

# Replication lag (if applicable)
pg_replication_lag

# Table size
pg_stat_user_tables_n_tup_ins
pg_stat_user_tables_n_tup_upd
pg_stat_user_tables_n_tup_del
```

## Phần 7: Create Dashboards (Optional)

### blackbox_exporter Dashboard Queries

```promql
# Uptime percentage
avg_over_time(probe_success[24h]) * 100

# Average response time
avg(probe_http_duration_seconds{job="blackbox_http"})

# SSL certificate days until expiry
(probe_ssl_earliest_cert_expiry - time()) / 86400

# Failed probes
sum(probe_success == 0)
```

### postgres_exporter Dashboard Queries

```promql
# Connection usage
pg_stat_activity_count / pg_settings_max_connections * 100

# Transaction rate
sum(rate(pg_stat_database_xact_commit[5m]))

# Cache hit ratio
sum(pg_stat_database_blks_hit) / (sum(pg_stat_database_blks_hit) + sum(pg_stat_database_blks_read)) * 100

# Database size growth
rate(pg_database_size_bytes[1h])
```

## Phần 8: Create Alerts for Exporters

Edit `/etc/prometheus/rules/exporter_alerts.yml`:

```bash
sudo vim /etc/prometheus/rules/exporter_alerts.yml
```

Nội dung:

```yaml
groups:
  - name: exporter_alerts
    interval: 30s
    rules:
      # Blackbox alerts
      - alert: EndpointDown
        expr: probe_success == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Endpoint {{ $labels.instance }} is down"
          description: "HTTP probe failed for {{ $labels.instance }}."

      - alert: SlowResponse
        expr: probe_http_duration_seconds > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow response from {{ $labels.instance }}"
          description: "Response time is {{ $value }}s."

      - alert: SSLCertExpiringSoon
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "SSL certificate expiring soon for {{ $labels.instance }}"
          description: "Certificate expires in {{ $value }} days."

      # PostgreSQL alerts
      - alert: PostgreSQLDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is down"
          description: "PostgreSQL instance is not responding."

      - alert: PostgreSQLTooManyConnections
        expr: pg_stat_activity_count / pg_settings_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL too many connections"
          description: "Connection usage is {{ $value | humanizePercentage }}."

      - alert: PostgreSQLDeadLocks
        expr: rate(pg_stat_database_deadlocks[5m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL deadlocks detected"
          description: "Deadlock rate: {{ $value }}."
```

Reload Prometheus:

```bash
sudo /usr/local/bin/promtool check rules /etc/prometheus/rules/exporter_alerts.yml
curl -X POST http://localhost:9090/-/reload
```

## Troubleshooting

### Issue 1: blackbox_exporter probe fails

**Giải pháp**:
```bash
# Test probe manually
curl 'http://localhost:9115/probe?target=https://prometheus.io&module=http_2xx'

# Check logs
sudo journalctl -u blackbox_exporter -f

# Verify target is reachable
curl -I https://prometheus.io

# Check firewall
sudo ufw status
```

### Issue 2: postgres_exporter connection failed

**Giải pháp**:
```bash
# Test connection
psql -U prometheus_monitor -h localhost -d postgres

# Check pg_hba.conf
sudo cat /var/lib/pgsql/data/pg_hba.conf | grep md5

# Check PostgreSQL logs
sudo journalctl -u postgresql -f

# Verify DATA_SOURCE_NAME
sudo systemctl cat postgres_exporter | grep DATA_SOURCE_NAME
```

### Issue 3: SSL certificate check fails

**Giải pháp**:
```bash
# Use module with TLS verification disabled for testing
# Update blackbox.yml module to set insecure_skip_verify: true

# Or check certificate manually
openssl s_client -connect prometheus.io:443 -servername prometheus.io
```

## Best Practices

1. **Monitor Critical Endpoints**: Focus on user-facing services
2. **Set Appropriate Timeouts**: Balance between false positives and detection speed
3. **Use Multiple Probe Types**: HTTP, TCP, ICMP for comprehensive monitoring
4. **Monitor Database Health**: Not just availability, but performance metrics
5. **Alert on Trends**: Not just current state (e.g., connection growth)
6. **Secure Credentials**: Use secrets management for database passwords
7. **Regular Testing**: Verify exporters are collecting accurate data

## Tiếp Theo

Sau khi hoàn thành lab này, bạn đã có:
- ✅ blackbox_exporter monitoring HTTP/TCP endpoints
- ✅ postgres_exporter monitoring PostgreSQL
- ✅ Prometheus scraping exporter metrics
- ✅ Alerts configured for exporter metrics
- ✅ Understanding of exporter patterns

Tiếp tục với: [Verification](./verification.md) để kiểm tra kết quả, sau đó chuyển sang [Lab 06 - Service Discovery](../lab-06-service-discovery/README.md)
