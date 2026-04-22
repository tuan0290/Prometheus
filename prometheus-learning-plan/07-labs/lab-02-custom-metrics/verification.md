# Lab 02 - Custom Metrics Verification

## Mục Đích

Document này giúp bạn verify rằng custom metrics application đã được implement và integrate với Prometheus đúng cách.

## Checklist Verification

### ✅ 1. Application Running

Kiểm tra application đang chạy:

```bash
# Check process
ps aux | grep app.py

# Or if running as service
sudo systemctl status custom-metrics-app
```

**Expected**: Process running, no errors

### ✅ 2. Metrics Endpoint Accessible

Test metrics endpoint:

```bash
curl http://localhost:8080/metrics
```

**Expected Output** (sample):
```
# HELP app_requests_total Total number of requests
# TYPE app_requests_total counter
app_requests_total{endpoint="/",method="GET",status="200"} 5.0

# HELP app_active_users Number of active users
# TYPE app_active_users gauge
app_active_users 127.0

# HELP app_request_duration_seconds HTTP request latency in seconds
# TYPE app_request_duration_seconds histogram
app_request_duration_seconds_bucket{endpoint="/work",le="0.01"} 0.0
...
```

### ✅ 3. All Metric Types Present

Verify tất cả 4 metric types:

```bash
curl -s http://localhost:8080/metrics | grep -E "^# TYPE"
```

**Expected Output**:
```
# TYPE app_requests_total counter
# TYPE app_active_users gauge
# TYPE app_queue_size gauge
# TYPE app_request_duration_seconds histogram
# TYPE app_process_time_seconds summary
```

### ✅ 4. Prometheus Target UP

Trong Prometheus UI (`http://<server-ip>:9090`):

1. Navigate to **Status** → **Targets**
2. Find `custom_app` job

**Expected**:
- State: **UP** (green)
- Endpoint: `http://localhost:8080/metrics`
- Last Scrape: < 30 seconds ago
- No errors

### ✅ 5. Counter Metrics Working

#### Test 1: Generate Requests

```bash
# Generate traffic
for i in {1..10}; do
    curl -s http://localhost:8080/ > /dev/null
done
```

#### Test 2: Query Counter

Trong Prometheus UI, execute:

```promql
app_requests_total{endpoint="/"}
```

**Expected**:
- Value increases with each request
- Labels: `endpoint="/"`, `method="GET"`, `status="200"`
- Value >= 10

#### Test 3: Rate Calculation

```promql
rate(app_requests_total[1m])
```

**Expected**:
- Returns requests per second
- Value > 0 if traffic is active

### ✅ 6. Gauge Metrics Working

#### Test 1: Check Current Values

```promql
app_active_users
```

**Expected**:
- Returns current value (e.g., 145)
- Value changes over time (background thread updates every 5s)

#### Test 2: Check Queue Size

```promql
app_queue_size
```

**Expected**:
- Returns current queue size
- Value between 0 and 50
- Changes over time

#### Test 3: Gauge Over Time

Switch to **Graph** tab và observe:

**Expected**:
- Line graph shows values going up and down
- Not monotonically increasing (unlike counters)

### ✅ 7. Histogram Metrics Working

#### Test 1: Generate Latency Data

```bash
# Generate requests with varying latency
for i in {1..20}; do
    curl -s http://localhost:8080/work > /dev/null
    sleep 0.5
done
```

#### Test 2: Check Histogram Buckets

```promql
app_request_duration_seconds_bucket{endpoint="/work"}
```

**Expected**:
- Multiple time series (one per bucket)
- Labels include `le` (less than or equal)
- Buckets: 0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, +Inf
- Values are cumulative (increasing with le)

#### Test 3: Calculate Quantiles

```promql
histogram_quantile(0.95, rate(app_request_duration_seconds_bucket[5m]))
```

**Expected**:
- Returns 95th percentile latency
- Value between 0 and 0.5 (based on app logic)
- Reasonable value (e.g., 0.45 seconds)

#### Test 4: Average Latency

```promql
rate(app_request_duration_seconds_sum[5m]) / rate(app_request_duration_seconds_count[5m])
```

**Expected**:
- Returns average latency
- Value between 0.01 and 0.5 seconds

### ✅ 8. Summary Metrics Working

#### Test 1: Generate Processing Jobs

```bash
for i in {1..15}; do
    curl -s http://localhost:8080/process > /dev/null
    sleep 1
done
```

#### Test 2: Check Summary

```promql
app_process_time_seconds_sum
```

**Expected**:
- Returns total time spent processing
- Value increases with each request
- Counter-like behavior

#### Test 3: Average Process Time

```promql
rate(app_process_time_seconds_sum[5m]) / rate(app_process_time_seconds_count[5m])
```

**Expected**:
- Returns average process time
- Value between 0.1 and 1.0 seconds

### ✅ 9. Labels Working Correctly

#### Test 1: Query by Label

```promql
app_requests_total{endpoint="/work"}
```

**Expected**:
- Returns only /work endpoint metrics
- Other endpoints filtered out

#### Test 2: Aggregate by Label

```promql
sum by (endpoint) (app_requests_total)
```

**Expected**:
- Multiple time series (one per endpoint)
- Endpoints: `/`, `/work`, `/process`, `/error`
- Each shows total requests for that endpoint

#### Test 3: Error Rate

```promql
rate(app_requests_total{status="500"}[1m])
```

**Expected**:
- Returns error rate
- Value > 0 if /error endpoint was called

### ✅ 10. Metrics Metadata Correct

Check metrics have proper HELP and TYPE:

```bash
curl -s http://localhost:8080/metrics | grep -A1 "^# HELP app_requests_total"
```

