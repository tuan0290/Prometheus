# Domain 3: PromQL (28%)

## Tổng Quan

Domain này chiếm 28% kỳ thi PCA - trọng số cao nhất. Tập trung vào selectors, operators, functions, aggregations, và advanced queries.

## 1. Selectors và Matchers

### Instant Vector Selector

```promql
# Select all time series
http_requests_total

# Filter by label (exact match)
http_requests_total{job="api"}

# Multiple labels
http_requests_total{job="api", status="200"}
```

### Label Matchers

| Matcher | Ý Nghĩa | Ví Dụ |
|---------|---------|-------|
| `=` | Exact match | `{job="api"}` |
| `!=` | Not equal | `{status!="500"}` |
| `=~` | Regex match | `{status=~"2.."}` |
| `!~` | Regex not match | `{status!~"5.."}` |

```promql
# All 2xx responses
http_requests_total{status=~"2.."}

# All non-error responses
http_requests_total{status!~"5.."}

# Multiple jobs
up{job=~"api|web|db"}
```

### Range Vector Selector

```promql
# Last 5 minutes
http_requests_total[5m]

# Last 1 hour
node_cpu_seconds_total[1h]

# Last 30 seconds
up[30s]
```

### Offset Modifier

```promql
# Value 1 hour ago
http_requests_total offset 1h

# Rate 1 week ago
rate(http_requests_total[5m] offset 1w)
```

## 2. Data Types

| Type | Mô Tả | Ví Dụ |
|------|-------|-------|
| Instant Vector | Set of time series, single value each | `up` |
| Range Vector | Set of time series, range of values | `up[5m]` |
| Scalar | Single numeric value | `1.5` |
| String | String value (rarely used) | `"hello"` |

## 3. Operators

### Arithmetic Operators

```promql
# Addition
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# Division (percentage)
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# Multiplication
rate(http_requests_total[5m]) * 60  # per minute
```

### Comparison Operators

```promql
# Greater than (filter)
node_memory_MemAvailable_bytes > 1e9

# Less than or equal
http_request_duration_seconds < 0.5

# Equal
up == 1
```

### Logical Operators

```promql
# AND: both conditions true
http_requests_total > 100 and http_requests_total < 1000

# OR: either condition true
http_requests_total{status="500"} or http_requests_total{status="503"}

# UNLESS: left minus right
http_requests_total unless http_requests_total{status="200"}
```

### Vector Matching

```promql
# One-to-one matching
method_code:http_errors:rate5m / ignoring(code) method:http_requests:rate5m

# Many-to-one matching
method_code:http_errors:rate5m / on(method) group_left method:http_requests:rate5m
```

## 4. Functions

### Rate và Increase

```promql
# Per-second rate (for counters)
rate(http_requests_total[5m])

# Total increase over time window
increase(http_requests_total[1h])

# Per-second rate (allows negative - for gauges)
irate(http_requests_total[5m])  # Instant rate (last 2 samples)
```

**Quan trọng**: Luôn dùng `rate()` với counters, không query counter trực tiếp.

### Aggregation over Time

```promql
# Average over time
avg_over_time(node_memory_MemAvailable_bytes[1h])

# Max over time
max_over_time(node_cpu_seconds_total[1h])

# Min over time
min_over_time(node_memory_MemAvailable_bytes[24h])

# Sum over time
sum_over_time(http_requests_total[1h])
```

### Histogram Functions

```promql
# Calculate quantile from histogram
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# With grouping
histogram_quantile(0.99,
  sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
)
```

### Math Functions

```promql
# Absolute value
abs(delta(cpu_temp_celsius[10m]))

# Ceiling/Floor
ceil(node_memory_MemAvailable_bytes / 1e9)
floor(node_memory_MemAvailable_bytes / 1e9)

# Round
round(node_memory_MemAvailable_bytes / 1e9, 0.1)

# Clamp
clamp_min(node_memory_MemAvailable_bytes, 0)
clamp_max(node_memory_MemAvailable_bytes, 1e10)
```

### Predict Linear

```promql
# Predict value in 4 hours based on last 1 hour trend
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600)
```

### Absent

```promql
# Returns 1 if metric is absent (useful for alerting)
absent(up{job="critical-service"})
```

### Label Functions

