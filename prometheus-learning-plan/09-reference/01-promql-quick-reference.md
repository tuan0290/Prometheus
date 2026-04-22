# PromQL Quick Reference

## Selectors

```promql
metric_name                          # All time series
metric_name{label="value"}           # Exact match
metric_name{label!="value"}          # Not equal
metric_name{label=~"regex"}          # Regex match
metric_name{label!~"regex"}          # Regex not match
metric_name[5m]                      # Range vector (last 5 min)
metric_name offset 1h                # Value 1 hour ago
metric_name @ 1609459200             # Value at timestamp
```

## Arithmetic Operators

| Operator | Ví Dụ |
|----------|-------|
| `+` | `metric_a + metric_b` |
| `-` | `total - available` |
| `*` | `rate * 60` |
| `/` | `available / total * 100` |
| `%` | `metric % 100` |
| `^` | `metric ^ 2` |

## Comparison Operators

| Operator | Ý Nghĩa |
|----------|---------|
| `==` | Equal |
| `!=` | Not equal |
| `>` | Greater than |
| `<` | Less than |
| `>=` | Greater or equal |
| `<=` | Less or equal |

## Logical Operators

```promql
metric_a and metric_b    # Both present
metric_a or metric_b     # Either present
metric_a unless metric_b # Left minus right
```

## Aggregation Operators

| Operator | Mô Tả |
|----------|-------|
| `sum` | Sum values |
| `avg` | Average |
| `max` | Maximum |
| `min` | Minimum |
| `count` | Count time series |
| `stddev` | Standard deviation |
| `stdvar` | Standard variance |
| `topk(k, expr)` | Top k values |
| `bottomk(k, expr)` | Bottom k values |
| `quantile(φ, expr)` | φ-quantile |
| `count_values("label", expr)` | Count by value |

```promql
sum by (job) (metric)          # Group by job
sum without (instance) (metric) # Remove instance label
topk(5, sum by (job) (metric)) # Top 5 jobs
```

## Functions

### Counter Functions

| Function | Mô Tả |
|----------|-------|
| `rate(v[d])` | Per-second average rate |
| `irate(v[d])` | Instant per-second rate |
| `increase(v[d])` | Total increase |
| `resets(v[d])` | Number of counter resets |

### Gauge Functions

| Function | Mô Tả |
|----------|-------|
| `delta(v[d])` | Change over time |
| `idelta(v[d])` | Instant change |
| `deriv(v[d])` | Per-second derivative |
| `predict_linear(v[d], t)` | Predict value in t seconds |

### Aggregation over Time

| Function | Mô Tả |
|----------|-------|
| `avg_over_time(v[d])` | Average |
| `min_over_time(v[d])` | Minimum |
| `max_over_time(v[d])` | Maximum |
| `sum_over_time(v[d])` | Sum |
| `count_over_time(v[d])` | Count |
| `stddev_over_time(v[d])` | Std deviation |
| `last_over_time(v[d])` | Last value |

### Math Functions

| Function | Mô Tả |
|----------|-------|
| `abs(v)` | Absolute value |
| `ceil(v)` | Round up |
| `floor(v)` | Round down |
| `round(v, to)` | Round to nearest |
| `sqrt(v)` | Square root |
| `exp(v)` | e^v |
| `ln(v)` | Natural log |
| `log2(v)` | Log base 2 |
| `log10(v)` | Log base 10 |
| `clamp(v, min, max)` | Clamp value |
| `clamp_min(v, min)` | Minimum clamp |
| `clamp_max(v, max)` | Maximum clamp |

### Histogram Functions

```promql
# Quantile from histogram
histogram_quantile(0.95, rate(metric_bucket[5m]))

# With grouping
histogram_quantile(0.99,
  sum by (job, le) (rate(metric_bucket[5m]))
)
```

### Label Functions

```promql
# Replace label value
label_replace(v, "dst_label", "replacement", "src_label", "regex")

# Join labels
label_join(v, "dst_label", "separator", "src_label1", "src_label2")
```

### Utility Functions

| Function | Mô Tả |
|----------|-------|
| `absent(v)` | 1 if no time series |
| `absent_over_time(v[d])` | 1 if no data in range |
| `changes(v[d])` | Number of value changes |
| `timestamp(v)` | Timestamp of each sample |
| `time()` | Current Unix timestamp |
| `vector(s)` | Scalar to vector |
| `scalar(v)` | Vector to scalar |
| `sort(v)` | Sort ascending |
| `sort_desc(v)` | Sort descending |

## Common Patterns

### CPU Usage

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Memory Usage

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

### Disk Usage

```promql
(node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100
```

### Error Rate

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100
```

### Latency p95

```promql
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

### Availability

```promql
avg(up{job="my-service"}) * 100
```

### Throughput

```promql
sum(rate(http_requests_total[5m]))
```

### Predict Disk Full

```promql
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600)
```

### Week-over-Week Comparison

```promql
rate(http_requests_total[5m]) / rate(http_requests_total[5m] offset 1w)
```

## Subqueries

```promql
# Max rate over last hour
max_over_time(rate(metric[5m])[1h:])

# Average of 5-min rates over 1 hour
avg_over_time(rate(metric[5m])[1h:1m])
```

Format: `expr[range:resolution]`

## Recording Rules

```yaml
groups:
  - name: example
    rules:
      - record: job:metric:rate5m
        expr: sum by (job) (rate(metric[5m]))
```

Naming: `level:metric:operations`
