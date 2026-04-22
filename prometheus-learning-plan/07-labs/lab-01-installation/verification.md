# Lab 01 - Installation Verification

## Mục Đích

Document này giúp bạn verify rằng Prometheus và node_exporter đã được cài đặt và cấu hình đúng.

## Checklist Verification

### ✅ 1. Services Running

Kiểm tra cả 2 services đang chạy:

```bash
sudo systemctl status prometheus
sudo systemctl status node_exporter
```

**Expected Output**:
- Status: `active (running)`
- Không có error messages trong logs

### ✅ 2. Ports Listening

Verify ports đang listen:

```bash
sudo netstat -tlnp | grep -E '9090|9100'
```

**Expected Output**:
```
tcp6  0  0 :::9090  :::*  LISTEN  <pid>/prometheus
tcp6  0  0 :::9100  :::*  LISTEN  <pid>/node_exporter
```

### ✅ 3. Prometheus Web UI Accessible

Truy cập trong browser:

```
http://<your-server-ip>:9090
```

**Expected**:
- Prometheus web interface loads
- Không có error messages
- Graph tab có thể access được

### ✅ 4. Targets Status

Trong Prometheus UI:
1. Navigate to **Status** → **Targets**
2. Verify cả 2 targets:

**Expected**:

| Endpoint | State | Labels |
|----------|-------|--------|
| http://localhost:9090/metrics | UP | job="prometheus" |
| http://localhost:9100/metrics | UP | job="node_exporter" |

- State phải là **UP** (màu xanh)
- Last Scrape phải là recent (< 30 seconds ago)
- Không có error messages

### ✅ 5. Metrics Endpoint Accessible

Test metrics endpoints:

```bash
# Prometheus metrics
curl -s http://localhost:9090/metrics | head -20

# node_exporter metrics
curl -s http://localhost:9100/metrics | head -20
```

**Expected**:
- Trả về metrics trong Prometheus format
- Không có error messages
- Có nhiều metrics lines

### ✅ 6. Query Prometheus Metrics

Trong Prometheus UI, execute query:

```promql
up
```

**Expected Output**:
```
up{instance="localhost:9090", job="prometheus"}  1
up{instance="localhost:9100", job="node_exporter"}  1
```

- Value = 1 (indicates target is up)
- Cả 2 instances hiển thị

### ✅ 7. Query node_exporter Metrics

Execute các queries sau:

#### Query 1: CPU Metrics
```promql
node_cpu_seconds_total
```

**Expected**:
- Trả về multiple time series
- Có các labels: `cpu`, `mode` (idle, system, user, etc.)
- Values là counters (increasing over time)

#### Query 2: Memory Metrics
```promql
node_memory_MemTotal_bytes
```

**Expected**:
- Trả về 1 time series
- Value là total memory in bytes
- Matches system memory: `free -b | grep Mem`

#### Query 3: Disk Metrics
```promql
node_filesystem_size_bytes
```

**Expected**:
- Trả về multiple time series (1 per filesystem)
- Labels: `device`, `fstype`, `mountpoint`
- Values match `df -B1`

### ✅ 8. Configuration Valid

Validate configuration:

```bash
/usr/local/bin/promtool check config /etc/prometheus/prometheus.yml
```

**Expected Output**:
```
Checking /etc/prometheus/prometheus.yml
  SUCCESS: 2 rule files found
  SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax
```

### ✅ 9. Data Directory

Verify Prometheus đang write data:

```bash
ls -lh /var/lib/prometheus/
```

**Expected**:
- Directory exists
- Contains subdirectories: `chunks_head`, `wal`
- Owner: `prometheus:prometheus`
- Files được update recently

### ✅ 10. Logs Clean

Check logs không có errors:

```bash
# Prometheus logs (last 50 lines)
sudo journalctl -u prometheus -n 50 --no-pager

# node_exporter logs (last 50 lines)
sudo journalctl -u node_exporter -n 50 --no-pager
```

