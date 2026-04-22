# Lab 03 - PromQL Queries Verification

## Mục Đích

Document này giúp bạn verify rằng bạn đã nắm vững PromQL queries thông qua các exercises.

## Verification Checklist

### ✅ 1. Basic Selectors

Execute và verify results:

```promql
up
```

**Success Criteria**:
- Returns multiple time series
- Values are 0 or 1
- Has labels: job, instance

```promql
up{job="node_exporter"}
```

**Success Criteria**:
- Returns only node_exporter target
- Filtered correctly by label

### ✅ 2. Range Vectors

```promql
app_requests_total[5m]
```

**Success Criteria**:
- Returns range vector (not instant vector)
- Shows multiple data points
- Cannot be graphed directly (need function like rate())

### ✅ 3. Rate Function

```promql
rate(app_requests_total[5m])
```

**Success Criteria**:
- Returns per-second rate
- Values are decimals (e.g., 0.5 = 0.5 requests/sec)
- Can be graphed

### ✅ 4. Arithmetic Operations

```promql
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
```

**Success Criteria**:
- Returns percentage (0-100)
- Reasonable value (e.g., 45.5%)
- Matches system memory: `free -h`

### ✅ 5. Aggregations

```promql
sum by (endpoint) (app_requests_total)
```

**Success Criteria**:
- Multiple time series (one per endpoint)
- Labels show only: endpoint
- Values are totals per endpoint

### ✅ 6. Histogram Quantile

```promql
histogram_quantile(0.95, rate(app_request_duration_seconds_bucket[5m]))
```

**Success Criteria**:
- Returns single value (95th percentile)
- Value in seconds (e.g., 0.45)
- Reasonable latency value

### ✅ 7. TopK

```promql
topk(3, sum by (endpoint) (app_requests_total))
```

**Success Criteria**:
- Returns exactly 3 time series
- Sorted by value (highest first)
- Shows top 3 endpoints

### ✅ 8. Comparison Operators

```promql
app_active_users > 100
```

**Success Criteria**:
- Returns data only when condition true
- Empty result when all values <= 100
- Can be used for alerting

### ✅ 9. Offset

```promql
app_requests_total - app_requests_total offset 1h
```

**Success Criteria**:
- Returns difference from 1 hour ago
- Positive value (counter increased)
- Shows growth over time

### ✅ 10. Recording Rules (if implemented)

```promql
job:app_requests:rate5m
```

**Success Criteria**:
- Returns pre-computed values
- Faster than computing rate() live
- Same result as: `sum by (job, endpoint) (rate(app_requests_total[5m]))`

## Practical Verification Tests

### Test 1: Error Rate Calculation

Execute:
```promql
sum(rate(app_requests_total{status="500"}[5m])) / sum(rate(app_requests_total[5m])) * 100
```

**Verify**:
1. Generate errors: `for i in {1..10}; do curl http://localhost:8080/error; done`
2. Wait 30 seconds
3. Query should show error rate > 0
4. Value should be reasonable (e.g., 5-20%)

**Success**: Error rate reflects actual errors

### Test 2: CPU Usage

Execute:
```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Verify**:
1. Compare with system: `top` or `htop`
2. Values should be similar
3. Reasonable range (0-100%)

**Success**: CPU usage matches system monitoring

### Test 3: Memory Usage

Execute:
```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**Verify**:
1. Compare with: `free -h`
2. Calculate manually: (Total - Available) / Total * 100
3. Values should match

**Success**: Memory calculation is accurate

### Test 4: Latency Percentiles

Execute all:
```promql
histogram_quantile(0.5, rate(app_request_duration_seconds_bucket[5m]))
histogram_quantile(0.95, rate(app_request_duration_seconds_bucket[5m]))
histogram_quantile(0.99, rate(app_request_duration_seconds_bucket[5m]))
```

**Verify**:
1. p50 < p95 < p99 (increasing order)
2. All values reasonable (0.01 - 0.5 seconds based on app)
3. p99 should be higher than p50

**Success**: Percentiles in correct order and reasonable range

### Test 5: Request Distribution

Execute:
```promql
sum by (endpoint) (rate(app_requests_total[5m]))
```

**Verify**:
1. All endpoints present (/, /work, /process, /error)
2. Values sum to total request rate
3. Distribution makes sense based on traffic

**Success**: All endpoints accounted for

### Test 6: Time-based Comparison

Execute:
```promql
rate(app_requests_total[5m]) / rate(app_requests_total[5m] offset 1h)
```

**Verify**:
1. Returns ratio (e.g., 1.5 = 50% increase)
2. Value > 0
3. Reflects traffic changes

**Success**: Comparison shows traffic trend

## Advanced Verification

### Test 1: Subquery

```promql
max_over_time(rate(app_requests_total[5m])[1h:])
```

**Verify**:
- Returns maximum rate in last hour
- Value >= current rate
- Makes sense based on traffic pattern