**Expected Output**:
```
# HELP app_requests_total Total number of requests
# TYPE app_requests_total counter
```

Verify for all metrics:
- `app_requests_total` - counter
- `app_active_users` - gauge
- `app_queue_size` - gauge
- `app_request_duration_seconds` - histogram
- `app_process_time_seconds` - summary

## Advanced Verification

### Test 1: Cardinality Check

Check number of unique time series:

```promql
count(app_requests_total)
```

**Expected**:
- Reasonable number (< 100)
- Not thousands (indicates cardinality explosion)

### Test 2: Scrape Duration

Check how long Prometheus takes to scrape:

```promql
scrape_duration_seconds{job="custom_app"}
```

**Expected**:
- Value < 1 second
- Typically 0.01 - 0.1 seconds

### Test 3: Scrape Success

```promql
up{job="custom_app"}
```

**Expected**:
- Value = 1 (target is up)
- Consistent over time

### Test 4: Data Freshness

Check timestamp of last scrape:

```promql
timestamp(app_requests_total)
```

**Expected**:
- Recent timestamp (within last 30 seconds)
- Updates every scrape interval

### Test 5: Histogram Accuracy

Verify histogram buckets are cumulative:

```bash
curl -s http://localhost:8080/metrics | grep "app_request_duration_seconds_bucket"
```

**Expected**:
```
app_request_duration_seconds_bucket{endpoint="/work",le="0.01"} 2.0
app_request_duration_seconds_bucket{endpoint="/work",le="0.05"} 5.0
app_request_duration_seconds_bucket{endpoint="/work",le="0.1"} 8.0
...
```

Each bucket value >= previous bucket (cumulative).

## Integration Tests

### Test 1: End-to-End Flow

```bash
# 1. Generate request
curl http://localhost:8080/work

# 2. Wait for scrape
sleep 20

# 3. Query in Prometheus
curl -s 'http://localhost:9090/api/v1/query?query=app_requests_total{endpoint="/work"}' | jq .
```

**Expected**:
- API returns success
- Data includes recent request

### Test 2: Multiple Endpoints

```bash
# Generate traffic to all endpoints
curl http://localhost:8080/
curl http://localhost:8080/work
curl http://localhost:8080/process
curl http://localhost:8080/error

# Wait and query
sleep 20
```

Query:
```promql
sum by (endpoint, status) (app_requests_total)
```

**Expected**:
- 4 time series (one per endpoint)
- Different status codes (200, 500)

### Test 3: Continuous Monitoring

Run load generator:

```bash
# In background
while true; do
    curl -s http://localhost:8080/work > /dev/null
    sleep 2
done &

# Save PID to stop later
LOAD_PID=$!
```

Monitor in Prometheus:
```promql
rate(app_requests_total{endpoint="/work"}[1m])
```

**Expected**:
- Steady rate (~0.5 requests/sec)
- Graph shows consistent line

Stop load:
```bash
kill $LOAD_PID
```

## Common Issues & Solutions

### Issue: Metrics Not Appearing

**Symptoms**: Queries return empty results

**Diagnosis**:
```bash
# Check app is exposing metrics
curl http://localhost:8080/metrics | grep app_requests_total

# Check Prometheus is scraping
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.job=="custom_app")'
```

**Solution**:
- Verify app is running
- Check Prometheus config includes custom_app job
- Reload Prometheus config

### Issue: Histogram Quantiles Return NaN

**Symptoms**: `histogram_quantile()` returns NaN

**Diagnosis**:
```promql
# Check if histogram has data
app_request_duration_seconds_count
```

**Solution**:
- Generate more traffic (need multiple samples)
- Wait for more data (5m range needs 5 minutes of data)
- Check buckets are properly defined

### Issue: High Cardinality

**Symptoms**: Too many time series, Prometheus slow

**Diagnosis**:
```promql
# Count time series per metric
count by (__name__) ({__name__=~"app_.*"})
```

**Solution**:
- Remove high-cardinality labels (user IDs, timestamps)
- Use label values with limited set (status codes, not URLs)
- Aggregate before storing

### Issue: Gauge Not Updating

**Symptoms**: Gauge shows same value

**Diagnosis**:
```bash
# Check background thread is running
curl http://localhost:8080/metrics | grep app_active_users

# Wait 10 seconds and check again
sleep 10
curl http://localhost:8080/metrics | grep app_active_users
```

**Solution**:
- Verify background thread started
- Check for exceptions in app logs
- Restart application

## Performance Benchmarks

Expected performance:

| Metric | Expected Value |
|--------|----------------|
| App memory usage | 30-100 MB |
| Metrics endpoint response time | < 100ms |
| Number of time series | < 100 |
| Scrape duration | < 100ms |

## Success Criteria

Lab này được coi là hoàn thành khi:

- ✅ Application running và stable
- ✅ All 4 metric types implemented (Counter, Gauge, Histogram, Summary)
- ✅ Metrics endpoint accessible
- ✅ Prometheus scraping successfully
- ✅ All metrics queryable trong Prometheus UI
- ✅ Labels working correctly
- ✅ Histogram quantiles calculable
- ✅ No high cardinality issues

## Tiếp Theo

Nếu tất cả verifications pass:

1. ✅ Mark Lab 02 as complete
2. 📝 Experiment với different metric types
3. 🎯 Proceed to [Lab 03 - PromQL Queries](../lab-03-promql-queries/README.md)

Nếu có issues:
- Review [Troubleshooting section](./instructions.md#troubleshooting)
- Check application logs
- Verify Prometheus configuration
- Test metrics endpoint directly
