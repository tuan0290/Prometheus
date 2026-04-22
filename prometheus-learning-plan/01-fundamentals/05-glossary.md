# Bảng Thuật Ngữ Prometheus

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [A](#a)
- [B](#b)
- [C](#c)
- [D](#d)
- [E](#e)
- [F](#f)
- [G](#g)
- [H](#h)
- [I](#i)
- [J](#j)
- [L](#l)
- [M](#m)
- [N](#n)
- [O](#o)
- [P](#p)
- [Q](#q)
- [R](#r)
- [S](#s)
- [T](#t)
- [U](#u)
- [V](#v)
- [W](#w)

## Giới Thiệu

Bảng thuật ngữ này cung cấp định nghĩa cho các khái niệm và thuật ngữ quan trọng trong Prometheus. Các thuật ngữ được sắp xếp theo thứ tự bảng chữ cái để dễ tra cứu.

---

## A

### Alert

**Định nghĩa**: Một cảnh báo được kích hoạt khi một điều kiện (alerting rule) được đáp ứng.

**Ví dụ**:
```yaml
alert: HighCPUUsage
expr: cpu_usage_percent > 80
for: 5m
```

**Trạng thái alert**:
- **Inactive**: Điều kiện không đúng
- **Pending**: Điều kiện đúng nhưng chưa đủ thời gian `for`
- **Firing**: Điều kiện đúng và đã vượt qua thời gian `for`

### Alertmanager

**Định nghĩa**: Thành phần xử lý alerts từ Prometheus, thực hiện grouping, inhibition, silencing, và routing đến các notification channels.

**Chức năng chính**:
- Grouping: Gom các alerts tương tự
- Inhibition: Tắt alerts phụ thuộc
- Silencing: Tạm tắt alerts
- Routing: Gửi đến đúng receiver

### Alerting Rule

**Định nghĩa**: Quy tắc định nghĩa điều kiện để kích hoạt alert.

**Cú pháp**:
```yaml
groups:
  - name: example
    rules:
      - alert: HighErrorRate
        expr: rate(errors_total[5m]) > 0.05
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
```

---

## B

### Bucket

**Định nghĩa**: Trong histogram, một bucket là một khoảng giá trị được định nghĩa trước để phân loại observations.

**Ví dụ**:
```
Buckets: [0.1, 0.5, 1.0, 5.0, +Inf]

http_request_duration_seconds_bucket{le="0.1"}  100
http_request_duration_seconds_bucket{le="0.5"}  250
http_request_duration_seconds_bucket{le="1.0"}  400
```

**Giải thích**: 
- 100 requests ≤ 0.1s
- 250 requests ≤ 0.5s (cumulative)
- 400 requests ≤ 1.0s (cumulative)

---

## C

### Cardinality

**Định nghĩa**: Số lượng unique time series trong một metric. High cardinality có thể gây vấn đề về performance và storage.

**Ví dụ**:
```
http_requests_total{method="GET", path="/api"}
http_requests_total{method="POST", path="/api"}
http_requests_total{method="GET", path="/users"}

Cardinality = 3 (3 unique combinations)
```

**Best practice**: Tránh labels với giá trị unbounded (user_id, email, etc.)

### Client Library

**Định nghĩa**: Thư viện để instrument application code và expose metrics cho Prometheus.

**Ngôn ngữ hỗ trợ**:
- Go: `github.com/prometheus/client_golang`
- Python: `prometheus_client`
- Java: `io.prometheus:simpleclient`
- Ruby: `prometheus-client`
- .NET: `prometheus-net`

### Counter

**Định nghĩa**: Loại metric chỉ tăng (hoặc reset về 0). Dùng để đếm số lần một sự kiện xảy ra.

**Đặc điểm**:
- Monotonically increasing
- Có thể reset về 0 khi restart
- Không bao giờ giảm

**Ví dụ**:
```
http_requests_total
errors_total
bytes_sent_total
```

---

## D

### Data Model

**Định nghĩa**: Cách Prometheus tổ chức và lưu trữ metrics. Mỗi time series được xác định bởi metric name và labels.

**Cấu trúc**:
```
<metric_name>{<label_name>=<label_value>, ...}

Ví dụ:
http_requests_total{method="GET", path="/api", status="200"}
```

### Downsampling

**Định nghĩa**: Giảm resolution của time-series data để tiết kiệm storage.

**Ví dụ**:
```
Original: 1 sample/15s (high resolution)
Downsampled: 1 sample/5m (lower resolution)
```

---

## E

### Endpoint

**Định nghĩa**: URL mà Prometheus scrape metrics từ đó, thường là `/metrics`.

**Ví dụ**:
```
http://localhost:9100/metrics
http://app-server:8080/metrics
```

### Evaluation Interval

**Định nghĩa**: Tần suất Prometheus đánh giá alerting và recording rules.

**Cấu hình**:
```yaml
global:
  evaluation_interval: 15s
```

### Exporter

**Định nghĩa**: Chương trình expose metrics từ hệ thống không hỗ trợ Prometheus native format.

**Ví dụ**:
- **Node Exporter**: System metrics (CPU, memory, disk)
- **Blackbox Exporter**: Probe endpoints (HTTP, TCP, ICMP)
- **MySQL Exporter**: MySQL database metrics

---

## F

### Federation

**Định nghĩa**: Cơ chế cho phép một Prometheus server scrape metrics từ Prometheus server khác.

**Use case**: Hierarchical monitoring, cross-datacenter aggregation

**Endpoint**: `/federate`

---

## G

### Gauge

**Định nghĩa**: Loại metric có thể tăng hoặc giảm. Dùng để đo giá trị hiện tại.

**Đặc điểm**:
- Có thể tăng hoặc giảm
- Đại diện giá trị snapshot

**Ví dụ**:
```
cpu_usage_percent
memory_available_bytes
active_connections
temperature_celsius
```

### Grouping

**Định nghĩa**: Trong Alertmanager, gom các alerts tương tự thành một notification.

**Ví dụ**:
```yaml
route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
```

---

## H

### Histogram

**Định nghĩa**: Loại metric đo phân phối của các giá trị trong các buckets định trước.

**Tạo ra 3 metrics**:
```
<metric>_bucket{le="<upper_bound>"}
<metric>_sum
<metric>_count
```

**Use case**: Request latency, response size

---

## I

### Inhibition

**Định nghĩa**: Trong Alertmanager, tắt một số alerts khi alert khác đang firing.

**Ví dụ**:
```yaml
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

**Giải thích**: Tắt warning alerts khi có critical alert cho cùng instance.

### Instance

**Định nghĩa**: Một target cụ thể mà Prometheus scrape metrics từ đó. Được xác định bởi `<host>:<port>`.

**Ví dụ**:
```
localhost:9100
server1.example.com:9100
192.168.1.10:8080
```

### Instant Vector

**Định nghĩa**: Tập hợp các time series, mỗi series có một sample tại cùng một timestamp.

**Ví dụ PromQL**:
```promql
http_requests_total

Result:
http_requests_total{method="GET"} 1234 @1610000000
http_requests_total{method="POST"} 567 @1610000000
```

---

## J

### Job

**Định nghĩa**: Tập hợp các instances có cùng mục đích (cùng role).

**Ví dụ**:
```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets:
        - 'server1:9100'
        - 'server2:9100'
        - 'server3:9100'
```

Tất cả instances trên đều thuộc job `node`.

---

## L

### Label

**Định nghĩa**: Cặp key-value để phân biệt các dimensions của một metric.

**Ví dụ**:
```
http_requests_total{
  method="GET",
  path="/api",
  status="200",
  instance="localhost:8080"
}
```

**Best practices**:
- Dùng snake_case cho label names
- Tránh high cardinality labels
- Không dùng labels cho giá trị unbounded

### Label Matcher

**Định nghĩa**: Điều kiện để filter time series dựa trên labels.

**Operators**:
- `=`: Equal
- `!=`: Not equal
- `=~`: Regex match
- `!~`: Regex not match

**Ví dụ**:
```promql
http_requests_total{method="GET"}
http_requests_total{status=~"5.."}
http_requests_total{path!="/health"}
```

---

## M

### Metric

**Định nghĩa**: Một measurement được thu thập theo thời gian.

**Cấu trúc**:
```
<metric_name>{<labels>} <value> <timestamp>

Ví dụ:
http_requests_total{method="GET"} 1234 1610000000
```

### Metric Name

**Định nghĩa**: Tên của metric, nên mô tả rõ ràng điều gì được đo.

**Naming conventions**:
```
<namespace>_<name>_<unit>_<suffix>

Ví dụ:
http_requests_total
process_cpu_seconds_total
node_memory_available_bytes
```

### MTTR

**Định nghĩa**: Mean Time To Resolution - Thời gian trung bình để khắc phục sự cố.

**Công thức**:
```
MTTR = Tổng thời gian khắc phục / Số lượng sự cố
```

---

## N

### Notification

**Định nghĩa**: Thông báo được gửi bởi Alertmanager khi alert firing.

**Channels**:
- Email
- Slack
- PagerDuty
- Webhook
- OpsGenie

---

## O

### Observability

**Định nghĩa**: Khả năng hiểu được trạng thái bên trong của hệ thống dựa trên dữ liệu đầu ra.

**Ba trụ cột**:
1. **Metrics**: Dữ liệu số theo thời gian
2. **Logs**: Bản ghi sự kiện chi tiết
3. **Traces**: Theo dõi request qua nhiều services

---

## P

### PromQL

**Định nghĩa**: Prometheus Query Language - Ngôn ngữ truy vấn của Prometheus.

**Ví dụ**:
```promql
# Instant query
http_requests_total

# Range query
rate(http_requests_total[5m])

# Aggregation
sum(rate(http_requests_total[5m])) by (status)
```

### Pull Model

**Định nghĩa**: Mô hình Prometheus chủ động scrape (pull) metrics từ targets, thay vì targets push metrics.

**Ưu điểm**:
- Prometheus kiểm soát scrape rate
- Dễ phát hiện target down
- Targets không cần biết Prometheus

### Pushgateway

**Định nghĩa**: Thành phần cho phép short-lived jobs push metrics vào, sau đó Prometheus scrape từ Pushgateway.

**Use case**: Batch jobs, cron jobs

**Lưu ý**: Không dùng cho long-running services

---

## Q

### Quantile

**Định nghĩa**: Giá trị phân vị trong phân phối dữ liệu.

**Ví dụ**:
- **p50 (median)**: 50% giá trị ≤ p50
- **p95**: 95% giá trị ≤ p95
- **p99**: 99% giá trị ≤ p99

**PromQL**:
```promql
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m])
)
```

---

## R

### Range Vector

**Định nghĩa**: Tập hợp các time series, mỗi series có nhiều samples trong một khoảng thời gian.

**Ví dụ PromQL**:
```promql
http_requests_total[5m]

Result:
http_requests_total{method="GET"} 
  1000 @1610000000
  1050 @1610000015
  1120 @1610000030
  ...
```

### Rate

**Định nghĩa**: Tốc độ tăng per second của một counter.

**PromQL**:
```promql
rate(http_requests_total[5m])
```

**Công thức**:
```
rate = (value_end - value_start) / time_range_seconds
```

### Recording Rule

**Định nghĩa**: Quy tắc tính toán trước và lưu kết quả của một PromQL query phức tạp.

**Ví dụ**:
```yaml
groups:
  - name: example
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
```

**Use case**: Tối ưu queries phức tạp, tính toán trước aggregations

### Relabeling

**Định nghĩa**: Thay đổi labels của time series trước hoặc sau khi scrape.

**Actions**:
- `replace`: Thay thế label value
- `keep`: Giữ lại series matching
- `drop`: Bỏ series matching
- `labelmap`: Map labels
- `labeldrop`: Xóa labels

### Retention

**Định nghĩa**: Thời gian Prometheus lưu trữ dữ liệu trước khi xóa.

**Cấu hình**:
```yaml
storage:
  tsdb:
    retention.time: 15d
    retention.size: 50GB
```

---

## S

### Sample

**Định nghĩa**: Một điểm dữ liệu đơn lẻ trong time series, gồm timestamp và value.

**Cấu trúc**:
```
timestamp: 1610000000
value: 123.45
```

### Scalar

**Định nghĩa**: Một giá trị số đơn lẻ, không có time series.

**Ví dụ PromQL**:
```promql
42
3.14
count(up)
```

### Scrape

**Định nghĩa**: Quá trình Prometheus pull metrics từ một target.

**Workflow**:
```
1. HTTP GET /metrics
2. Parse Prometheus format
3. Add labels (job, instance)
4. Store in TSDB
```

### Scrape Interval

**Định nghĩa**: Tần suất Prometheus scrape metrics từ targets.

**Cấu hình**:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    scrape_interval: 30s  # Override global
```

### Service Discovery

**Định nghĩa**: Cơ chế tự động phát hiện targets để scrape.

**Mechanisms**:
- Static config
- File-based
- Kubernetes
- Consul
- EC2, Azure, GCE
- DNS

### Silencing

**Định nghĩa**: Tạm thời tắt notifications cho alerts matching điều kiện.

**Use case**: Maintenance windows, known issues

### Summary

**Định nghĩa**: Loại metric tính quantiles chính xác ở client-side.

**Tạo ra metrics**:
```
<metric>{quantile="0.5"}
<metric>{quantile="0.9"}
<metric>{quantile="0.99"}
<metric>_sum
<metric>_count
```

**Lưu ý**: Không thể aggregate giữa nhiều instances

---

## T

### Target

**Định nghĩa**: Một endpoint mà Prometheus scrape metrics từ đó.

**Ví dụ**:
```
http://localhost:9100/metrics
http://app:8080/metrics
```

### Time Series

**Định nghĩa**: Chuỗi các data points được sắp xếp theo thời gian, được xác định bởi metric name và labels.

**Ví dụ**:
```
http_requests_total{method="GET", path="/api"}
  1000 @10:00:00
  1050 @10:00:15
  1120 @10:00:30
  1180 @10:00:45
```

### TSDB

**Định nghĩa**: Time-Series Database - Cơ sở dữ liệu được tối ưu cho time-series data.

**Đặc điểm**:
- Write-heavy workload
- Immutable data
- Time-ordered
- Efficient compression

---

## U

### Up

**Định nghĩa**: Metric tự động được Prometheus tạo ra để chỉ trạng thái của target.

**Giá trị**:
- `1`: Target up (scrape thành công)
- `0`: Target down (scrape thất bại)

**Ví dụ**:
```promql
up{job="node", instance="localhost:9100"} 1
```

---

## V

### Vector

**Định nghĩa**: Tập hợp các time series.

**Loại**:
- **Instant Vector**: Một sample per series
- **Range Vector**: Nhiều samples per series

---

## W

### WAL

**Định nghĩa**: Write-Ahead Log - Log ghi trước để đảm bảo durability khi crash.

**Vị trí**:
```
prometheus/data/wal/
```

**Chức năng**: Recovery data khi Prometheus restart

---

## Tài Liệu Liên Quan

- [Monitoring and Observability](./01-monitoring-observability.md) - Khái niệm cơ bản
- [Time-Series Databases](./02-time-series-databases.md) - TSDB và cách hoạt động
- [Metrics Types](./03-metrics-types.md) - Counter, Gauge, Histogram, Summary
- [Prometheus Architecture](./04-prometheus-architecture.md) - Kiến trúc tổng thể

## Tài Liệu Tham Khảo

- [Prometheus Glossary](https://prometheus.io/docs/introduction/glossary/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [PromQL Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
