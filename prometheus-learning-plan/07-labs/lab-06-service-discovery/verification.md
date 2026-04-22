# Lab 06 - Service Discovery Verification

## Mục Đích

Document này giúp bạn verify rằng file-based service discovery đã được cấu hình đúng và targets được discovered tự động.

## Checklist Verification

### ✅ 1. Service Discovery Files Exist

```bash
ls -la /etc/prometheus/file_sd/
```

**Expected**:
- Multiple JSON/YAML files
- Owned by prometheus:prometheus
- Readable permissions (644)

### ✅ 2. JSON Files Valid

```bash
for file in /etc/prometheus/file_sd/*.json; do
    echo "Checking $file..."
    jq . "$file"
done
```

**Expected**: All files parse successfully without errors

### ✅ 3. Prometheus Config Valid

```bash
sudo /usr/local/bin/promtool check config /etc/prometheus/prometheus.yml
```

**Expected**: SUCCESS message, no syntax errors

### ✅ 4. File SD Jobs in Prometheus

Trong Prometheus UI (`http://<server-ip>:9090`):

Navigate to **Status** → **Targets**

**Expected**:
- Jobs starting with `file_sd_*` visible
- Targets discovered from JSON files
- All targets showing UP status

### ✅ 5. Targets Discovered

Query:

```promql
up{job=~"file_sd_.*"}
```

**Expected**:
- Multiple time series
- All values = 1 (targets up)
- Labels from JSON files applied

### ✅ 6. Labels Applied Correctly

Check target labels:

```bash
curl -s 'http://localhost:9090/api/v1/targets' | jq '.data.activeTargets[] | select(.scrapePool | startswith("file_sd")) | {job: .labels.job, env: .labels.env, team: .labels.team}'
```

**Expected**:
- Labels from JSON files present
- Correct values for env, team, etc.

### ✅ 7. Dynamic Updates Working

#### Test 1: Add New Target

```bash
# Update file
sudo vim /etc/prometheus/file_sd/node_exporters.json
```

Add new target:

```json
[
  {
    "targets": ["localhost:9100", "localhost:9115"],
    "labels": {
      "job": "node",
      "env": "production"
    }
  }
]
```

Wait refresh_interval (30s), then check Prometheus UI.

**Expected**: New target appears automatically

#### Test 2: Remove Target

Remove target from JSON, wait 30s.

**Expected**: Target disappears from Prometheus

#### Test 3: Update Labels

Change label values, wait 30s.

**Expected**: Labels update in Prometheus

### ✅ 8. Relabeling Working

If relabel_configs configured, verify:

```promql
# Check relabeled metrics
up{monitored_by="prometheus-lab"}

# Check renamed labels
up{dc!=""}

# Check combined labels
up{service_id!=""}
```

**Expected**: Relabeling rules applied correctly

### ✅ 9. Metric Relabeling Working

If metric_relabel_configs configured:

```promql
# Check dropped metrics are gone
go_goroutines{job=~"file_sd_.*"}

# Check kept metrics exist
up{job=~"file_sd_.*"}
```

**Expected**: Only allowed metrics present

### ✅ 10. Refresh Interval Working

Monitor target updates:

```bash
# Update file
echo "Updated at $(date)" | sudo tee -a /etc/prometheus/file_sd/node_exporters.json

# Watch for changes
watch -n 5 'curl -s http://localhost:9090/api/v1/targets | jq ".data.activeTargets[] | select(.scrapePool==\"file_sd_nodes\") | .lastScrape"'
```

**Expected**: Changes detected within refresh_interval

## Advanced Verification

### Test 1: Multiple File Patterns

```bash
# Create files with different patterns
sudo touch /etc/prometheus/file_sd/prod_app1.json
sudo touch /etc/prometheus/file_sd/prod_app2.json
sudo touch /etc/prometheus/file_sd/staging_app1.json
```

Add content and verify Prometheus discovers all matching files.

**Expected**: All files matching pattern discovered

### Test 2: Environment-Based Discovery

Query targets by environment:

```promql
count by (env) (up{job=~"file_sd_.*"})
```

**Expected**:
- Separate counts for production, staging, dev
- Matches actual target distribution

### Test 3: Team-Based Discovery

```promql
count by (team) (up{job=~"file_sd_.*"})
```

**Expected**: Targets grouped by team label

### Test 4: Relabel Action: Keep

Configure relabel to keep only production:

```yaml
relabel_configs:
  - source_labels: [env]
    regex: 'production'
    action: keep
```

**Expected**: Only production targets scraped

### Test 5: Relabel Action: Drop

Configure relabel to drop dev targets:

```yaml
relabel_configs:
  - source_labels: [env]
    regex: 'dev'
    action: drop
```

**Expected**: Dev targets not scraped

### Test 6: Label Extraction

Extract hostname from address:

```yaml
relabel_configs:
  - source_labels: [__address__]
    regex: '([^:]+):.*'
    target_label: hostname
    replacement: '${1}'
```

Query:

```promql
up{hostname!=""}
```

**Expected**: hostname label populated

### Test 7: Port Extraction

Extract port from address:

```yaml
relabel_configs:
  - source_labels: [__address__]
    regex: '.*:([0-9]+)'
    target_label: port
    replacement: '${1}'
```

Query:

```promql
up{port!=""}
```

**Expected**: port label shows correct port numbers

## Integration Tests

### Test 1: End-to-End Discovery Flow

1. Create new JSON file
2. Add targets with labels
3. Wait refresh_interval
4. Verify targets in Prometheus UI
5. Query metrics with labels
6. Update file
7. Verify changes reflected

**Expected**: Complete flow works smoothly

### Test 2: Validation Pipeline

```bash
# Create validation script
cat > /tmp/validate_and_deploy.sh <<'EOF'
#!/bin/bash

# Validate JSON
if ! jq empty /tmp/new_targets.json; then
    echo "Invalid JSON"
    exit 1
fi

# Deploy
sudo cp /tmp/new_targets.json /etc/prometheus/file_sd/
sudo chown prometheus:prometheus /etc/prometheus/file_sd/new_targets.json

echo "Deployed successfully"
EOF

chmod +x /tmp/validate_and_deploy.sh
```

**Expected**: Only valid files deployed

### Test 3: Rollback Capability

```bash
# Backup current state
sudo cp /etc/prometheus/file_sd/node_exporters.json /tmp/backup.json

# Make breaking change
echo "invalid json" | sudo tee /etc/prometheus/file_sd/node_exporters.json

# Wait and observe (targets should disappear)

# Rollback
sudo cp /tmp/backup.json /etc/prometheus/file_sd/node_exporters.json
```

**Expected**: Can rollback to previous state

## Practical Use Cases Verification

### Use Case 1: Multi-Environment Monitoring

```promql
# Production targets
count(up{env="production"})

# Staging targets
count(up{env="staging"})

# All environments
count by (env) (up)
```

**Expected**: Clear separation by environment

### Use Case 2: Team-Based Monitoring

```promql
# Infrastructure team targets
up{team="infrastructure"}

# Database team targets
up{team="database"}

# Targets per team
count by (team) (up)
```

**Expected**: Targets organized by team

### Use Case 3: Application Versioning

```promql
# Targets by version
count by (version) (up{job="custom_app"})

# Old versions
up{version=~"0\\..*"}
```

**Expected**: Can track application versions

### Use Case 4: Datacenter Distribution

```promql
# Targets per datacenter
count by (datacenter) (up)

# Specific datacenter
up{datacenter="dc1"}
```

**Expected**: Geographic distribution visible

## Performance Verification

### Test 1: Discovery Latency

Measure time from file update to target discovery:

```bash
# Update file and timestamp
echo "$(date +%s)" | sudo tee /tmp/update_time
sudo vim /etc/prometheus/file_sd/node_exporters.json

# Check when Prometheus discovers
# Compare with update_time
```

**Expected**: Discovery within refresh_interval + scrape_interval

