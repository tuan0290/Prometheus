# Lab 03 - PromQL Queries Instructions

## Mục Tiêu

Trong lab này, bạn sẽ thực hành viết PromQL queries từ cơ bản đến nâng cao. Bạn sẽ học cách query, filter, aggregate, và analyze metrics data.

## Yêu Cầu

- Lab 01 và Lab 02 đã hoàn thành
- Prometheus running với data
- node_exporter và custom app running
- Browser để access Prometheus UI

## Setup: Generate Sample Data

Trước khi bắt đầu, generate traffic để có data:

```bash
# Terminal 1: Generate continuous traffic
while true; do
    curl -s http://localhost:8080/ > /dev/null
    curl -s http://localhost:8080/work > /dev/null
    curl -s http://localhost:8080/process > /dev/null
    sleep 2
done &

# Save PID để stop sau
TRAFFIC_PID=$!
echo "Traffic generator PID: $TRAFFIC_PID"
```

Đợi 5-10 phút để có đủ data, sau đó bắt đầu exercises.

## Phần 1: Basic Queries

### Exercise 1.1: Instant Vector Selectors

Truy cập Prometheus UI: `http://<server-ip>:9090`

**Query 1**: Select all time series cho một metric

```promql
up
```

**Expected**: Tất cả targets với up status (1 = up, 0 = down)

**Query 2**: Filter by label

```promql
up{job="node_exporter"}
```

**Expected**: Chỉ node_exporter target

**Query 3**: Multiple label matchers

```promql
app_requests_total{endpoint="/work", status="200"}
```

**Expected**: Requests to /work endpoint với status 200

**Query 4**: Regex matching

```promql
app_requests_total{endpoint=~"/work|/process"}
```

**Expected**: Requests to /work hoặc /process

**Query 5**: Negative matching

```promql
app_requests_total{status!="500"}
```

**Expected**: All requests except errors

### Exercise 1.2: Range Vector Selectors

**Query 1**: Last 5 minutes of data

```promql
app_requests_total[5m]
```

**Expected**: Range vector showing data points over 5 minutes

**Query 2**: Last 1 hour

```promql
node_cpu_seconds_total[1h]
```

**Expected**: CPU metrics for last hour

## Phần 2: Operators

### Exercise 2.1: Arithmetic Operators

**Query 1**: Calculate percentage

```promql
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
```

**Expected**: Available memory as percentage

**Query 2**: Subtract values