**Expected**:
- Không có ERROR hoặc FATAL messages
- Có thể có INFO hoặc WARN (acceptable)
- Scrape logs showing successful scrapes

## Advanced Verification

### Test 1: Scrape Interval

Verify scrape interval là 15s:

1. Trong Prometheus UI, query: `up`
2. Switch to **Graph** tab
3. Observe data points
4. Zoom in to see individual scrapes

**Expected**: Data points cách nhau ~15 seconds

### Test 2: Query Performance

Execute complex query:

```promql
rate(node_cpu_seconds_total{mode="idle"}[5m])
```

**Expected**:
- Query completes trong < 1 second
- Trả về time series cho mỗi CPU core
- Values between 0 and 1

### Test 3: Configuration Reload

Test hot reload:

```bash
# Backup config
sudo cp /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.bak

# Add comment to config
echo "# Test comment" | sudo tee -a /etc/prometheus/prometheus.yml

# Reload
curl -X POST http://localhost:9090/-/reload

# Check logs
sudo journalctl -u prometheus -n 10
```

**Expected**:
- Reload successful
- Prometheus không restart
- Logs show "Loading configuration file"

### Test 4: API Endpoints

Test Prometheus API:

```bash
# Query API
curl -s 'http://localhost:9090/api/v1/query?query=up' | jq .

# Targets API
curl -s 'http://localhost:9090/api/v1/targets' | jq .

# Config API
curl -s 'http://localhost:9090/api/v1/status/config' | jq .
```

**Expected**:
- All APIs return JSON
- Status: "success"
- Data contains expected information

## Common Issues & Solutions

### Issue: Target DOWN

**Symptoms**: Target shows state "DOWN" in UI

**Diagnosis**:
```bash
# Check service
sudo systemctl status node_exporter

# Check connectivity
curl http://localhost:9100/metrics

# Check Prometheus logs
sudo journalctl -u prometheus | grep -i error
```

**Solution**: Restart service hoặc fix network/firewall issues

### Issue: No Data in Queries

**Symptoms**: Queries return empty results

**Diagnosis**:
```bash
# Check scrape config
cat /etc/prometheus/prometheus.yml

# Check targets status
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job, health, lastError}'
```

**Solution**: Wait for first scrape (15s) hoặc fix target configuration

### Issue: High Memory Usage

**Symptoms**: Prometheus using too much memory

**Diagnosis**:
```bash
# Check memory usage
ps aux | grep prometheus

# Check TSDB stats
curl http://localhost:9090/api/v1/status/tsdb | jq .
```

**Solution**: Adjust retention hoặc reduce scrape frequency

## Performance Benchmarks

Sau khi cài đặt, bạn nên thấy:

| Metric | Expected Value |
|--------|----------------|
| Prometheus memory usage | 50-200 MB |
| node_exporter memory usage | 10-30 MB |
| Scrape duration | < 100ms |
| Query response time | < 1s |
| Disk usage growth | ~1-5 MB/day |

## Success Criteria

Lab này được coi là hoàn thành khi:

- ✅ Prometheus service running và stable
- ✅ node_exporter service running và stable
- ✅ Cả 2 targets UP trong Prometheus UI
- ✅ Có thể query metrics successfully
- ✅ Configuration valid và reload works
- ✅ Không có errors trong logs
- ✅ Web UI accessible và functional

## Tiếp Theo

Nếu tất cả verifications pass:

1. ✅ Mark Lab 01 as complete
2. 📝 Document any issues encountered và solutions
3. 🎯 Proceed to [Lab 02 - Custom Metrics](../lab-02-custom-metrics/README.md)

Nếu có issues:
- Review [Troubleshooting section](./instructions.md#troubleshooting) trong instructions
- Check logs carefully
- Verify all steps were followed
- Ask for help in community forums nếu cần