### Test 2: Large File Handling

Create file with many targets:

```bash
# Generate 100 targets
python3 <<EOF
import json
targets = [{"targets": [f"host{i}:9100"], "labels": {"job": "node", "id": str(i)}} for i in range(100)]
with open('/tmp/large_targets.json', 'w') as f:
    json.dump(targets, f)
EOF

sudo cp /tmp/large_targets.json /etc/prometheus/file_sd/
```

**Expected**: Prometheus handles large files efficiently

### Test 3: Update Frequency

Test rapid updates:

```bash
for i in {1..10}; do
    echo "Update $i" | sudo tee -a /etc/prometheus/file_sd/test.json
    sleep 5
done
```

**Expected**: Prometheus handles frequent updates

## Common Issues & Solutions

### Issue: Targets Not Discovered

**Diagnosis**:
```bash
# Check file exists
ls -la /etc/prometheus/file_sd/

# Check JSON valid
jq . /etc/prometheus/file_sd/*.json

# Check Prometheus logs
sudo journalctl -u prometheus | grep file_sd

# Check file permissions
sudo -u prometheus cat /etc/prometheus/file_sd/node_exporters.json
```

**Solution**: Fix JSON syntax, permissions, or file path

### Issue: Labels Missing

**Diagnosis**:
```bash
# Check labels in file
jq '.[] | .labels' /etc/prometheus/file_sd/node_exporters.json

# Check labels in Prometheus
curl 'http://localhost:9090/api/v1/query?query=up{job="file_sd_nodes"}' | jq '.data.result[0].metric'
```

**Solution**: Verify labels in JSON file, check relabel_configs

### Issue: Updates Not Reflected

**Diagnosis**:
```bash
# Check refresh_interval
grep refresh_interval /etc/prometheus/prometheus.yml

# Check file modification time
stat /etc/prometheus/file_sd/node_exporters.json

# Force reload
curl -X POST http://localhost:9090/-/reload
```

**Solution**: Wait for refresh_interval, check file was actually modified

### Issue: Relabeling Not Working

**Diagnosis**:
```bash
# Validate config
sudo /usr/local/bin/promtool check config /etc/prometheus/prometheus.yml

# Check relabel_configs syntax
grep -A 10 relabel_configs /etc/prometheus/prometheus.yml
```

**Solution**: Fix relabel_configs syntax, reload Prometheus

## Success Criteria

Lab này được coi là hoàn thành khi:

- ✅ File-based service discovery configured
- ✅ Targets discovered automatically from JSON files
- ✅ Labels applied correctly
- ✅ Dynamic updates working (add/remove/update targets)
- ✅ Relabeling rules working
- ✅ Refresh interval working as expected
- ✅ Multiple file patterns supported
- ✅ Can query targets by labels
- ✅ Validation and rollback procedures in place
- ✅ No errors in Prometheus logs

## Documentation Checklist

Document your service discovery setup:

- [ ] File naming conventions
- [ ] Label schema
- [ ] Relabeling rules
- [ ] Refresh intervals
- [ ] Validation procedures
- [ ] Rollback procedures
- [ ] Team responsibilities

## Tiếp Theo

Nếu tất cả verifications pass:

1. ✅ Mark Lab 06 as complete
2. ✅ All 6 labs completed!
3. 📝 Review all labs và consolidate learnings
4. 🎯 Proceed to [PCA Preparation](../../08-pca-preparation/README.md)

Nếu có issues:
- Review [Troubleshooting section](./instructions.md#troubleshooting)
- Check Prometheus logs
- Validate JSON files
- Verify file permissions
- Test with simple examples first

## Congratulations!

Bạn đã hoàn thành tất cả 6 labs trong Prometheus Learning Plan! Bạn giờ đã có:

- ✅ Prometheus và exporters installed
- ✅ Custom metrics implementation
- ✅ PromQL query skills
- ✅ Alerting với Alertmanager
- ✅ Multiple exporters integration
- ✅ Service discovery automation

Bạn đã sẵn sàng cho PCA certification preparation!
