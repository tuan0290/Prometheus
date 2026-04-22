# Grafana Integration

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc](#kiến-trúc)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Cài Đặt Grafana](#cài-đặt-grafana)
- [Kết Nối Prometheus Data Source](#kết-nối-prometheus-data-source)
- [Tạo Dashboards](#tạo-dashboards)
- [Panel Types](#panel-types)
- [Variables và Templating](#variables-và-templating)
- [Alerting Trong Grafana](#alerting-trong-grafana)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

## Tổng Quan

Grafana là nền tảng visualization và analytics mã nguồn mở, thường được dùng cùng Prometheus để tạo dashboards trực quan. Grafana kết nối với Prometheus qua data source và cho phép tạo các panel đa dạng để hiển thị metrics.

**Lợi ích của Grafana:**
- Visualization phong phú với nhiều loại panel
- Dashboard templating với variables
- Alerting tích hợp
- Hỗ trợ nhiều data sources (Prometheus, InfluxDB, Elasticsearch, ...)
- Sharing và collaboration
- Plugin ecosystem phong phú

## Kiến Trúc

```
┌─────────────────────────────────────────────────────────────┐
│                         Grafana                             │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  Dashboard   │    │   Panel      │    │   Alert      │  │
│  │  Manager     │───▶│   Renderer   │    │   Engine     │  │
│  │              │    │              │    │              │  │
│  └──────────────┘    └──────┬───────┘    └──────┬───────┘  │
│                             │                   │          │
│  ┌──────────────────────────▼───────────────────▼───────┐  │
│  │                   Data Source Layer                   │  │
│  │                                                       │  │
│  │  PrometheusDS  InfluxDS  ElasticDS  MySQLDS  ...      │  │
│  └──────────────────────────┬──────────────────────────┘  │
└─────────────────────────────┼───────────────────────────────┘
                              │ PromQL queries
                              ▼
                    ┌──────────────────┐
                    │   Prometheus     │
                    │   Server         │
                    └──────────────────┘
```

### Các Thành Phần

| Thành Phần | Chức Năng |
|-----------|-----------|
| Dashboard | Container chứa nhiều panels |
| Panel | Đơn vị visualization (graph, stat, table, ...) |
| Data Source | Kết nối đến backend data (Prometheus) |
| Variable | Template variable cho dynamic dashboards |
| Alert Rule | Điều kiện trigger alert trong Grafana |
| Notification Channel | Nơi gửi Grafana alerts |

## Cách Hoạt Động

### Query Flow

```
User opens dashboard
        │
        ▼
Grafana evaluates variables
        │
        ▼
For each panel:
  1. Build PromQL query (with variable substitution)
  2. Send query to Prometheus
  3. Receive time series data
  4. Render visualization
        │
        ▼
Display dashboard to user
```

### Refresh Cycle

- Grafana tự động refresh theo interval (5s, 10s, 30s, 1m, ...)
- Mỗi refresh gửi queries mới đến Prometheus
- Time range picker điều khiển khoảng thời gian query

## Cài Đặt Grafana

### Cài Đặt Trên Linux (Ubuntu/Debian)

```bash
# Thêm Grafana repository
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

# Cài đặt Grafana OSS
sudo apt-get update
sudo apt-get install grafana

# Khởi động service
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# Kiểm tra status
sudo systemctl status grafana-server
```

### Cài Đặt Trên CentOS/RHEL

```bash
# Thêm repository
cat <<EOF | sudo tee /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF

# Cài đặt
sudo yum install grafana

# Khởi động
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

### Cài Đặt Với Docker

```bash
# Chạy Grafana container
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -v grafana-storage:/var/lib/grafana \
  -e GF_SECURITY_ADMIN_PASSWORD=admin \
  grafana/grafana-oss:latest

# Với custom config
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -v /path/to/grafana.ini:/etc/grafana/grafana.ini \
  -v grafana-storage:/var/lib/grafana \
  grafana/grafana-oss:latest
```

### Cài Đặt Với Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

  grafana:
    image: grafana/grafana-oss:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus

volumes:
  prometheus-data:
  grafana-data:
```

### Truy Cập Grafana

```
URL: http://localhost:3000
Username: admin
Password: admin (đổi ngay sau khi đăng nhập lần đầu)
```

## Kết Nối Prometheus Data Source

### Qua UI

1. Đăng nhập Grafana
2. Vào **Configuration** (⚙️) → **Data Sources**
3. Click **Add data source**
4. Chọn **Prometheus**
5. Điền thông tin:
   - **Name**: Prometheus (hoặc tên tùy chọn)
   - **URL**: `http://prometheus:9090` (hoặc địa chỉ Prometheus)
   - **Access**: Server (default)
6. Click **Save & Test**

### Qua Provisioning (Infrastructure as Code)

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: "15s"
      queryTimeout: "60s"
      httpMethod: POST
    editable: true
```

### Cấu Hình Data Source Nâng Cao

```yaml
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      # Scrape interval của Prometheus
      timeInterval: "15s"
      # Timeout cho queries
      queryTimeout: "60s"
      # HTTP method (GET hoặc POST)
      httpMethod: POST
      # Exemplars support
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: tempo
    # Basic auth nếu cần
    basicAuth: false
    basicAuthUser: ""
    secureJsonData:
      basicAuthPassword: ""
```

## Tạo Dashboards

### Tạo Dashboard Mới

1. Click **+** (Create) → **Dashboard**
2. Click **Add new panel**
3. Trong panel editor:
   - Chọn **Data source**: Prometheus
   - Nhập **PromQL query**
   - Chọn **Visualization type**
   - Cấu hình **Panel options**
4. Click **Apply**
5. **Save dashboard** (Ctrl+S)

### Ví Dụ Dashboard Node Exporter

```
Panel 1: CPU Usage
  Query: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
  Type: Time series
  Unit: Percent (0-100)

Panel 2: Memory Usage
  Query: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
  Type: Gauge
  Unit: Percent (0-100)
  Thresholds: 70 (yellow), 90 (red)

Panel 3: Disk Usage
  Query: (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100
  Type: Bar gauge
  Unit: Percent (0-100)

Panel 4: Network Traffic
  Query (in): rate(node_network_receive_bytes_total[5m])
  Query (out): rate(node_network_transmit_bytes_total[5m])
  Type: Time series
  Unit: bytes/sec
```

### Import Dashboard Từ Grafana.com

```
1. Vào grafana.com/grafana/dashboards
2. Tìm dashboard phù hợp (ví dụ: Node Exporter Full - ID: 1860)
3. Copy Dashboard ID
4. Trong Grafana: + → Import
5. Nhập Dashboard ID → Load
6. Chọn Prometheus data source
7. Import
```

### Dashboard JSON Model

```json
{
  "title": "My Dashboard",
  "uid": "my-dashboard",
  "panels": [
    {
      "id": 1,
      "title": "CPU Usage",
      "type": "timeseries",
      "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
      "targets": [
        {
          "datasource": {"type": "prometheus"},
          "expr": "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode='idle'}[5m])) * 100)",
          "legendFormat": "{{instance}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "steps": [
              {"color": "green", "value": null},
              {"color": "yellow", "value": 70},
              {"color": "red", "value": 90}
            ]
          }
        }
      }
    }
  ]
}
```

## Panel Types

### Time Series (Graph)

Panel phổ biến nhất, hiển thị dữ liệu theo thời gian:

```
Dùng cho:
- CPU, memory, network usage theo thời gian
- Request rate, error rate
- Latency trends

Cấu hình quan trọng:
- Unit: bytes, percent, requests/sec, ...
- Fill opacity: 0-100
- Line width: 1-10
- Thresholds: màu sắc theo ngưỡng
```

### Stat Panel

Hiển thị một giá trị đơn lẻ với màu sắc:

```
Dùng cho:
- Tổng số requests
- Uptime percentage
- Current value của metric

Cấu hình:
- Reduction: Last, Mean, Max, Min, Sum
- Color mode: Background, Value
- Thresholds: green/yellow/red
```

### Gauge Panel

Hiển thị giá trị dạng đồng hồ đo:

```
Dùng cho:
- CPU usage hiện tại (%)
- Memory usage (%)
- Disk usage (%)

Cấu hình:
- Min/Max values
- Thresholds
- Unit
```

### Bar Gauge

Hiển thị nhiều giá trị dạng thanh ngang/dọc:

```
Dùng cho:
- So sánh nhiều instances
- Top N metrics
- Disk usage per mount point

Cấu hình:
- Orientation: horizontal/vertical
- Display mode: basic/lcd/gradient
```

### Table Panel

Hiển thị dữ liệu dạng bảng:

```
Dùng cho:
- Danh sách instances với nhiều metrics
- Alert status table
- Top N với nhiều columns

Cấu hình:
- Column alignment
- Cell display mode
- Sorting
- Filtering
```

### Heatmap Panel

Hiển thị phân phối dữ liệu theo thời gian:

```
Dùng cho:
- Histogram data (request duration distribution)
- Latency heatmap
- Error distribution

Query ví dụ:
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
```

### Logs Panel

Hiển thị log data (cần Loki data source):

```
Dùng cho:
- Application logs
- System logs
- Audit logs
```

## Variables và Templating

### Tại Sao Dùng Variables

Variables cho phép tạo dynamic dashboards có thể filter theo instance, job, environment, ...

```
Không có variables:
  Dashboard cứng cho server1
  Dashboard cứng cho server2
  → Cần N dashboards cho N servers

Có variables:
  1 dashboard với $instance variable
  → Chọn server từ dropdown
```

### Tạo Variable

1. Vào **Dashboard Settings** (⚙️) → **Variables**
2. Click **Add variable**
3. Cấu hình:

```
Type: Query
Name: instance
Label: Instance
Data source: Prometheus
Query: label_values(up, instance)
Refresh: On Dashboard Load
Multi-value: true
Include All option: true
```

### Các Loại Variable

**Query Variable** - Lấy values từ Prometheus:
```
Query: label_values(node_cpu_seconds_total, instance)
→ Dropdown với tất cả instances
```

**Custom Variable** - Giá trị cố định:
```
Values: production,staging,development
→ Dropdown với 3 môi trường
```

**Constant Variable** - Giá trị không đổi:
```
Value: http://prometheus:9090
→ Dùng trong queries
```

**Interval Variable** - Time intervals:
```
Values: 1m,5m,10m,30m,1h
→ Dùng trong rate() functions
```

### Sử Dụng Variables Trong Queries

```promql
# Dùng $instance variable
node_cpu_seconds_total{instance="$instance", mode="idle"}

# Dùng $job variable
up{job="$job"}

# Dùng $interval variable
rate(http_requests_total[${interval}])

# Multi-value variable (regex)
node_cpu_seconds_total{instance=~"$instance"}
```

### Chaining Variables

```
Variable 1: $environment
  Query: label_values(up{job="node"}, environment)

Variable 2: $instance (phụ thuộc $environment)
  Query: label_values(up{environment="$environment"}, instance)
```

### Repeat Panels

Tự động tạo panel cho mỗi value của variable:

```
Panel settings → Repeat options:
  Repeat by variable: instance
  Max per row: 3
→ Tự động tạo panel cho mỗi instance
```

## Alerting Trong Grafana

### Grafana Alerting vs Prometheus Alerting

| Tính Năng | Prometheus Alerting | Grafana Alerting |
|-----------|--------------------|--------------------|
| Alert definition | Rule files (YAML) | Dashboard panels |
| Data sources | Prometheus only | Multi-source |
| Notification | Alertmanager | Grafana contact points |
| State management | Prometheus | Grafana |
| Visualization | Không | Có (trên panel) |

### Tạo Alert Rule Trong Grafana

1. Mở panel → **Edit**
2. Vào tab **Alert**
3. Click **Create alert rule from this panel**
4. Cấu hình:

```
Alert name: High CPU Usage
Folder: Infrastructure
Group: Node Alerts

Conditions:
  WHEN avg() OF query(A, 5m, now) IS ABOVE 80

No Data & Error Handling:
  If no data or all values are null: No Data
  If execution error or timeout: Alerting

Notifications:
  Contact point: slack-alerts
  Message: CPU usage cao trên {{ $labels.instance }}
```

### Unified Alerting (Grafana 8+)

```yaml
# Grafana provisioning: alerting/rules.yml
apiVersion: 1

groups:
  - orgId: 1
    name: Infrastructure
    folder: Alerts
    interval: 1m
    rules:
      - uid: high-cpu-alert
        title: High CPU Usage
        condition: C
        data:
          - refId: A
            datasourceUid: prometheus
            model:
              expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
              intervalMs: 1000
              maxDataPoints: 43200
          - refId: C
            datasourceUid: __expr__
            model:
              conditions:
                - evaluator:
                    params: [80]
                    type: gt
                  operator:
                    type: and
                  query:
                    params: [A]
                  reducer:
                    type: avg
              type: classic_conditions
        noDataState: NoData
        execErrState: Error
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: CPU usage cao
          description: CPU usage là {{ $value }}%
```

### Contact Points (Notification Channels)

```yaml
# grafana/provisioning/alerting/contact-points.yml
apiVersion: 1

contactPoints:
  - orgId: 1
    name: slack-alerts
    receivers:
      - uid: slack-receiver
        type: slack
        settings:
          url: https://hooks.slack.com/services/xxx
          channel: '#alerts'
          title: '{{ template "slack.default.title" . }}'
          text: '{{ template "slack.default.text" . }}'
        disableResolveMessage: false

  - orgId: 1
    name: email-team
    receivers:
      - uid: email-receiver
        type: email
        settings:
          addresses: team@example.com
          singleEmail: false
```

## Use Cases

### Infrastructure Monitoring Dashboard

```
Panels:
- CPU Usage (Time series) - tất cả nodes
- Memory Usage (Gauge) - per node
- Disk Usage (Bar gauge) - per mount
- Network I/O (Time series) - in/out
- Load Average (Stat) - 1m, 5m, 15m
- Uptime (Stat) - per node
```

### Application Performance Dashboard

```
Panels:
- Request Rate (Time series) - requests/sec
- Error Rate (Time series) - errors/sec
- P50/P95/P99 Latency (Time series)
- Active Connections (Gauge)
- Response Size (Heatmap)
- Top Endpoints (Table)
```

### SLO Dashboard

```
Panels:
- Availability (Stat) - % uptime
- Error Budget Remaining (Gauge)
- Request Success Rate (Time series)
- Latency SLO (Time series)
- Error Budget Burn Rate (Time series)
```

## Cấu Hình

### Grafana Configuration File

```ini
# /etc/grafana/grafana.ini

[server]
http_port = 3000
domain = grafana.example.com
root_url = https://grafana.example.com

[database]
type = sqlite3
path = grafana.db

[security]
admin_user = admin
admin_password = strong-password
secret_key = SW2YcwTIb9zpOOhoPsMm

[users]
allow_sign_up = false
default_theme = dark

[auth.anonymous]
enabled = false

[alerting]
enabled = true
execute_alerts = true

[unified_alerting]
enabled = true
```

### Provisioning Dashboards

```yaml
# grafana/provisioning/dashboards/default.yml
apiVersion: 1

providers:
  - name: Default
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/dashboards
      foldersFromFilesStructure: true
```

### Dashboard JSON File

```bash
# Lưu dashboard vào file
# Trong Grafana UI: Dashboard Settings → JSON Model → Copy

# Đặt file vào thư mục provisioning
cp my-dashboard.json /etc/grafana/dashboards/

# Grafana tự động load khi restart hoặc theo interval
```

### Grafana Với LDAP/OAuth

```ini
# /etc/grafana/grafana.ini

[auth.github]
enabled = true
client_id = your-github-client-id
client_secret = your-github-client-secret
scopes = user:email,read:org
auth_url = https://github.com/login/oauth/authorize
token_url = https://github.com/login/oauth/access_token
api_url = https://api.github.com/user
allowed_organizations = your-org
```

## Best Practices

### 1. Tổ Chức Dashboards

```
Folders:
├── Infrastructure/
│   ├── Node Overview
│   ├── Kubernetes Cluster
│   └── Network
├── Applications/
│   ├── API Service
│   ├── Worker Service
│   └── Database
└── Business/
    ├── User Activity
    └── Revenue Metrics
```

### 2. Naming Conventions

```
Dashboard: [Team] - [Service] - [Type]
Ví dụ: Platform - Node Exporter - Overview

Panel: [Metric] [per/by] [Dimension]
Ví dụ: CPU Usage per Instance
```

### 3. Sử Dụng Variables Cho Flexibility

```
Luôn thêm variables:
- $datasource: Chọn data source
- $environment: production/staging/dev
- $instance: Chọn server
- $interval: Time interval cho rate()
```

### 4. Thresholds Nhất Quán

```
CPU/Memory/Disk:
  Green: < 70%
  Yellow: 70-90%
  Red: > 90%

Latency (ms):
  Green: < 100ms
  Yellow: 100-500ms
  Red: > 500ms

Error Rate:
  Green: < 1%
  Yellow: 1-5%
  Red: > 5%
```

### 5. Dashboard as Code

```bash
# Dùng Grafonnet (Jsonnet library) để tạo dashboards
# https://github.com/grafana/grafonnet

# Hoặc dùng Terraform Grafana provider
terraform {
  required_providers {
    grafana = {
      source = "grafana/grafana"
    }
  }
}

resource "grafana_dashboard" "node_overview" {
  config_json = file("dashboards/node-overview.json")
}
```

### 6. Performance Tips

```
- Sử dụng recording rules cho queries phức tạp
- Đặt refresh interval phù hợp (không quá thấp)
- Giới hạn time range mặc định (last 1h, 6h)
- Dùng $__interval variable thay vì hardcode
- Tránh quá nhiều panels trong một dashboard
```

## Tài Liệu Liên Quan

- [Alerting Rules](./01-alerting-rules.md) - Prometheus alerting rules
- [Alertmanager Config](./02-alertmanager-config.md) - Alertmanager routing
- [Notification Channels](./03-notification-channels.md) - Notification integrations
- [PromQL Basics](../03-promql/01-promql-basics.md) - Viết queries cho Grafana panels
- [Lab 04 - Alertmanager](../07-labs/lab-04-alertmanager/README.md) - Bài tập thực hành

## Tài Liệu Tham Khảo

- [Grafana Documentation](https://grafana.com/docs/grafana/latest/) - Official docs
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/) - Community dashboards
- [Grafana Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/) - Infrastructure as Code
- [Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/) - Unified alerting guide
- [Grafonnet](https://github.com/grafana/grafonnet) - Jsonnet library cho Grafana dashboards