```promql
# Get label value as string
label_replace(up, "host", "$1", "instance", "(.*):.*")

# Join labels
label_join(up, "address", ":", "instance", "job")
```

## 5. Aggregation Operators

### Basic Aggregations

```promql
# Sum all
sum(http_requests_total)

# Average
avg(node_memory_MemAvailable_bytes)

# Maximum
max(node_cpu_seconds_total)

# Minimum
min(node_memory_MemAvailable_bytes)

# Count time series
count(up)

# Standard deviation
stddev(http_request_duration_seconds)
```

### Grouping với by/without

```promql
# Sum by job
sum by (job) (http_requests_total)

# Sum by multiple labels
sum by (job, status) (http_requests_total)

# Sum without instance (keep all other labels)
sum without (instance) (http_requests_total)
```

### TopK và BottomK

```promql
# Top 5 endpoints by request count
topk(5, sum by (endpoint) (http_requests_total))

# Bottom 3 by availability
bottomk(3, avg by (job) (up))
```

### Quantile

```promql
# 95th percentile across instances
quantile(0.95, http_request_duration_seconds)
```

## 6. Subqueries

```promql
# Max rate over last hour, evaluated every minute
max_over_time(rate(http_requests_total[5m])[1h:1m])

# Average of 5-minute rates over 1 hour
avg_over_time(rate(http_requests_total[5m])[1h:])
```

Format: `<query>[<range>:<resolution>]`

## 7. Recording Rules

Pre-compute expensive queries:

```yaml
groups:
  - name: example
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: instance:cpu_usage:percent
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Naming convention**: `level:metric:operations`

## 8. Common Query Patterns

### Error Rate

```promql
# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m])) /
sum(rate(http_requests_total[5m])) * 100
```

### Availability

```promql
# Service availability
avg(up{job="my-service"}) * 100
```

### CPU Usage

```promql
# CPU usage percentage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Memory Usage

```promql
# Memory usage percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

### Disk Usage

```promql
# Disk usage percentage
(node_filesystem_size_bytes - node_filesystem_avail_bytes) /
node_filesystem_size_bytes * 100
```

### Latency Percentiles

```promql
# p50, p95, p99
histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

## 9. Quick Reference

### Functions Cheat Sheet

| Function | Input | Output | Use Case |
|----------|-------|--------|----------|
| `rate()` | Counter range vector | Instant vector | Per-second rate |
| `increase()` | Counter range vector | Instant vector | Total increase |
| `irate()` | Counter range vector | Instant vector | Instant rate |
| `delta()` | Gauge range vector | Instant vector | Change over time |
| `avg_over_time()` | Range vector | Instant vector | Average over time |
| `histogram_quantile()` | Float, histogram | Instant vector | Percentile |
| `predict_linear()` | Range vector, seconds | Instant vector | Prediction |
| `absent()` | Instant vector | Instant vector | Alert on missing |

## 10. Practice Questions

**Q1**: Sự khác biệt giữa `rate()` và `irate()` là gì?

<details>
<summary>Đáp án</summary>

`rate()` tính per-second rate trung bình trong toàn bộ time range, smooth hơn, phù hợp cho alerting. `irate()` tính instant rate dựa trên 2 samples cuối cùng, responsive hơn với changes đột ngột nhưng volatile hơn. Dùng `rate()` cho alerting và dashboards, `irate()` khi cần detect spikes nhanh.

</details>

**Q2**: Tại sao cần dùng `sum by (job, le)` khi tính histogram_quantile?

<details>
<summary>Đáp án</summary>

Khi aggregate histogram buckets across multiple instances, cần giữ lại label `le` (less than or equal) vì đây là label xác định bucket boundaries. Nếu không giữ `le`, các buckets sẽ bị merge không đúng cách và quantile calculation sẽ sai.

</details>

**Q3**: Khi nào dùng `without` thay vì `by`?

<details>
<summary>Đáp án</summary>

Dùng `by` khi muốn giữ lại một số labels cụ thể. Dùng `without` khi muốn loại bỏ một số labels và giữ tất cả labels còn lại. `without` tiện hơn khi có nhiều labels và chỉ muốn loại bỏ 1-2 labels như `instance`.

</details>

## Tiếp Theo

- [Domain 4: Instrumentation](./06-domain-instrumentation.md)
- [Tài liệu tham khảo: PromQL](../03-promql/)
