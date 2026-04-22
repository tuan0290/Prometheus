# Docker Monitoring

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc](#kiến-trúc)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
  - [Docker Daemon Metrics](#docker-daemon-metrics)
  - [cAdvisor Installation](#cadvisor-installation)
  - [Docker Compose Monitoring Stack](#docker-compose-monitoring-stack)
  - [Container Resource Metrics](#container-resource-metrics)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

---

## Tổng Quan

Docker là nền tảng container phổ biến nhất cho development và production. Monitoring Docker containers giúp bạn hiểu resource usage, phát hiện bottlenecks, và đảm bảo ứng dụng hoạt động ổn định.

Có hai nguồn metrics chính cho Docker:
- **Docker daemon metrics**: Metrics về Docker engine (số containers, images, networks...)
- **cAdvisor (Container Advisor)**: Metrics chi tiết về resource usage của từng container (CPU, memory, network, disk I/O)

---

## Kiến Trúc

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Host                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Docker Engine                      │   │
│  │                                                 │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │   │
│  │  │Container │  │Container │  │  Container   │  │   │
│  │  │  App 1   │  │  App 2   │  │  App 3       │  │   │
│  │  └──────────┘  └──────────┘  └──────────────┘  │   │
│  │                                                 │   │
│  │  Docker Daemon Metrics (:9323)                  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  cAdvisor (:8080)                               │   │
│  │  - Reads /sys/fs/cgroup                         │   │
│  │  - Reads /var/lib/docker                        │   │
│  │  - Exposes /metrics endpoint                    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  node-exporter (:9100)                          │   │
│  │  - Host OS metrics                              │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼ scrape
              ┌───────────────────────┐
              │   Prometheus (:9090)  │
              └───────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   Grafana (:3000)     │
              └───────────────────────┘
```

---

## Cách Hoạt Động

### 1. Docker Daemon Metrics

Docker daemon có thể expose metrics trực tiếp qua HTTP endpoint khi được bật trong cấu hình. Metrics này cung cấp thông tin về Docker engine:
- Số containers đang chạy/dừng
- Số images
- Số networks và volumes
- Docker engine health

### 2. cAdvisor

cAdvisor (Container Advisor) là tool của Google để phân tích resource usage và performance của containers. Nó:
- Đọc thông tin từ Linux cgroups (`/sys/fs/cgroup`)
- Đọc thông tin từ Docker API
- Expose metrics qua HTTP endpoint `/metrics`
- Cung cấp metrics chi tiết: CPU, memory, network, filesystem

### 3. Metrics Collection Flow

```
Linux cgroups ──────────────────────────────────────────┐
Docker API ─────────────────────────────────────────────┤
                                                         ▼
                                               ┌──────────────────┐
                                               │    cAdvisor      │
                                               │  :8080/metrics   │
                                               └──────────────────┘
                                                         │
Docker Daemon ──────────────────────────────────────────┤
  :9323/metrics                                          │
                                                         ▼
node-exporter ──────────────────────────────► Prometheus scrape
  :9100/metrics
```

---

## Use Cases

1. **Container Resource Monitoring**: Theo dõi CPU, memory, network của từng container
2. **Capacity Planning**: Phân tích resource usage để quyết định sizing
3. **Performance Troubleshooting**: Phát hiện containers tiêu thụ quá nhiều resources
4. **OOM Detection**: Phát hiện containers bị kill do out-of-memory
5. **Network Monitoring**: Theo dõi network traffic giữa containers
6. **Disk I/O Monitoring**: Phát hiện I/O bottlenecks
7. **Container Lifecycle Tracking**: Theo dõi containers start/stop/restart

---

## Cấu Hình

### Docker Daemon Metrics

Bật metrics endpoint trong Docker daemon:

```bash
# Chỉnh sửa Docker daemon configuration
sudo nano /etc/docker/daemon.json
```

```json
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
# Restart Docker daemon
sudo systemctl restart docker

# Kiểm tra metrics endpoint
curl http://localhost:9323/metrics | head -20
```

Cấu hình Prometheus để scrape Docker daemon:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'docker'
    static_configs:
      - targets: ['localhost:9323']
    metrics_path: /metrics
```

**Metrics quan trọng từ Docker daemon:**

```promql
# Số containers đang chạy
engine_daemon_container_states_containers{state="running"}

# Số containers đã dừng
engine_daemon_container_states_containers{state="stopped"}

# Số images
engine_daemon_image_actions_seconds_count

# Docker engine health
engine_daemon_health_checks_total
```

### cAdvisor Installation

**Chạy cAdvisor bằng Docker:**

```bash
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  --privileged \
  --device=/dev/kmsg \
  gcr.io/cadvisor/cadvisor:v0.47.2
```

**Kiểm tra cAdvisor:**

```bash
# Xem metrics
curl http://localhost:8080/metrics | grep container_cpu

# Xem web UI
open http://localhost:8080
```

**Cấu hình Prometheus để scrape cAdvisor:**

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: /metrics
    scrape_interval: 15s
    # Giảm số metrics để tránh cardinality explosion
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'container_(network_tcp_usage_total|tasks_state|cpu_load_average_10s)'
        action: drop
```

### Docker Compose Monitoring Stack

Stack hoàn chỉnh với Prometheus, Grafana, cAdvisor, và node-exporter:

```yaml
# docker-compose.monitoring.yml
version: '3.8'

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data: {}
  grafana_data: {}

services:
  prometheus:
    image: prom/prometheus:v3.5.2
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
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
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_DOMAIN=localhost
    ports:
      - "3000:3000"
    networks:
      - monitoring
    depends_on:
      - prometheus

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    restart: unless-stopped
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    ports:
      - "9093:9093"
    networks:
      - monitoring

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
      - /cgroup:/cgroup:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:v1.9.1
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    networks:
      - monitoring
```

**Prometheus config cho Docker Compose stack:**

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    environment: 'docker-compose'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    metrics_path: /metrics
    scrape_interval: 15s

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'docker'
    static_configs:
      - targets: ['host.docker.internal:9323']
```

**Khởi động stack:**

```bash
# Tạo thư mục cấu hình
mkdir -p prometheus/rules alertmanager grafana/provisioning

# Khởi động stack
docker compose -f docker-compose.monitoring.yml up -d

# Kiểm tra trạng thái
docker compose -f docker-compose.monitoring.yml ps

# Xem logs
docker compose -f docker-compose.monitoring.yml logs -f prometheus
```

### Container Resource Metrics

**Queries PromQL quan trọng cho Docker containers:**

```promql
# CPU usage theo container (%)
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100

# Memory usage theo container (bytes)
container_memory_usage_bytes{name!=""}

# Memory limit theo container
container_spec_memory_limit_bytes{name!=""}

# Memory usage percentage
(container_memory_usage_bytes{name!=""} / container_spec_memory_limit_bytes{name!=""}) * 100

# Network receive bytes/sec
rate(container_network_receive_bytes_total{name!=""}[5m])

# Network transmit bytes/sec
rate(container_network_transmit_bytes_total{name!=""}[5m])

# Disk read bytes/sec
rate(container_fs_reads_bytes_total{name!=""}[5m])

# Disk write bytes/sec
rate(container_fs_writes_bytes_total{name!=""}[5m])

# Container restart count
increase(container_start_time_seconds{name!=""}[1h])

# Containers đang chạy
count(container_last_seen{name!=""})
```

**Alerting rules cho Docker:**

```yaml
# prometheus/rules/docker-alerts.yml
groups:
  - name: docker-container-alerts
    rules:
      - alert: ContainerHighCPU
        expr: |
          rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} CPU cao"
          description: "CPU usage: {{ $value | humanize }}%"

      - alert: ContainerHighMemory
        expr: |
          (container_memory_usage_bytes{name!=""} / container_spec_memory_limit_bytes{name!=""}) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} memory cao"
          description: "Memory usage: {{ $value | humanize }}%"

      - alert: ContainerOOMKilled
        expr: |
          increase(container_oom_events_total{name!=""}[5m]) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Container {{ $labels.name }} bị OOM killed"

      - alert: ContainerDown
        expr: |
          absent(container_last_seen{name="my-critical-app"})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container my-critical-app không chạy"
```

---

## Best Practices

1. **Dùng cAdvisor**: Luôn deploy cAdvisor để có metrics chi tiết về containers
2. **Giới hạn cardinality**: Drop metrics không cần thiết bằng `metric_relabel_configs`
3. **Resource limits**: Đặt memory/CPU limits cho tất cả containers
4. **Logging**: Cấu hình log rotation để tránh đầy disk
5. **Network naming**: Dùng Docker networks có tên rõ ràng thay vì default bridge
6. **Health checks**: Thêm HEALTHCHECK vào Dockerfile để Docker tự monitor
7. **Labels**: Dùng Docker labels để phân loại containers
8. **Scrape interval**: 15s là phù hợp cho hầu hết use cases
9. **Retention**: Cân nhắc retention period dựa trên storage capacity
10. **Grafana dashboards**: Import dashboard ID 193 (Docker monitoring) từ Grafana.com

---

## Tài Liệu Liên Quan

- [Kubernetes Integration](./01-kubernetes-integration.md) - Container monitoring trên Kubernetes
- [Monitoring Stacks](./04-monitoring-stacks.md) - Complete monitoring stack
- [Common Exporters](../04-instrumentation/03-common-exporters.md) - Danh sách exporters
- [Alerting Rules](../05-alerting/01-alerting-rules.md) - Viết alerting rules

---

## Tài Liệu Tham Khảo

- [cAdvisor GitHub](https://github.com/google/cadvisor)
- [Docker Metrics Documentation](https://docs.docker.com/config/daemon/prometheus/)
- [Grafana Docker Dashboard](https://grafana.com/grafana/dashboards/193)
- [Prometheus Docker Compose Example](https://github.com/prometheus/prometheus/blob/main/documentation/examples/prometheus-docker.yml)