```promql
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

**Expected**: Used memory in bytes

**Query 3**: Add constant

```promql
app_active_users + 10
```

**Expected**: Active users plus 10

### Exercise 2.2: Comparison Operators

**Query 1**: Filter by value

```promql
app_active_users > 100
```

**Expected**: Only when active users > 100

**Query 2**: Memory usage threshold

```promql
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.8
```

**Expected**: Only when memory usage > 80%

### Exercise 2.3: Logical Operators

**Query 1**: AND operator

```promql
app_requests_total{endpoint="/work"} and app_requests_total > 10
```

**Expected**: /work requests where count > 10

**Query 2**: OR operator

```promql
app_requests_total{status="500"} or app_requests_total{status="404"}
```

**Expected**: Error responses (500 or 404)

## Phần 3: Functions

### Exercise 3.1: Rate and Increase

**Query 1**: Requests per second

```promql
rate(app_requests_total[5m])
```

**Expected**: Request rate per second over 5 minutes

**Query 2**: Total increase

```promql
increase(app_requests_total[1h])
```

**Expected**: Total requests in last hour

**Query 3**: CPU usage percentage

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Expected**: CPU usage percentage per instance

### Exercise 3.2: Aggregation Functions

**Query 1**: Sum across all endpoints

```promql
sum(app_requests_total)
```

**Expected**: Total requests across all endpoints

**Query 2**: Average active users

```promql
avg(app_active_users)
```

**Expected**: Average active users

**Query 3**: Max queue size

```promql
max(app_queue_size)
```

**Expected**: Maximum queue size observed

**Query 4**: Min, Max, Avg

```promql
min(app_active_users)
max(app_active_users)
avg(app_active_users)
```

**Expected**: Min, max, and average values

### Exercise 3.3: Time Functions

**Query 1**: Time since last scrape

```promql
time() - timestamp(app_requests_total)
```

**Expected**: Seconds since last update

**Query 2**: Day of week

```promql
day_of_week()
```

**Expected**: Current day (0 = Sunday, 6 = Saturday)

## Phần 4: Aggregations with Grouping

### Exercise 4.1: Group By

**Query 1**: Sum by endpoint

```promql
sum by (endpoint) (app_requests_total)
```

**Expected**: Total requests grouped by endpoint

**Query 2**: Rate by endpoint and status

```promql
sum by (endpoint, status) (rate(app_requests_total[5m]))
```

**Expected**: Request rate grouped by endpoint and status

**Query 3**: CPU by mode

```promql
sum by (mode) (rate(node_cpu_seconds_total[5m]))
```

**Expected**: CPU time grouped by mode (idle, system, user, etc.)

### Exercise 4.2: Without Clause

**Query 1**: Sum without instance

```promql
sum without (instance) (app_requests_total)
```

**Expected**: Aggregated across instances, keeping other labels

### Exercise 4.3: TopK and BottomK

**Query 1**: Top 3 endpoints by requests

```promql
topk(3, sum by (endpoint) (app_requests_total))
```

**Expected**: Top 3 endpoints with most requests

**Query 2**: Bottom 2 endpoints

```promql
bottomk(2, sum by (endpoint) (rate(app_requests_total[5m])))
```

**Expected**: 2 endpoints with lowest request rate

### Exercise 4.4: Count

**Query 1**: Count targets

```promql
count(up)
```

**Expected**: Number of targets

**Query 2**: Count by job

```promql
count by (job) (up)
```

**Expected**: Number of targets per job

## Phần 5: Histogram Functions

### Exercise 5.1: Histogram Quantile

**Query 1**: 95th percentile latency

```promql
histogram_quantile(0.95, rate(app_request_duration_seconds_bucket[5m]))
```

**Expected**: 95th percentile request latency

**Query 2**: 50th percentile (median)

```promql
histogram_quantile(0.5, rate(app_request_duration_seconds_bucket[5m]))
```

**Expected**: Median latency

**Query 3**: 99th percentile by endpoint

```promql
histogram_quantile(0.99, sum by (endpoint, le) (rate(app_request_duration_seconds_bucket[5m])))
```

**Expected**: 99th percentile latency per endpoint

### Exercise 5.2: Average from Histogram

**Query 1**: Average latency

```promql
rate(app_request_duration_seconds_sum[5m]) / rate(app_request_duration_seconds_count[5m])
```

**Expected**: Average request duration

## Phần 6: Advanced Queries

### Exercise 6.1: Subqueries

**Query 1**: Max rate over time

```promql
max_over_time(rate(app_requests_total[5m])[1h:])
```

**Expected**: Maximum request rate in last hour

**Query 2**: Average of averages

```promql
avg_over_time(avg(app_active_users)[10m:1m])
```

**Expected**: Average of 1-minute averages over 10 minutes

### Exercise 6.2: Offset

**Query 1**: Compare with 1 hour ago

```promql
app_requests_total - app_requests_total offset 1h
```

**Expected**: Increase in requests compared to 1 hour ago

**Query 2**: Week-over-week comparison

```promql
rate(app_requests_total[5m]) / rate(app_requests_total[5m] offset 1w)
```

**Expected**: Request rate ratio compared to last week

### Exercise 6.3: Predict Linear

**Query 1**: Predict disk full time

```promql
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600)
```

**Expected**: Predicted available bytes in 4 hours

### Exercise 6.4: Absent Function

**Query 1**: Alert if metric missing

```promql
absent(up{job="custom_app"})
```

**Expected**: 1 if metric absent, empty if present

### Exercise 6.5: Changes and Resets

**Query 1**: Count resets

```promql
resets(app_requests_total[1h])
```

**Expected**: Number of counter resets in last hour

**Query 2**: Count changes

```promql
changes(app_active_users[10m])
```

**Expected**: Number of times value changed

## Phần 7: Practical Use Cases

### Use Case 1: Error Rate

Calculate error rate percentage:

```promql
sum(rate(app_requests_total{status="500"}[5m])) / sum(rate(app_requests_total[5m])) * 100
```

**Expected**: Error rate as percentage

### Use Case 2: Latency SLO

Check if 95th percentile latency is under 500ms:

```promql
histogram_quantile(0.95, rate(app_request_duration_seconds_bucket[5m])) < 0.5
```

**Expected**: 1 if SLO met, 0 if violated

### Use Case 3: Memory Pressure

Identify high memory usage:

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
```

