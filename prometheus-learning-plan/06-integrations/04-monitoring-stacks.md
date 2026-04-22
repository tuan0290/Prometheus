# Monitoring Stacks

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc](#kiến-trúc)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
  - [Prometheus + Grafana Stack](#prometheus--grafana-stack)
  - [Prometheus + Alertmanager + Grafana](#prometheus--alertmanager--grafana)
  - [Thanos cho Long-term Storage](#thanos-cho-long-term-storage)
  - [Production-ready Considerations](#production-ready-considerations)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

---

## Tổng Quan

Một monitoring stack hoàn chỉnh không chỉ bao gồm Prometheus mà còn cần các thành phần bổ sung để visualization, alerting, và long-term storage. Stack phổ biến nhất là:

**Prometheus + Grafana + Alertmanager** - Stack cơ bản cho hầu hết use cases

Với production environments cần high availability và long-term storage, stack được mở rộng với:

**Prometheus + Grafana + Alertmanager + Thanos** - Stack production-grade

---

## Kiến Trúc

### Stack Cơ Bản

```
┌─────────────────────────────────────────────────────────────┐
│                    Monitoring Stack                         │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Data Collection Layer                   │  │
│  │                                                      │  │
│  │  node-exporter  cAdvisor  app-metrics  exporters     │  │
│  └──────────────────────────┬───────────────────────────┘  │
│                             │ scrape                        │
│  ┌──────────────────────────▼───────────────────────────┐  │
│  │              Storage & Processing Layer              │  │
│  │                                                      │  │
│  │              Prometheus Server                       │  │
│  │         (TSDB + Rule Evaluation)                     │  │
│  └──────────────────────────┬───────────────────────────┘  │
│                             │                               │
│              ┌──────────────┼──────────────┐               │
│              │              │              │               │
│              ▼              ▼              ▼               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Alertmanager│  │   Grafana    │  │  Prometheus  │     │
│  │  (Alerting)  │  │  (Dashboards)│  │  Web UI      │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│              │                                              │
│              ▼                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Notification Channels                      │  │
│  │   Email    Slack    PagerDuty    Webhook              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Stack với Thanos (Production)

```
┌─────────────────────────────────────────────────────────────┐
│                 Production Monitoring Stack                  │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐                       │
│  │ Prometheus 1 │    │ Prometheus 2 │  (HA pair)            │
│  │  + Sidecar   │    │  + Sidecar   │                       │
│  └──────┬───────┘    └──────┬───────┘                       │
│         │                  │                                │
│         └────────┬─────────┘                                │
│                  │ upload blocks                            │
│                  ▼                                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              Object Storage (S3/GCS/Azure)            │  │
│  └───────────────────────────────────────────────────────┘  │
│                  │                                          │
│         ┌────────┼────────┐                                 │
│         ▼        ▼        ▼                                 │
│  ┌──────────┐ ┌──────┐ ┌──────────────┐                    │
│  │  Thanos  │ │Thanos│ │    Thanos    │                    │
│  │  Store   │ │Query │ │  Compactor   │                    │
│  └──────────┘ └──┬───┘ └──────────────┘                    │
│                  │                                          │
│                  ▼                                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    Grafana                            │  │
│  │         (Thanos Query as datasource)                  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Cách Hoạt Động

### Data Flow

```
Targets (apps, exporters)
        │
        │ HTTP scrape /metrics
        ▼
Prometheus Server
        │
        ├── Evaluate alerting rules ──► Alertmanager ──► Notifications
        │
        ├── Store in local TSDB
        │
        └── Expose HTTP API ──► Grafana ──► Dashboards
                            └──► Thanos Sidecar ──► Object Storage
```

### Thanos Architecture

Thanos mở rộng Prometheus với:
- **Sidecar**: Chạy cạnh Prometheus, upload blocks lên object storage
- **Store Gateway**: Serve metrics từ object storage
- **Query**: Federated query layer, merge data từ nhiều Prometheus instances
- **Compactor**: Compact và downsample data trong object storage
- **Ruler**: Evaluate recording/alerting rules trên long-term data
- **Receive**: Nhận remote write từ Prometheus (push model)

---

## Use Cases

1. **Development**: Stack đơn giản Prometheus + Grafana cho local development
2. **Small production**: Prometheus + Grafana + Alertmanager cho single-server
3. **Multi-service**: Stack với nhiều exporters và service discovery
4. **High availability**: Dual Prometheus + Thanos cho zero-downtime
5. **Long-term analytics**: Thanos với object storage cho data retention nhiều năm
6. **Multi-cluster**: Thanos Query để aggregate metrics từ nhiều Kubernetes clusters

---

## Cấu Hình

### Prometheus + Grafana Stack

Stack tối giản cho development và testing:

```yaml
# docker-compose.yml
version: '3.8'

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data: {}
  grafana_data: {}

services:
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    restart: unless-stopped
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    ports:
      - "3000:3000"
    networks:
      - monitoring
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    networks:
      - monitoring
```

**Prometheus config:**

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

**Grafana provisioning - datasource:**

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

**Grafana provisioning - dashboard:**

```yaml
# grafana/provisioning/dashboards/dashboard.yml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    options:
      path: /etc/grafana/provisioning/dashboards
```

**Khởi động:**

```bash
# Tạo thư mục
mkdir -p grafana/provisioning/datasources grafana/provisioning/dashboards

# Khởi động stack
docker compose up -d

# Kiểm tra
docker compose ps
docker compose logs prometheus

# Truy cập
# Prometheus: http://localhost:9090
# Grafana: http://localhost:3000 (admin/admin)
```

---

### Prometheus + Alertmanager + Grafana

Stack đầy đủ với alerting:

```yaml
# docker-compose.full.yml
version: '3.8'

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data: {}
  grafana_data: {}
  alertmanager_data: {}

services:
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./config/prometheus/rules:/etc/prometheus/rules
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
      - '--web.external-url=http://prometheus.example.com'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    restart: unless-stopped
    volumes:
      - ./config/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--web.external-url=http://alertmanager.example.com'
    ports:
      - "9093:9093"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    restart: unless-stopped
    volumes:
      - grafana_data:/var/lib/grafana
      - ./config/grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD:-admin}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_DOMAIN=grafana.example.com
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=smtp.gmail.com:587
      - GF_SMTP_USER=${SMTP_USER}
      - GF_SMTP_PASSWORD=${SMTP_PASSWORD}
    ports:
      - "3000:3000"
    networks:
      - monitoring
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    network_mode: host

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    restart: unless-stopped
    privileged: true
    devices:
      - /dev/kmsg:/dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring
```

**Prometheus config với alerting:**

```yaml
# config/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    region: 'us-east-1'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
      timeout: 10s

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'alertmanager'
    static_configs:
      - targets: ['alertmanager:9093']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

**Alertmanager config:**

```yaml
# config/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'your-app-password'
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
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
        send_resolved: true

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ template "slack.default.title" . }}'
        text: '{{ template "slack.default.text" . }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
```

---

### Thanos cho Long-term Storage

**Cài đặt Thanos với Docker Compose:**

```yaml
# docker-compose.thanos.yml
version: '3.8'

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus1_data: {}
  prometheus2_data: {}
  thanos_data: {}

services:
  # Prometheus instance 1
  prometheus1:
    image: prom/prometheus:v2.48.0
    container_name: prometheus1
    restart: unless-stopped
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus1_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=2h'  # Thanos sẽ handle long-term
      - '--storage.tsdb.min-block-duration=2h'
      - '--storage.tsdb.max-block-duration=2h'
      - '--web.enable-lifecycle'
    networks:
      - monitoring

  # Thanos Sidecar cho Prometheus 1
  thanos-sidecar1:
    image: thanosio/thanos:v0.32.0
    container_name: thanos-sidecar1
    restart: unless-stopped
    volumes:
      - prometheus1_data:/prometheus
      - ./config/thanos/bucket.yml:/etc/thanos/bucket.yml
    command:
      - sidecar
      - --tsdb.path=/prometheus
      - --prometheus.url=http://prometheus1:9090
      - --objstore.config-file=/etc/thanos/bucket.yml
      - --http-address=0.0.0.0:10902
      - --grpc-address=0.0.0.0:10901
    networks:
      - monitoring
    depends_on:
      - prometheus1

  # Prometheus instance 2 (HA pair)
  prometheus2:
    image: prom/prometheus:v2.48.0
    container_name: prometheus2
    restart: unless-stopped
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus2_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=2h'
      - '--storage.tsdb.min-block-duration=2h'
      - '--storage.tsdb.max-block-duration=2h'
      - '--web.enable-lifecycle'
    networks:
      - monitoring

  # Thanos Sidecar cho Prometheus 2
  thanos-sidecar2:
    image: thanosio/thanos:v0.32.0
    container_name: thanos-sidecar2
    restart: unless-stopped
    volumes:
      - prometheus2_data:/prometheus
      - ./config/thanos/bucket.yml:/etc/thanos/bucket.yml
    command:
      - sidecar
      - --tsdb.path=/prometheus
      - --prometheus.url=http://prometheus2:9090
      - --objstore.config-file=/etc/thanos/bucket.yml
      - --http-address=0.0.0.0:10902
      - --grpc-address=0.0.0.0:10901
    networks:
      - monitoring
    depends_on:
      - prometheus2

  # Thanos Store Gateway (serve từ object storage)
  thanos-store:
    image: thanosio/thanos:v0.32.0
    container_name: thanos-store
    restart: unless-stopped
    volumes:
      - ./config/thanos/bucket.yml:/etc/thanos/bucket.yml
      - thanos_data:/data
    command:
      - store
      - --data-dir=/data
      - --objstore.config-file=/etc/thanos/bucket.yml
      - --http-address=0.0.0.0:10902
      - --grpc-address=0.0.0.0:10901
    networks:
      - monitoring

  # Thanos Query (federated query layer)
  thanos-query:
    image: thanosio/thanos:v0.32.0
    container_name: thanos-query
    restart: unless-stopped
    command:
      - query
      - --http-address=0.0.0.0:9090
      - --grpc-address=0.0.0.0:10901
      - --store=thanos-sidecar1:10901
      - --store=thanos-sidecar2:10901
      - --store=thanos-store:10901
      - --query.replica-label=replica
    ports:
      - "9091:9090"
    networks:
      - monitoring
    depends_on:
      - thanos-sidecar1
      - thanos-sidecar2
      - thanos-store

  # Thanos Compactor
  thanos-compactor:
    image: thanosio/thanos:v0.32.0
    container_name: thanos-compactor
    restart: unless-stopped
    volumes:
      - ./config/thanos/bucket.yml:/etc/thanos/bucket.yml
      - thanos_data:/data
    command:
      - compact
      - --data-dir=/data
      - --objstore.config-file=/etc/thanos/bucket.yml
      - --http-address=0.0.0.0:10902
      - --wait
      - --retention.resolution-raw=30d
      - --retention.resolution-5m=90d
      - --retention.resolution-1h=1y
    networks:
      - monitoring

  # Grafana với Thanos Query
  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    restart: unless-stopped
    volumes:
      - ./config/grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - "3000:3000"
    networks:
      - monitoring
    depends_on:
      - thanos-query
```

**Thanos bucket config (S3):**

```yaml
# config/thanos/bucket.yml
type: S3
config:
  bucket: my-thanos-bucket
  endpoint: s3.amazonaws.com
  region: us-east-1
  # Dùng IAM role thay vì access keys nếu chạy trên EC2
  access_key: YOUR_ACCESS_KEY
  secret_key: YOUR_SECRET_KEY
  insecure: false
  signature_version2: false
  encrypt_sse: false
  put_user_metadata: {}
  http_config:
    idle_conn_timeout: 90s
    response_header_timeout: 2m
    insecure_skip_verify: false
  trace:
    enable: false
```

**Thanos bucket config (GCS):**

```yaml
# config/thanos/bucket-gcs.yml
type: GCS
config:
  bucket: my-thanos-bucket
  service_account: |-
    {
      "type": "service_account",
      "project_id": "my-project",
      "private_key_id": "...",
      "private_key": "...",
      "client_email": "...",
      "client_id": "...",
      "auth_uri": "https://accounts.google.com/o/oauth2/auth",
      "token_uri": "https://oauth2.googleapis.com/token"
    }
```

---

### Production-ready Considerations

**1. High Availability cho Alertmanager:**

```yaml
# Chạy 3 Alertmanager instances với clustering
alertmanager1:
  image: prom/alertmanager:v0.26.0
  command:
    - '--config.file=/etc/alertmanager/alertmanager.yml'
    - '--cluster.listen-address=0.0.0.0:9094'
    - '--cluster.peer=alertmanager2:9094'
    - '--cluster.peer=alertmanager3:9094'

alertmanager2:
  image: prom/alertmanager:v0.26.0
  command:
    - '--config.file=/etc/alertmanager/alertmanager.yml'
    - '--cluster.listen-address=0.0.0.0:9094'
    - '--cluster.peer=alertmanager1:9094'
    - '--cluster.peer=alertmanager3:9094'
```

**2. Recording Rules để tối ưu performance:**

```yaml
# config/prometheus/rules/recording-rules.yml
groups:
  - name: node_recording_rules
    interval: 1m
    rules:
      # Pre-compute CPU usage
      - record: instance:node_cpu_utilisation:rate5m
        expr: |
          1 - avg without (cpu) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          )

      # Pre-compute memory usage
      - record: instance:node_memory_utilisation:ratio
        expr: |
          1 - (
            node_memory_MemAvailable_bytes /
            node_memory_MemTotal_bytes
          )

      # Pre-compute network traffic
      - record: instance:node_network_receive_bytes:rate5m
        expr: |
          sum without (device) (
            rate(node_network_receive_bytes_total{device!~"lo|veth.*"}[5m])
          )
```

**3. Prometheus Federation cho multi-cluster:**

```yaml
# Global Prometheus scraping từ regional Prometheus instances
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true
    metrics_path: '/federate'
    params:
      match[]:
        - '{job="node-exporter"}'
        - '{job="cadvisor"}'
        - '{__name__=~"job:.*"}'
    static_configs:
      - targets:
          - 'prometheus-us-east:9090'
          - 'prometheus-eu-west:9090'
          - 'prometheus-ap-south:9090'
```

**4. Security với TLS và authentication:**

```yaml
# prometheus.yml với TLS
scrape_configs:
  - job_name: 'secure-target'
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/certs/ca.crt
      cert_file: /etc/prometheus/certs/client.crt
      key_file: /etc/prometheus/certs/client.key
    basic_auth:
      username: prometheus
      password_file: /etc/prometheus/secrets/password
    static_configs:
      - targets: ['secure-app:443']
```

**5. Resource sizing guidelines:**

```yaml
# Prometheus resource recommendations
# Small (< 100 targets, < 1M series):
#   CPU: 2 cores, Memory: 4GB, Storage: 100GB

# Medium (100-500 targets, 1-10M series):
#   CPU: 4 cores, Memory: 16GB, Storage: 500GB

# Large (500+ targets, 10M+ series):
#   CPU: 8+ cores, Memory: 32GB+, Storage: 1TB+
#   Consider: Thanos, Cortex, or VictoriaMetrics

resources:
  requests:
    cpu: 2000m
    memory: 8Gi
  limits:
    cpu: 4000m
    memory: 16Gi
```

---

## Best Practices

1. **Start simple**: Bắt đầu với stack đơn giản, mở rộng khi cần
2. **Persistent storage**: Luôn dùng persistent volumes cho Prometheus và Grafana
3. **Backup**: Backup Prometheus data và Grafana dashboards định kỳ
4. **Version pinning**: Pin versions của tất cả images để tránh breaking changes
5. **Environment variables**: Dùng `.env` file cho secrets, không hardcode
6. **Health checks**: Thêm health checks cho tất cả services
7. **Resource limits**: Đặt resource limits để tránh OOM
8. **Monitoring the monitoring**: Monitor Prometheus itself (meta-monitoring)
9. **Grafana dashboards as code**: Lưu dashboards trong version control
10. **Alertmanager HA**: Chạy ít nhất 2 Alertmanager instances cho production
11. **Thanos cho long-term**: Dùng Thanos khi cần retention > 30 ngày
12. **Regular testing**: Test alerting pipeline định kỳ

---

## Tài Liệu Liên Quan

- [Kubernetes Integration](./01-kubernetes-integration.md) - Deploy stack trên Kubernetes
- [Docker Monitoring](./02-docker-monitoring.md) - Monitor Docker containers
- [Cloud Providers](./03-cloud-providers.md) - Tích hợp cloud metrics
- [Alertmanager Config](../05-alerting/02-alertmanager-config.md) - Cấu hình Alertmanager chi tiết
- [Grafana Integration](../05-alerting/04-grafana-integration.md) - Grafana dashboards

---

## Tài Liệu Tham Khảo

- [Thanos Documentation](https://thanos.io/tip/thanos/getting-started.md/)
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [Grafana Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [Prometheus Storage](https://prometheus.io/docs/prometheus/latest/storage/)
- [Alertmanager HA](https://prometheus.io/docs/alerting/latest/alertmanager/#high-availability)
