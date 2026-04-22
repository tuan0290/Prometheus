# Lab 04 - Alertmanager Verification

## Mục Đích

Document này giúp bạn verify rằng Alertmanager đã được cài đặt, cấu hình, và hoạt động đúng với alerting rules và notifications.

## Checklist Verification

### ✅ 1. Alertmanager Service Running

```bash
sudo systemctl status alertmanager
```

**Expected**:
- Status: `active (running)`
- No errors in logs

### ✅ 2. Alertmanager UI Accessible

Truy cập: `http://<server-ip>:9093`

**Expected**:
- Alertmanager web interface loads
- Shows "Alerts" and "Silences" tabs
- No connection errors

### ✅ 3. Alertmanager Port Listening

```bash
sudo netstat -tlnp | grep 9093
```

**Expected**:
```
tcp6  0  0 :::9093  :::*  LISTEN  <pid>/alertmanager
```

### ✅ 4. Alert Rules Loaded in Prometheus

Trong Prometheus UI (`http://<server-ip>:9090`):

1. Navigate to **Status** → **Rules**
2. Verify alert rules are loaded

**Expected**:
- Group: `lab_alerts`
- Rules visible: InstanceDown, HighCPUUsage, HighMemoryUsage, HighErrorRate, HighLatency, DiskSpaceLow
- State shows for each rule

### ✅ 5. Prometheus Connected to Alertmanager

Trong Prometheus UI:

1. Navigate to **Status** → **Runtime & Build Information**
2. Check Alertmanagers section

**Expected**:
- Shows `http://localhost:9093/api/v2/alerts`
- Status: UP

Hoặc query:

```promql
alertmanager_build_info
```

**Expected**: Returns metrics from Alertmanager

### ✅ 6. Configuration Valid

```bash
# Validate Alertmanager config
sudo /usr/local/bin/amtool check-config /etc/alertmanager/alertmanager.yml

# Validate Prometheus rules
sudo /usr/local/bin/promtool check rules /etc/prometheus/rules/alerts.yml
```

**Expected**: Both show SUCCESS

### ✅ 7. Test Alert Firing

#### Test 1: Trigger InstanceDown Alert

```bash
# Stop node_exporter
sudo systemctl stop node_exporter

# Wait 1-2 minutes
sleep 120

# Check Prometheus alerts
curl -s http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | select(.labels.alertname=="InstanceDown")'
```

**Expected**:
- Alert state: "pending" → "firing"
- Labels include: alertname, instance, job, severity

**Restore**:
```bash
sudo systemctl start node_exporter
```

#### Test 2: Check Alert in Alertmanager

Trong Alertmanager UI:

**Expected**:
- Alert appears in Alerts tab
- Shows correct labels and annotations
- Summary and description visible

### ✅ 8. Alert Annotations Working

Check alert has proper annotations:

```bash
curl -s http://localhost:9090/api/v1/alerts | jq '.data.alerts[0].annotations'
```

**Expected**:
```json
{
  "summary": "Instance localhost:9100 is down",
  "description": "localhost:9100 of job node_exporter has been down for more than 1 minute."
}
```

### ✅ 9. Alert Grouping Working

Generate multiple alerts và verify grouping:

```bash
# Stop multiple services
sudo systemctl stop node_exporter
sudo systemctl stop custom-metrics-app

# Wait and check Alertmanager UI
```

**Expected**:
- Alerts grouped by alertname and severity
- Single notification for group (not individual alerts)

### ✅ 10. Notification Delivery (if configured)

**For Email**:
- Check inbox for alert email
- Verify subject line correct
- Verify alert details in body

**For Slack**:
- Check Slack channel
- Verify message received
- Verify alert details visible

**For Webhook**:
```bash
# Check webhook logs
# Verify POST request received with alert payload
```

## Advanced Verification

### Test 1: Alert Lifecycle

Monitor complete alert lifecycle:

1. **Inactive** → No alert
2. **Pending** → Condition met, waiting for `for` duration
3. **Firing** → Alert active, notification sent
4. **Resolved** → Condition no longer met

```bash
# Trigger alert
sudo systemctl stop node_exporter

# Check state progression
watch -n 10 'curl -s http://localhost:9090/api/v1/alerts | jq ".data.alerts[] | select(.labels.alertname==\"InstanceDown\") | .state"'

# Resolve alert
sudo systemctl start node_exporter
```

**Expected**: State transitions: pending → firing → (empty/resolved)

### Test 2: Alert Routing

Verify different severity levels route correctly:

```bash
# Check routing config
curl -s http://localhost:9093/api/v2/status | jq '.config.route'
```

**Expected**:
- Default receiver configured
- Routes for different severities (if configured)
- Grouping parameters visible

### Test 3: Inhibition Rules

Test that critical alerts inhibit warnings:

1. Trigger critical alert (e.g., InstanceDown)
2. Trigger warning alert on same instance
3. Verify warning is inhibited

```bash
curl -s http://localhost:9093/api/v2/alerts | jq '.[] | select(.status.inhibitedBy | length > 0)'
```

**Expected**: Warning alerts show inhibitedBy field

### Test 4: Silences

Create and verify silence:

```bash
# Create silence
amtool silence add alertname=HighCPUUsage \
  --duration=1h \
  --author="test" \
  --comment="Testing silences"

# List silences
amtool silence query

# Check in UI
```

**Expected**:
- Silence appears in Alertmanager UI
- Matching alerts show as silenced
- Notifications not sent for silenced alerts

### Test 5: Alert Metrics

Check Alertmanager metrics:

```promql
# Total alerts
alertmanager_alerts

# Alerts by state
alertmanager_alerts{state="active"}
alertmanager_alerts{state="suppressed"}

# Notifications sent
alertmanager_notifications_total

# Notification failures
alertmanager_notifications_failed_total
```

**Expected**: Metrics reflect actual alert activity

### Test 6: Repeat Interval

Verify repeat_interval works:

1. Trigger alert
2. Wait for first notification
3. Wait for repeat_interval duration
4. Verify notification sent again

**Expected**: Notifications repeat according to configuration

### Test 7: Group Wait and Interval

Test grouping timing:

```bash
# Trigger multiple alerts quickly
sudo systemctl stop node_exporter
sudo systemctl stop custom-metrics-app

# Monitor notification timing
```

**Expected**:
- First notification after `group_wait` (30s)
- Subsequent alerts batched with `group_interval` (5m)

## Practical Use Cases Verification

### Use Case 1: High Error Rate Alert

```bash
# Generate errors
for i in {1..200}; do
    curl -s http://localhost:8080/error > /dev/null
    sleep 0.1
done

# Wait 1-2 minutes
# Check alert fires
```

**Expected**: HighErrorRate alert fires when error rate > 5%

### Use Case 2: High Latency Alert

```bash
# Generate slow requests (if app supports)
# Or modify app to add artificial delay

# Check alert
```

**Expected**: HighLatency alert fires when p95 > 500ms

### Use Case 3: Disk Space Alert

```bash
# Check current disk usage
df -h /

# If space > 80%, alert should fire
```

**Expected**: DiskSpaceLow alert fires when space < 20%

### Use Case 4: Memory Pressure Alert

```bash
# Generate memory pressure
stress --vm 2 --vm-bytes 1G --timeout 300s

# Wait 2-3 minutes
# Check alert
```

**Expected**: HighMemoryUsage alert fires when usage > 85%

## Integration Verification

### Test 1: Prometheus → Alertmanager Flow

```bash
# Check Prometheus sends alerts to Alertmanager
curl -s http://localhost:9090/api/v1/alertmanagers | jq .
```

**Expected**:
- Alertmanager URL listed
- State: "up"
- Last push successful

### Test 2: Alertmanager → Receiver Flow

Check notification delivery:

```bash
# Check Alertmanager logs
sudo journalctl -u alertmanager -n 50 | grep -i "notify"
```

**Expected**:
- Log entries showing notification attempts
- Success messages (or error details if failed)

### Test 3: End-to-End Alert Flow

Complete flow test:

1. Trigger condition (stop service)
2. Wait for `for` duration
3. Alert fires in Prometheus
4. Alert sent to Alertmanager
5. Alertmanager groups alert
6. Notification sent to receiver
7. Verify notification received

**Expected**: Complete flow works within expected timeframes

## Common Issues & Solutions

### Issue: Alerts Not Firing

**Diagnosis**:
```bash
# Check rules loaded
curl http://localhost:9090/api/v1/rules | jq '.data.groups[] | select(.name=="lab_alerts")'

# Check evaluation
curl http://localhost:9090/api/v1/alerts | jq .

# Check Prometheus logs
sudo journalctl -u prometheus | grep -i alert
```

**Solution**: Verify rules syntax, reload Prometheus

### Issue: Notifications Not Sent

**Diagnosis**:
```bash
# Check Alertmanager logs
sudo journalctl -u alertmanager -f

# Check config
amtool check-config /etc/alertmanager/alertmanager.yml

# Test receiver manually
```

**Solution**: Fix receiver configuration, check credentials

### Issue: Too Many Notifications

**Diagnosis**:
```bash
# Check repeat_interval
curl http://localhost:9093/api/v2/status | jq '.config.route.repeat_interval'

# Check grouping
curl http://localhost:9093/api/v2/status | jq '.config.route.group_by'
```

**Solution**: Increase repeat_interval, improve grouping

### Issue: Alerts Flapping

**Diagnosis**:
```bash
# Check alert history
curl http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | {name: .labels.alertname, state: .state, activeAt: .activeAt}'
```

**Solution**: Increase `for` duration, adjust thresholds

## Performance Benchmarks

Expected performance:

| Metric | Expected Value |
|--------|----------------|
| Alertmanager memory | 20-50 MB |
| Alert evaluation time | < 1s |
| Notification latency | < 30s |
| Alerts processed/sec | > 100 |

## Success Criteria

Lab này được coi là hoàn thành khi:

- ✅ Alertmanager service running và stable
- ✅ Alert rules loaded và evaluating
- ✅ Prometheus connected to Alertmanager
- ✅ Alerts fire when conditions met
- ✅ Alerts appear in Alertmanager UI
- ✅ Notifications delivered (if configured)
- ✅ Grouping working correctly
- ✅ Silences working
- ✅ Inhibition rules working (if configured)
- ✅ Alert lifecycle complete (pending → firing → resolved)
- ✅ No errors in logs

## Documentation Checklist

Document your alerting setup:

- [ ] List of all alert rules và thresholds
- [ ] Notification channels configured
- [ ] Routing rules documented
- [ ] Runbook links for each alert
- [ ] On-call procedures
- [ ] Escalation paths

## Tiếp Theo

Nếu tất cả verifications pass:

1. ✅ Mark Lab 04 as complete
2. 📝 Document your alert rules và runbooks
3. 🎯 Proceed to [Lab 05 - Exporters](../lab-05-exporters/README.md)

Nếu có issues:
- Review [Troubleshooting section](./instructions.md#troubleshooting)
- Check Prometheus và Alertmanager logs
- Validate configurations
- Test notification channels separately