**Expected**: 1 when memory usage > 85%

### Use Case 4: Request Distribution

See request distribution across endpoints:

```promql
sum by (endpoint) (rate(app_requests_total[5m])) / ignoring(endpoint) group_left sum(rate(app_requests_total[5m])) * 100
```

**Expected**: Percentage of requests per endpoint

### Use Case 5: Throughput Trend

Compare current vs previous hour throughput:

```promql
(sum(rate(app_requests_total[5m])) - sum(rate(app_requests_total[5m] offset 1h))) / sum(rate(app_requests_total[5m] offset 1h)) * 100
```

**Expected**: Percentage change in throughput

### Use Case 6: Disk Space Remaining

Calculate days until disk full:

```promql
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 24 * 3600) / 1024 / 1024 / 1024
```

**Expected**: Predicted available GB in 24 hours

### Use Case 7: Top Consumers

Find top CPU consuming processes:

```promql
topk(5, rate(node_cpu_seconds_total{mode!="idle"}[5m]))
```

**Expected**: Top 5 CPU consumers

## Phần 8: Recording Rules (Optional)

Create recording rules để pre-compute expensive queries.

### Bước 1: Create Rules File

```bash
sudo vim /etc/prometheus/rules/recording_rules.yml
```

Nội dung:

```yaml
groups:
  - name: lab_recording_rules
    interval: 30s
    rules:
      # Pre-compute request rate
      - record: job:app_requests:rate5m
        expr: sum by (job, endpoint) (rate(app_requests_total[5m]))
      
      # Pre-compute error rate
      - record: job:app_error_rate:rate5m
        expr: sum by (job) (rate(app_requests_total{status="500"}[5m])) / sum by (job) (rate(app_requests_total[5m]))
      
      # Pre-compute CPU usage
      - record: instance:cpu_usage:percent
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
      
      # Pre-compute memory usage
      - record: instance:memory_usage:percent
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

### Bước 2: Reload Prometheus

```bash
# Validate rules
sudo /usr/local/bin/promtool check rules /etc/prometheus/rules/recording_rules.yml

# Reload Prometheus
curl -X POST http://localhost:9090/-/reload
```

### Bước 3: Query Recording Rules

```promql
# Use pre-computed metrics
job:app_requests:rate5m
instance:cpu_usage:percent
```

**Expected**: Faster query execution

## Troubleshooting

### Issue 1: Query Returns Empty

**Giải pháp**:
```promql
# Check if metric exists
{__name__=~"app_.*"}

# Check time range
app_requests_total[1h]

# Verify data is being scraped
up{job="custom_app"}
```

### Issue 2: Query Too Slow

**Giải pháp**:
- Reduce time range: `[5m]` instead of `[1h]`
- Use recording rules for complex queries
- Add more specific label filters
- Check query stats in Prometheus UI

### Issue 3: Unexpected Results

**Giải pháp**:
```promql
# Debug step by step
# 1. Check raw metric
app_requests_total

# 2. Check rate
rate(app_requests_total[5m])

# 3. Check aggregation
sum(rate(app_requests_total[5m]))
```

## Best Practices

1. **Use rate() for counters**: Always use `rate()` or `increase()` with counters
2. **Choose appropriate time range**: Match range to scrape interval (at least 4x)
3. **Use recording rules**: Pre-compute expensive queries
4. **Label efficiently**: Use `by` and `without` to control cardinality
5. **Test incrementally**: Build complex queries step by step

## Cleanup

Stop traffic generator:

```bash
kill $TRAFFIC_PID
```

## Tiếp Theo

Sau khi hoàn thành lab này, bạn đã có:
- ✅ Mastered basic PromQL syntax
- ✅ Understand operators và functions
- ✅ Can write aggregations với grouping
- ✅ Know how to use histogram functions
- ✅ Can write advanced queries với subqueries và offset

Tiếp tục với: [Verification](./verification.md) để kiểm tra kết quả, sau đó chuyển sang [Lab 04 - Alertmanager](../lab-04-alertmanager/README.md)