### Test 2: Predict Linear

```promql
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 3600)
```

**Verify**:
- Returns predicted value in 1 hour
- Value < current available bytes (assuming disk usage)
- Reasonable prediction

### Test 3: Absent Function

```promql
absent(up{job="nonexistent"})
```

**Verify**:
- Returns 1 (metric is absent)
- Can be used for alerting on missing metrics

```promql
absent(up{job="prometheus"})
```

**Verify**:
- Returns empty (metric exists)

### Test 4: Changes Function

```promql
changes(app_active_users[10m])
```

**Verify**:
- Returns number of value changes
- Value > 0 (gauge changes frequently)
- Reflects metric volatility

## Query Performance Verification

### Test 1: Query Execution Time

In Prometheus UI, check query stats:

**Good Performance**:
- Execution time < 1 second
- Reasonable number of samples
- No timeout errors

**Poor Performance**:
- Execution time > 5 seconds
- Too many samples (millions)
- Consider using recording rules

### Test 2: Cardinality Check

```promql
count({__name__=~"app_.*"})
```

**Verify**:
- Reasonable number (< 1000 for lab)
- Not growing exponentially
- Manageable cardinality

## Common Query Patterns Verification

### Pattern 1: Availability

```promql
avg(up) * 100
```

**Expected**: Overall availability percentage (should be ~100%)

### Pattern 2: Request Success Rate

```promql
sum(rate(app_requests_total{status!="500"}[5m])) / sum(rate(app_requests_total[5m])) * 100
```

**Expected**: Success rate percentage (should be > 90%)

### Pattern 3: Resource Utilization

```promql
# CPU
instance:cpu_usage:percent

# Memory
instance:memory_usage:percent

# Disk
(node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100
```

**Expected**: All resources within normal ranges

### Pattern 4: Throughput

```promql
sum(rate(app_requests_total[5m]))
```

**Expected**: Total requests per second across all endpoints

## Troubleshooting Verification

### Issue: Query Returns Empty

**Test**:
```promql
# Check metric exists
{__name__=~"app_requests.*"}

# Check time range has data
app_requests_total[1h]
```

**Success**: Can identify why query is empty

### Issue: Unexpected Values

**Test**:
```promql
# Break down query step by step
app_requests_total
rate(app_requests_total[5m])
sum(rate(app_requests_total[5m]))
```

**Success**: Can debug query by examining each step

### Issue: Query Too Slow

**Test**:
```promql
# Check query stats in UI
# Reduce time range
rate(app_requests_total[1m])  # instead of [1h]
```

**Success**: Can optimize slow queries

## Documentation Verification

Create a query cheat sheet with your most useful queries:

```promql
# Error Rate
sum(rate(app_requests_total{status="500"}[5m])) / sum(rate(app_requests_total[5m]))

# Latency p95
histogram_quantile(0.95, rate(app_request_duration_seconds_bucket[5m]))

# CPU Usage
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory Usage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Top Endpoints
topk(5, sum by (endpoint) (rate(app_requests_total[5m])))
```

**Success**: Have documented useful queries for future reference

## Success Criteria

Lab này được coi là hoàn thành khi:

- ✅ Can write basic instant and range vector selectors
- ✅ Understand và use rate(), increase() correctly
- ✅ Can perform arithmetic operations on metrics
- ✅ Can aggregate với sum, avg, max, min
- ✅ Can use grouping với by và without
- ✅ Can calculate histogram quantiles
- ✅ Can use topk, bottomk, count
- ✅ Can write comparison và logical operators
- ✅ Can use offset for time-based comparisons
- ✅ Can write practical queries for real use cases
- ✅ Understand query performance implications
- ✅ Can troubleshoot query issues

## Self-Assessment Questions

Answer these to verify understanding:

1. **What's the difference between instant and range vectors?**
   - Instant: Single value at a point in time
   - Range: Multiple values over a time range

2. **Why use rate() instead of raw counter values?**
   - Counters always increase, rate() shows per-second change
   - Handles counter resets automatically

3. **What's the difference between sum by (label) and sum without (label)?**
   - `by`: Keep only specified labels
   - `without`: Remove specified labels, keep others

4. **When to use histogram_quantile()?**
   - To calculate percentiles (p50, p95, p99) from histogram metrics

5. **What does offset do?**
   - Shifts query back in time for comparisons

6. **Why are recording rules useful?**
   - Pre-compute expensive queries
   - Faster query execution
   - Reduce load on Prometheus

## Tiếp Theo

Nếu tất cả verifications pass:

1. ✅ Mark Lab 03 as complete
2. 📝 Save your useful queries for reference
3. 🎯 Proceed to [Lab 04 - Alertmanager](../lab-04-alertmanager/README.md)

Nếu có issues:
- Review PromQL documentation
- Practice more with different queries
- Experiment with query variations
- Check Prometheus UI query stats
