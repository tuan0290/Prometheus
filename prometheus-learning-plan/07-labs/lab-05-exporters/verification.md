# Lab 05 - Exporters Verification

## Mục Đích

Document này giúp bạn verify rằng blackbox_exporter và postgres_exporter đã được cài đặt, cấu hình, và hoạt động đúng.

## Checklist Verification

### ✅ 1. Exporters Running

```bash
sudo systemctl status blackbox_exporter
sudo systemctl status postgres_exporter
```

**Expected**: Both services active (running)

### ✅ 2. Ports Listening

```bash
sudo netstat -tlnp | grep -E '9115|9187'
```

**Expected**:
```
tcp6  0  0 :::9115  :::*  LISTEN  <pid>/blackbox_exporter
tcp6  0  0 :::9187  :::*  LISTEN  <pid>/postgres_exporter
```

### ✅ 3. Metrics Endpoints Accessible

```bash
# blackbox_exporter
curl http://localhost:9115/metrics

# postgres_exporter
curl http://localhost:9187/metrics
```

**Expected**: Both return Prometheus metrics

### ✅ 4. Prometheus Targets UP

Trong Prometheus UI (`http://<server-ip>:9090`):

Navigate to **Status** → **Targets**

**Expected**:
- `blackbox_http` job: All targets UP
- `blackbox_tcp` job: All targets UP
- `blackbox_exporter` job: UP
- `postgres` job: UP

### ✅ 5. blackbox_exporter Probes Working

#### Test HTTP Probe

```bash
curl 'http://localhost:9115/probe?target=https://prometheus.io&module=http_2xx'
```

**Expected**:
```
probe_success 1
probe_http_status_code 200
probe_http_duration_seconds <value>
```

#### Test TCP Probe

```bash
curl 'http://localhost:9115/probe?target=localhost:9090&module=tcp_connect'
```

**Expected**:
```
probe_success 1
probe_duration_seconds <value>
```

### ✅ 6. blackbox_exporter Metrics in Prometheus

Execute queries:

```promql
probe_success{job="blackbox_http"}
```

**Expected**:
- Multiple time series (one per target)
- Values: 1 (success) or 0 (failure)
- All should be 1 if targets are up

```promql
probe_http_duration_seconds{job="blackbox_http"}
```

**Expected**:
- Response times for each target
- Reasonable values (< 5 seconds typically)

```promql
probe_http_status_code{job="blackbox_http"}
```

**Expected**:
- HTTP status codes (200, 301, etc.)
- Should match expected codes for each target

### ✅ 7. PostgreSQL Connection Working

```bash
psql -U prometheus_monitor -h localhost -d postgres
# Password: prometheus123
```

**Expected**: Successfully connects to PostgreSQL

### ✅ 8. postgres_exporter Metrics

```bash
curl http://localhost:9187/metrics | grep -E "^pg_"
```

**Expected**: Many PostgreSQL metrics including:
- `pg_up`
- `pg_database_size_bytes`
- `pg_stat_activity_count`
- `pg_stat_database_xact_commit`

### ✅ 9. postgres_exporter Metrics in Prometheus

Execute queries:

```promql
pg_up
```

**Expected**: Value = 1 (PostgreSQL is up)

```promql
pg_database_size_bytes
```

**Expected**: Database sizes in bytes

```promql
pg_stat_activity_count
```

**Expected**: Current number of connections

```promql
pg_settings_max_connections
```

**Expected**: Max connections configured

### ✅ 10. Exporter Alerts Loaded

Trong Prometheus UI:

Navigate to **Status** → **Rules**

**Expected**:
- Group: `exporter_alerts`
- Rules: EndpointDown, SlowResponse, SSLCertExpiringSoon, PostgreSQLDown, PostgreSQLTooManyConnections, PostgreSQLDeadLocks

## Advanced Verification

### Test 1: HTTP Probe Failure Detection

```bash
# Probe non-existent endpoint
curl 'http://localhost:9115/probe?target=http://localhost:9999&module=http_2xx'
```

**Expected**:
```
probe_success 0
```

Verify in Prometheus:
```promql
probe_success{instance="http://localhost:9999"} == 0
```

### Test 2: SSL Certificate Expiry Check

```promql
(probe_ssl_earliest_cert_expiry{job="blackbox_http"} - time()) / 86400
```

**Expected**:
- Returns days until certificate expiry
- Positive values (certificates not expired)
- Reasonable values (e.g., 30-365 days)

### Test 3: Response Time Monitoring

```promql
histogram_quantile(0.95, rate(probe_http_duration_seconds_bucket[5m]))
```

**Expected**:
- 95th percentile response time
- Reasonable value (< 5 seconds)

### Test 4: PostgreSQL Connection Usage

```promql
pg_stat_activity_count / pg_settings_max_connections * 100
```

**Expected**:
- Connection usage percentage
- Should be < 80% under normal load

### Test 5: PostgreSQL Transaction Rate

```promql
rate(pg_stat_database_xact_commit[5m])
```

**Expected**:
- Transactions per second
- Value > 0 if database is active

### Test 6: PostgreSQL Cache Hit Ratio

```promql
sum(pg_stat_database_blks_hit) / (sum(pg_stat_database_blks_hit) + sum(pg_stat_database_blks_read)) * 100
```

**Expected**:
- Cache hit ratio percentage
- Should be > 90% for good performance

### Test 7: Multiple Probe Modules

Test different probe types:

```bash
# HTTP
curl 'http://localhost:9115/probe?target=https://prometheus.io&module=http_2xx'

# TCP
curl 'http://localhost:9115/probe?target=localhost:5432&module=tcp_connect'

# ICMP (may require root)
curl 'http://localhost:9115/probe?target=8.8.8.8&module=icmp'
```

