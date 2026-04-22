# Advanced Queries - Truy Vấn Nâng Cao Trong PromQL

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Subqueries](#subqueries)
- [Recording Rules](#recording-rules)
- [Offset Modifier](#offset-modifier)
- [@ Modifier](#-modifier)
- [Complex Query Patterns](#complex-query-patterns)
- [Query Optimization](#query-optimization)
- [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Giới Thiệu

Advanced PromQL queries cho phép bạn thực hiện phân tích phức tạp, tối ưu performance, và xây dựng monitoring solutions mạnh mẽ. Module này bao gồm:
- **Subqueries** - Queries lồng nhau
- **Recording Rules** - Pre-computed queries
- **Time Modifiers** - Offset và @ operators
- **Complex Patterns** - Kết hợp nhiều techniques
- **Optimization** - Cải thiện query performance

## Subqueries

### Định Nghĩa

Subquery cho phép bạn thực hiện range query trên kết quả của một instant query.

**Cú pháp:**

```promql
<instant_query>[<range>:<resolution>]
```

- `<instant_query>`: Query trả về instant vector
- `<range>`: Time range để evaluate
- `<resolution>`: Step size (optional, mặc định = global evaluation interval)

### Ví Dụ Cơ Bản

**Query:**

```promql
rate(http_requests_total[5m])[10m:1m]
```

**Giải thích:**
1. Inner query: `rate(http_requests_total[5m])` - tính rate
2. Outer query: `[10m:1m]` - evaluate rate trong 10 phút, mỗi 1 phút
3. Kết quả: Range vector với 10 data points (1 point mỗi phút)

### Use Cases

**1. Tính Max Rate Trong Khoảng Thời Gian**

```promql
max_over_time(rate(http_requests_total[5m])[1h:1m])
```

**Giải thích:**
- Tính rate mỗi phút trong 1 giờ
- Tìm max rate trong khoảng đó

**Use case:** Tìm peak traffic trong 1 giờ qua.

**2. Tính Average of Rates**

```promql
avg_over_time(rate(http_requests_total[5m])[30m:])
```

**Giải thích:**
- Tính rate trong 30 phút
- Tính average của các rates đó
- Resolution mặc định (không chỉ định)

**Use case:** Smooth out spikes, tính average traffic.

**3. Detecting Anomalies**

```promql
rate(http_requests_total[5m])
>
avg_over_time(rate(http_requests_total[5m])[1h:1m]) * 1.5
```

**Giải thích:**
- So sánh rate hiện tại với average 1 giờ qua
- Alert nếu rate > 150% average

**Use case:** Phát hiện traffic spikes bất thường.

### Lưu Ý Performance

Subqueries có thể tốn resources:
- Mỗi step trong range phải evaluate inner query
- Range lớn + resolution nhỏ = nhiều evaluations

**Ví dụ:**

```promql
rate(metric[5m])[24h:1m]
```

**Evaluations:** 24 * 60 = 1440 evaluations!

**Best practice:** Sử dụng recording rules thay vì subqueries phức tạp.

## Recording Rules

### Định Nghĩa

Recording rules cho phép bạn pre-compute queries phức tạp và lưu kết quả như metrics mới.

**Lợi ích:**
- Tăng tốc queries phức tạp
- Giảm load trên Prometheus
- Tái sử dụng queries trong nhiều dashboards/alerts

### Cú Pháp

**File cấu hình (prometheus.yml):**

```yaml
rule_files:
  - "rules/*.yml"
```

**Recording rule file (rules/recording_rules.yml):**

```yaml
groups:
  - name: example_recording_rules
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by(job) (rate(http_requests_total[5m]))
```

**Giải thích:**
- `record`: Tên của metric mới
- `expr`: PromQL expression để tính toán
- `interval`: Tần suất evaluate (mặc định = global evaluation_interval)

### Naming Convention

**Format:** `level:metric:operations`

**Ví dụ:**

```yaml
# Instance level
- record: instance:node_cpu:avg_rate5m
  expr: avg by(instance) (rate(node_cpu_seconds_total[5m]))

# Job level
- record: job:http_requests:rate5m
  expr: sum by(job) (rate(http_requests_total[5m]))

# Cluster level
- record: cluster:http_requests:rate5m
  expr: sum(rate(http_requests_total[5m]))
```

**Components:**
- `level`: Aggregation level (instance, job, cluster)
- `metric`: Base metric name
- `operations`: Operations applied (rate5m, sum, avg)

### Ví Dụ Recording Rules

**1. Pre-compute Request Rates**

```yaml
groups:
  - name: http_metrics
    interval: 30s
    rules:
      - record: instance:http_requests:rate5m
        expr: rate(http_requests_total[5m])
      
      - record: job:http_requests:rate5m
        expr: sum by(job) (rate(http_requests_total[5m]))
      
      - record: job:http_errors:rate5m
        expr: sum by(job) (rate(http_requests_total{status=~"5.."}[5m]))
```

**Sử dụng:**

```promql
# Thay vì
sum by(job) (rate(http_requests_total[5m]))

# Dùng
job:http_requests:rate5m
```

**2. Pre-compute Error Rates**

```yaml
- record: job:http_error_rate:ratio
  expr: |
    sum by(job) (rate(http_requests_total{status=~"5.."}[5m]))
    /
    sum by(job) (rate(http_requests_total[5m]))
```

**Sử dụng:**

```promql
job:http_error_rate:ratio > 0.05
```

**3. Pre-compute Resource Usage**

```yaml
- record: instance:node_memory_utilization:ratio
  expr: |
    (
      node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
    )
    /
    node_memory_MemTotal_bytes

- record: instance:node_cpu_utilization:rate5m
  expr: |
    1 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

### Best Practices

**1. Đặt Tên Rõ Ràng**

```yaml
# Tốt
- record: job:http_requests:rate5m
  
# Không tốt
- record: http_rate
```

**2. Document Rules**

```yaml
- record: job:http_error_rate:ratio
  expr: |
    sum by(job) (rate(http_requests_total{status=~"5.."}[5m]))
    /
    sum by(job) (rate(http_requests_total[5m]))
  # Tính error rate percentage cho mỗi job
```

**3. Chọn Interval Phù Hợp**

```yaml
# Fast-changing metrics
- name: realtime_metrics
  interval: 15s
  
# Slow-changing metrics
- name: daily_metrics
  interval: 5m
```

**4. Tránh Over-aggregation**

```yaml
# Tốt - giữ granularity
- record: instance:http_requests:rate5m
  expr: rate(http_requests_total[5m])

# Không tốt - mất thông tin
- record: total:http_requests:rate5m
  expr: sum(rate(http_requests_total[5m]))
```

## Offset Modifier

### Cú Pháp

```promql
<instant_vector> offset <duration>
```

### Ví Dụ

**So sánh với 1 giờ trước:**

```promql
http_requests_total - http_requests_total offset 1h
```

**Giải thích:** Tính số requests tăng thêm trong 1 giờ.

**So sánh rate hiện tại với 1 ngày trước:**

```promql
rate(http_requests_total[5m])
/
rate(http_requests_total[5m] offset 1d)
```

**Giải thích:** Tỷ lệ traffic so với cùng giờ hôm qua.

**Kết quả mẫu:**
```
{instance="server1"} 1.2  # Traffic tăng 20%
{instance="server2"} 0.8  # Traffic giảm 20%
```

### Use Cases

**1. Week-over-Week Comparison**

```promql
sum(rate(http_requests_total[5m]))
/
sum(rate(http_requests_total[5m] offset 1w))
```

**Use case:** So sánh traffic với tuần trước.

**2. Detecting Sudden Changes**

```promql
abs(
  rate(http_requests_total[5m])
  -
  rate(http_requests_total[5m] offset 5m)
) > 10
```

**Use case:** Alert khi traffic thay đổi đột ngột > 10 req/s.

**3. Business Hours Comparison**

```promql
# So sánh với cùng giờ hôm qua
rate(http_requests_total[5m])
/
rate(http_requests_total[5m] offset 24h)
```

**Use case:** Phát hiện anomalies trong business patterns.

## @ Modifier

### Cú Pháp

```promql
<instant_vector> @ <timestamp>
```

Hoặc:

```promql
<instant_vector> @ start()
<instant_vector> @ end()
```

### Ví Dụ

**Query tại timestamp cụ thể:**

```promql
http_requests_total @ 1609459200
```

**Giải thích:** Giá trị tại Unix timestamp 1609459200 (2021-01-01 00:00:00 UTC).

**Query tại start của range:**

```promql
rate(http_requests_total[5m]) @ start()
```

**Use case:** So sánh giá trị đầu và cuối của time range.

**So sánh start vs end:**

```promql
rate(http_requests_total[5m]) @ end()
-
rate(http_requests_total[5m]) @ start()
```

**Giải thích:** Thay đổi rate từ đầu đến cuối time range.

### Kết Hợp @ và offset

```promql
http_requests_total @ 1609459200 offset 1h
```

**Giải thích:** Giá trị tại 1 giờ trước timestamp 1609459200.

## Complex Query Patterns

### Pattern 1: Multi-level Aggregation

**Mục đích:** Tính average của per-instance maximums

```promql
avg(
  max by(instance) (
    rate(http_request_duration_seconds_sum[5m])
    /
    rate(http_request_duration_seconds_count[5m])
  )
)
```

**Giải thích:**
1. Tính average duration per instance
2. Tìm max duration per instance
3. Tính average của các max values

### Pattern 2: Conditional Aggregation

**Mục đích:** Sum chỉ các instances có CPU > 50%

```promql
sum(
  node_memory_used_bytes
  and
  (node_cpu_usage > 50)
)
```

**Giải thích:**
- `and` operator filter chỉ instances có CPU > 50%
- Sum memory của các instances đó

### Pattern 3: Ratio of Ratios

**Mục đích:** Tính error rate của error rate (meta-metric)

```promql
(
  rate(http_requests_total{status=~"5.."}[5m])
  /
  rate(http_requests_total[5m])
)
/
(
  rate(http_requests_total{status=~"5.."}[5m] offset 1h)
  /
  rate(http_requests_total[5m] offset 1h)
)
```

**Giải thích:** So sánh error rate hiện tại với 1 giờ trước.

### Pattern 4: Absent Metrics Detection

**Mục đích:** Alert khi metric không tồn tại

```promql
absent(up{job="api"})
```

**Giải thích:** Trả về 1 nếu không có time-series nào match, 0 nếu có.

**Use case:** Alert khi service không report metrics.

**Kết hợp với or:**

```promql
up{job="api"} == 0 or absent(up{job="api"})
```

**Giải thích:** Alert khi service down HOẶC không report metrics.

### Pattern 5: Predict Linear

**Mục đích:** Dự đoán khi nào hết disk space

```promql
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600) < 0
```

**Giải thích:**
- Dựa trên trend 1 giờ qua
- Dự đoán giá trị sau 4 giờ
- Alert nếu dự đoán < 0 (hết disk)

**Use case:** Proactive alerting trước khi hết resources.

### Pattern 6: Histogram Analysis

**Mục đích:** Tính multiple quantiles cùng lúc

```promql
# P50
histogram_quantile(0.5, sum by(le) (rate(http_request_duration_seconds_bucket[5m])))

# P90
histogram_quantile(0.9, sum by(le) (rate(http_request_duration_seconds_bucket[5m])))

# P99
histogram_quantile(0.99, sum by(le) (rate(http_request_duration_seconds_bucket[5m])))
```

**Visualization:** Multiple lines trên cùng graph.

### Pattern 7: SLO Calculation

**Mục đích:** Tính SLO compliance (99.9% uptime)

```promql
(
  sum(rate(http_requests_total{status!~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
) * 100
```

**Giải thích:**
- Tính success rate trong 30 ngày
- Nhân 100 để ra phần trăm

**Alert khi vi phạm SLO:**

```promql
(
  sum(rate(http_requests_total{status!~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
) < 0.999
```

## Query Optimization

### 1. Sử Dụng Recording Rules

**Chậm:**

```promql
sum by(job) (rate(http_requests_total[5m]))
```

**Nhanh:**

```promql
job:http_requests:rate5m
```

**Lý do:** Pre-computed, không cần tính toán lại.

### 2. Giảm Time Range

**Chậm:**

```promql
rate(http_requests_total[1h])
```

**Nhanh:**

```promql
rate(http_requests_total[5m])
```

**Lý do:** Ít data points hơn để xử lý.

### 3. Sử Dụng Label Matchers

**Chậm:**

```promql
http_requests_total
```

**Nhanh:**

```promql
http_requests_total{job="api"}
```

**Lý do:** Giảm số time-series cần query.

### 4. Aggregate Sớm

**Chậm:**

```promql
sum(rate(http_requests_total[5m])) by(job)
```

**Nhanh:**

```promql
sum by(job) (rate(http_requests_total[5m]))
```

**Lý do:** Aggregate trước khi tính toán phức tạp.

### 5. Tránh Regex Phức Tạp

**Chậm:**

```promql
http_requests_total{path=~".*api.*"}
```

**Nhanh:**

```promql
http_requests_total{path=~"/api/.*"}
```

**Lý do:** Regex cụ thể hơn, ít backtracking.

### 6. Sử Dụng Subqueries Cẩn Thận

**Chậm:**

```promql
max_over_time(rate(metric[5m])[24h:1m])
```

**Nhanh:**

```promql
max_over_time(rate(metric[5m])[1h:5m])
```

**Lý do:** Giảm số evaluations (12 thay vì 1440).

## Ví Dụ Thực Tế

### Ví Dụ 1: SLI/SLO Dashboard

**Mục đích:** Monitor Service Level Indicators

```promql
# Availability SLI (99.9% target)
(
  sum(rate(http_requests_total{status!~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
) * 100

# Latency SLI (P95 < 200ms)
histogram_quantile(0.95,
  sum by(le) (rate(http_request_duration_seconds_bucket[5m]))
) < 0.2

# Error Budget Remaining
1 - (
  (1 - 0.999) - 
  (
    sum(rate(http_requests_total{status=~"5.."}[30d]))
    /
    sum(rate(http_requests_total[30d]))
  )
) / (1 - 0.999)
```

### Ví Dụ 2: Capacity Planning

**Mục đích:** Dự đoán khi nào cần scale

```promql
# Dự đoán CPU usage sau 7 ngày
predict_linear(
  avg(node_cpu_usage)[7d:1h],
  7 * 24 * 3600
) > 80

# Dự đoán memory usage
predict_linear(
  (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)[7d:1h],
  7 * 24 * 3600
) / node_memory_MemTotal_bytes > 0.9
```

### Ví Dụ 3: Anomaly Detection

**Mục đích:** Phát hiện traffic bất thường

```promql
# Traffic > 2x standard deviation
abs(
  rate(http_requests_total[5m])
  -
  avg_over_time(rate(http_requests_total[5m])[1h:1m])
) > 2 * stddev_over_time(rate(http_requests_total[5m])[1h:1m])
```

### Ví Dụ 4: Multi-Window Analysis

**Mục đích:** So sánh multiple time windows

```promql
# Current vs 1h ago vs 1d ago
rate(http_requests_total[5m])
/
(
  rate(http_requests_total[5m] offset 1h)
  +
  rate(http_requests_total[5m] offset 1d)
) * 0.5
```

**Giải thích:** So sánh với average của 1h trước và 1d trước.

### Ví Dụ 5: Composite Health Score

**Mục đích:** Tính overall health score

```promql
(
  # Availability (40%)
  (avg(up) * 40)
  +
  # Low error rate (30%)
  ((1 - (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])))) * 30)
  +
  # Low latency (20%)
  ((histogram_quantile(0.95, sum by(le) (rate(http_request_duration_seconds_bucket[5m]))) < 0.2) * 20)
  +
  # Low CPU (10%)
  ((avg(node_cpu_usage) < 0.8) * 10)
)
```

**Kết quả:** Score từ 0-100.

## Best Practices

### 1. Comment Complex Queries

```promql
# Calculate error rate percentage
# Errors / Total requests * 100
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) * 100
```

### 2. Break Down Complex Queries

**Thay vì một query khổng lồ:**

```promql
# Sử dụng recording rules
- record: job:http_errors:rate5m
  expr: sum by(job) (rate(http_requests_total{status=~"5.."}[5m]))

- record: job:http_requests:rate5m
  expr: sum by(job) (rate(http_requests_total[5m]))

# Query đơn giản
job:http_errors:rate5m / job:http_requests:rate5m
```

### 3. Test Queries Incrementally

```promql
# Step 1: Base query
http_requests_total

# Step 2: Add rate
rate(http_requests_total[5m])

# Step 3: Add aggregation
sum by(job) (rate(http_requests_total[5m]))

# Step 4: Add calculation
sum by(job) (rate(http_requests_total[5m])) / 1000
```

### 4. Use Variables in Grafana

```promql
# Thay vì hardcode
rate(http_requests_total{job="api"}[5m])

# Dùng variable
rate(http_requests_total{job="$job"}[5m])
```

### 5. Document Recording Rules

```yaml
groups:
  - name: http_metrics
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by(job) (rate(http_requests_total[5m]))
        labels:
          team: platform
        annotations:
          description: "Request rate per job, 5m average"
          runbook_url: "https://wiki.example.com/runbooks/http-metrics"
```

## Troubleshooting

### Vấn Đề: Query Timeout

**Triệu chứng:** Query mất quá nhiều thời gian

**Giải pháp:**
1. Giảm time range
2. Thêm label matchers
3. Sử dụng recording rules
4. Tăng query timeout trong config

### Vấn Đề: Subquery Trả Về Empty

**Triệu chứng:** Subquery không có kết quả

**Nguyên nhân:** Resolution không phù hợp

**Giải pháp:** Chỉ định resolution explicitly

```promql
rate(metric[5m])[1h:1m]
```

### Vấn Đề: Recording Rule Không Update

**Triệu chứng:** Recorded metric có giá trị cũ

**Giải pháp:**
1. Kiểm tra rule syntax
2. Reload Prometheus config
3. Kiểm tra logs cho errors

```bash
promtool check rules rules/*.yml
curl -X POST http://localhost:9090/-/reload
```

## Tài Liệu Liên Quan

- [Functions](./04-functions.md) - Functions sử dụng trong advanced queries
- [Aggregations](./05-aggregations.md) - Aggregation techniques
- [Operators](./03-operators.md) - Operators cho complex expressions
- [Alerting Rules](../05-alerting/01-alerting-rules.md) - Sử dụng advanced queries trong alerts

## Tài Liệu Tham Khảo

- [Official PromQL Documentation](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Recording Rules Documentation](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
- [Query Performance Tips](https://prometheus.io/docs/prometheus/latest/querying/basics/#avoiding-slow-queries-and-overloads)
- [PromQL Quick Reference](../09-reference/01-promql-quick-reference.md)