**Expected**: All probes return appropriate metrics

### Test 8: Endpoint Availability Over Time

```promql
avg_over_time(probe_success{job="blackbox_http"}[24h]) * 100
```

**Expected**:
- Uptime percentage over 24 hours
- Should be close to 100% for stable endpoints

## Integration Tests

### Test 1: End-to-End HTTP Monitoring

1. Add new target to Prometheus config
2. Reload Prometheus
3. Verify target appears in UI
4. Check probe_success metric
5. Verify response time metrics

**Expected**: Complete flow works

### Test 2: Database Monitoring Flow

1. Create test database activity:
```bash
sudo -u postgres psql -c "SELECT pg_sleep(1);"
```

2. Query metrics:
```promql
pg_stat_activity_count
```

**Expected**: Metrics reflect database activity

### Test 3: Alert Triggering

#### Trigger EndpointDown Alert

```bash
# Stop custom app
sudo systemctl stop custom-metrics-app

# Wait 2-3 minutes
# Check alert fires
```

**Expected**: EndpointDown alert fires for localhost:8080

#### Trigger PostgreSQLDown Alert

```bash
# Stop PostgreSQL
sudo systemctl stop postgresql

# Wait 1-2 minutes
```

**Expected**: PostgreSQLDown alert fires

**Restore**:
```bash
sudo systemctl start postgresql
sudo systemctl start custom-metrics-app
```

## Practical Use Cases Verification

### Use Case 1: Website Monitoring

Monitor external websites:

```promql
# Check all websites are up
probe_success{job="blackbox_http"}

# Find slow websites
probe_http_duration_seconds{job="blackbox_http"} > 2

# Check SSL certificates expiring soon
(probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
```

**Expected**: Queries return relevant data

### Use Case 2: Service Health Check

Monitor internal services:

```promql
# Check all services responding
probe_success{job="blackbox_tcp"}

# Find connection issues
probe_success{job="blackbox_tcp"} == 0
```

**Expected**: All internal services show as up

### Use Case 3: Database Performance

Monitor database health:

```promql
# Connection pool usage
pg_stat_activity_count / pg_settings_max_connections

# Transaction throughput
sum(rate(pg_stat_database_xact_commit[5m]))

# Lock contention
pg_locks_count

# Database size growth
rate(pg_database_size_bytes[1h])
```

**Expected**: All metrics show healthy database

### Use Case 4: Capacity Planning

Predict resource needs:

```promql
# Predict when connections will be exhausted
predict_linear(pg_stat_activity_count[1h], 4 * 3600)

# Database growth rate
rate(pg_database_size_bytes[24h])
```

**Expected**: Predictions help with capacity planning

## Performance Benchmarks

Expected performance:

| Metric | Expected Value |
|--------|----------------|
| blackbox_exporter memory | 10-30 MB |
| postgres_exporter memory | 20-50 MB |
| Probe duration (HTTP) | < 5s |
| Probe duration (TCP) | < 1s |
| Scrape duration | < 1s |

## Common Issues & Solutions

### Issue: Probe Always Fails

**Diagnosis**:
```bash
# Test manually
curl -I <target-url>

# Check blackbox logs
sudo journalctl -u blackbox_exporter -f

# Test probe directly
curl 'http://localhost:9115/probe?target=<url>&module=http_2xx'
```

**Solution**: Fix target URL, adjust timeout, check network

### Issue: postgres_exporter Shows No Metrics

**Diagnosis**:
```bash
# Test connection
psql -U prometheus_monitor -h localhost -d postgres

# Check logs
sudo journalctl -u postgres_exporter -f

# Verify DATA_SOURCE_NAME
sudo systemctl cat postgres_exporter
```

**Solution**: Fix connection string, check PostgreSQL auth

### Issue: SSL Certificate Metrics Missing

**Diagnosis**:
```bash
# Check if target uses HTTPS
curl -I <target-url>

# Verify probe module
curl 'http://localhost:9115/probe?target=<url>&module=http_2xx'
```

**Solution**: Ensure target uses HTTPS, check probe config

### Issue: High Probe Failure Rate

**Diagnosis**:
```promql
# Check failure rate
rate(probe_success{job="blackbox_http"}[5m])

# Identify failing targets
probe_success == 0
```

**Solution**: Increase timeout, check target stability, adjust probe frequency

## Success Criteria

Lab này được coi là hoàn thành khi:

- ✅ blackbox_exporter service running
- ✅ postgres_exporter service running
- ✅ All Prometheus targets UP
- ✅ HTTP probes working correctly
- ✅ TCP probes working correctly
- ✅ PostgreSQL metrics being collected
- ✅ Can query all exporter metrics
- ✅ Alerts configured và working
- ✅ SSL certificate monitoring working
- ✅ No errors in exporter logs

## Documentation Checklist

Document your exporter setup:

- [ ] List of monitored endpoints
- [ ] Probe configurations
- [ ] Expected response times
- [ ] Alert thresholds
- [ ] Database monitoring queries
- [ ] Troubleshooting procedures

## Tiếp Theo

Nếu tất cả verifications pass:

1. ✅ Mark Lab 05 as complete
2. 📝 Document monitored endpoints và thresholds
3. 🎯 Proceed to [Lab 06 - Service Discovery](../lab-06-service-discovery/README.md)

Nếu có issues:
- Review [Troubleshooting section](./instructions.md#troubleshooting)
- Check exporter logs
- Verify configurations
- Test probes manually
