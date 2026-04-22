# Lab 06 - Service Discovery Instructions

## Mục Tiêu

Trong lab này, bạn sẽ thiết lập service discovery để tự động discover targets. Bạn sẽ configure file-based service discovery và learn về relabeling.

## Yêu Cầu

- Lab 01-05 đã hoàn thành
- Prometheus running
- Understanding of labels và relabeling

## Phần 1: File-Based Service Discovery

### Bước 1: Create Service Discovery Directory

```bash
sudo mkdir -p /etc/prometheus/file_sd
sudo chown prometheus:prometheus /etc/prometheus/file_sd
```

### Bước 2: Create Target Files

#### File 1: Node Exporters

```bash
sudo vim /etc/prometheus/file_sd/node_exporters.json
```

Nội dung:

```json
[
  {
    "targets": ["localhost:9100"],
    "labels": {
      "job": "node",
      "env": "production",
      "datacenter": "dc1",
      "team": "infrastructure"
    }
  }
]
```

#### File 2: Application Servers

```bash
sudo vim /etc/prometheus/file_sd/applications.json
```

Nội dung:

```json
[
  {
    "targets": ["localhost:8080"],
    "labels": {
      "job": "custom_app",
      "env": "production",
      "app": "metrics-demo",
      "version": "1.0.0"
    }
  },
  {
    "targets": ["localhost:9090"],
    "labels": {
      "job": "prometheus",
      "env": "production",
      "app": "prometheus-server"
    }
  }
]
```

#### File 3: Databases

```bash
sudo vim /etc/prometheus/file_sd/databases.json
```

Nội dung:

```json
[
  {
    "targets": ["localhost:9187"],
    "labels": {
      "job": "postgres",
      "env": "production",
      "database": "main",
      "team": "database"
    }
  }
]
```

Set ownership:

```bash
sudo chown prometheus:prometheus /etc/prometheus/file_sd/*.json
```

### Bước 3: Update Prometheus Configuration

Edit `/etc/prometheus/prometheus.yml`:

```bash
sudo vim /etc/prometheus/prometheus.yml
```

Add file_sd_configs:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'lab-cluster'
    region: 'us-east-1'

scrape_configs:
  # File-based service discovery for node exporters
  - job_name: 'file_sd_nodes'
    file_sd_configs:
      - files:
          - '/etc/prometheus/file_sd/node_exporters.json'
        refresh_interval: 30s

  # File-based service discovery for applications
  - job_name: 'file_sd_apps'
    file_sd_configs:
      - files:
          - '/etc/prometheus/file_sd/applications.json'
        refresh_interval: 30s

  # File-based service discovery for databases
  - job_name: 'file_sd_databases'
    file_sd_configs:
      - files:
          - '/etc/prometheus/file_sd/databases.json'
        refresh_interval: 30s

  # Wildcard pattern for all JSON files
  - job_name: 'file_sd_all'
    file_sd_configs:
      - files:
          - '/etc/prometheus/file_sd/*.json'
        refresh_interval: 30s
```

### Bước 4: Reload Prometheus

```bash
# Validate config
sudo /usr/local/bin/promtool check config /etc/prometheus/prometheus.yml

# Reload
curl -X POST http://localhost:9090/-/reload
```

### Bước 5: Verify Targets

Trong Prometheus UI:
1. Navigate to **Status** → **Targets**
2. Verify `file_sd_*` jobs
3. Check labels are applied correctly

## Phần 2: Relabeling

### Bước 1: Basic Relabeling

Update Prometheus config với relabeling:

```yaml
scrape_configs:
  - job_name: 'file_sd_with_relabel'
    file_sd_configs:
      - files:
          - '/etc/prometheus/file_sd/*.json'
        refresh_interval: 30s
    
    relabel_configs:
      # Keep only production targets
      - source_labels: [env]
        regex: 'production'
        action: keep
      
      # Add instance_name label from target
      - source_labels: [__address__]
        target_label: instance_name
        regex: '([^:]+):.*'
        replacement: '${1}'
      
      # Add custom label
      - target_label: monitored_by
        replacement: 'prometheus-lab'
```

### Bước 2: Advanced Relabeling Examples

```yaml
scrape_configs:
  - job_name: 'file_sd_advanced_relabel'
    file_sd_configs:
      - files:
          - '/etc/prometheus/file_sd/*.json'
    
    relabel_configs:
      # Drop targets with specific label
      - source_labels: [env]
        regex: 'dev|test'
        action: drop
      
      # Rename label
      - source_labels: [datacenter]
        target_label: dc
      
      # Combine labels
      - source_labels: [env, app]
        separator: '-'
        target_label: service_id
      
      # Extract port from address
      - source_labels: [__address__]
        regex: '.*:([0-9]+)'
        target_label: port
        replacement: '${1}'
      
      # Set metrics path based on label
      - source_labels: [job]
        regex: 'custom_app'
        target_label: __metrics_path__
        replacement: '/metrics'
      
      # Add scheme
      - source_labels: [__address__]
        regex: '.*'
        target_label: __scheme__
        replacement: 'http'
```

### Bước 3: Metric Relabeling

Add metric_relabel_configs:

```yaml
scrape_configs:
  - job_name: 'file_sd_metric_relabel'
    file_sd_configs:
      - files:
          - '/etc/prometheus/file_sd/*.json'
    
    metric_relabel_configs:
      # Drop specific metrics
      - source_labels: [__name__]
        regex: 'go_.*'
        action: drop
      
      # Keep only specific metrics
      - source_labels: [__name__]
        regex: '(up|app_requests_total|node_cpu_seconds_total)'
        action: keep
      
      # Rename metric
      - source_labels: [__name__]
        regex: 'old_metric_name'
        target_label: __name__
        replacement: 'new_metric_name'
      
      # Drop high cardinality labels
      - regex: 'user_id|session_id'
        action: labeldrop
```

## Phần 3: Dynamic Target Updates

### Bước 1: Create Script to Update Targets

```bash
vim ~/update_targets.sh
```

Nội dung:

```bash
#!/bin/bash

# Update targets dynamically
cat > /etc/prometheus/file_sd/dynamic_targets.json <<EOF
[
  {
    "targets": ["localhost:9100", "localhost:8080"],
    "labels": {
      "job": "dynamic",
      "env": "production",
      "updated_at": "$(date -Iseconds)"
    }
  }
]
EOF

echo "Targets updated at $(date)"
```

Make executable:

```bash
chmod +x ~/update_targets.sh
```

### Bước 2: Test Dynamic Updates

```bash
# Run script
sudo ~/update_targets.sh

# Wait 30 seconds (refresh_interval)
sleep 30

# Check Prometheus UI - targets should update
```

### Bước 3: Automate with Cron (Optional)

```bash
# Add to crontab
sudo crontab -e

# Add line:
*/5 * * * * /home/<user>/update_targets.sh
```

## Phần 4: Multiple File Patterns

### Bước 1: Organize by Environment

```bash
# Create environment-specific files
sudo vim /etc/prometheus/file_sd/prod_targets.json
sudo vim /etc/prometheus/file_sd/staging_targets.json
sudo vim /etc/prometheus/file_sd/dev_targets.json
```

### Bước 2: Configure Separate Jobs

```yaml
scrape_configs:
  - job_name: 'production'
    file_sd_configs:
      - files:
          - '/etc/prometheus/file_sd/prod_*.json'
    relabel_configs:
      - target_label: env
        replacement: 'production'

  - job_name: 'staging'
    file_sd_configs:
      - files:
          - '/etc/prometheus/file_sd/staging_*.json'
    relabel_configs:
      - target_label: env
        replacement: 'staging'

  - job_name: 'development'
    file_sd_configs:
      - files:
          - '/etc/prometheus/file_sd/dev_*.json'
    relabel_configs:
      - target_label: env
        replacement: 'development'
```

## Phần 5: YAML Format (Alternative)

File-based SD also supports YAML:

```bash
sudo vim /etc/prometheus/file_sd/targets.yml
```

Nội dung:

```yaml
- targets:
    - localhost:9100
    - localhost:8080
  labels:
    job: mixed
    env: production
    format: yaml

- targets:
    - localhost:9090
  labels:
    job: prometheus
    env: production
```

Update Prometheus config:

```yaml
scrape_configs:
  - job_name: 'file_sd_yaml'
    file_sd_configs:
      - files:
          - '/etc/prometheus/file_sd/*.yml'
          - '/etc/prometheus/file_sd/*.yaml'
```

## Phần 6: Monitoring Service Discovery

### Query Service Discovery Metrics

```promql
# Number of discovered targets
count(up{job=~"file_sd_.*"})

# Targets by environment
count by (env) (up{job=~"file_sd_.*"})

# Targets by team
count by (team) (up{job=~"file_sd_.*"})

# Recently updated targets (if using updated_at label)
up{updated_at!=""}
```

### Check Target Metadata

```bash
# API endpoint for targets
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.scrapePool | startswith("file_sd"))'
```

## Phần 7: Best Practices

### Practice 1: Consistent Label Schema

```json
{
  "targets": ["host:port"],
  "labels": {
    "job": "service-name",
    "env": "production|staging|dev",
    "team": "team-name",
    "region": "us-east-1",
    "datacenter": "dc1",
    "version": "1.0.0"
  }
}
```

### Practice 2: Validation Script

```bash
vim ~/validate_targets.sh
```

Nội dung:

```bash
#!/bin/bash

for file in /etc/prometheus/file_sd/*.json; do
    echo "Validating $file..."
    if jq empty "$file" 2>/dev/null; then
        echo "✓ Valid JSON"
    else
        echo "✗ Invalid JSON"
        exit 1
    fi
done

echo "All files valid"
```

### Practice 3: Backup Targets

```bash
# Backup script
vim ~/backup_targets.sh
```

Nội dung:

```bash
#!/bin/bash

BACKUP_DIR="/var/backups/prometheus/file_sd"
mkdir -p "$BACKUP_DIR"

tar -czf "$BACKUP_DIR/targets_$(date +%Y%m%d_%H%M%S).tar.gz" \
    /etc/prometheus/file_sd/

# Keep only last 7 days
find "$BACKUP_DIR" -name "targets_*.tar.gz" -mtime +7 -delete
```

## Troubleshooting

### Issue 1: Targets Not Discovered

**Giải pháp**:
```bash
# Check file exists
ls -la /etc/prometheus/file_sd/

# Validate JSON
jq . /etc/prometheus/file_sd/node_exporters.json

# Check Prometheus logs
sudo journalctl -u prometheus | grep -i "file_sd"

# Verify refresh_interval passed
# Wait at least refresh_interval seconds
```

### Issue 2: Labels Not Applied

**Giải pháp**:
```bash
# Check target labels in UI
# Navigate to Status → Targets

# Query with labels
curl 'http://localhost:9090/api/v1/query?query=up{job="file_sd_nodes"}'

# Check relabel_configs syntax
sudo /usr/local/bin/promtool check config /etc/prometheus/prometheus.yml
```

### Issue 3: Relabeling Not Working

**Giải pháp**:
```bash
# Test relabeling with promtool
# Create test file
cat > /tmp/test_relabel.yml <<EOF
- source_labels: [env]
  regex: 'production'
  action: keep
EOF

# Check relabel config
sudo /usr/local/bin/promtool check config /etc/prometheus/prometheus.yml
```

### Issue 4: File Permission Issues

**Giải pháp**:
```bash
# Fix ownership
sudo chown -R prometheus:prometheus /etc/prometheus/file_sd/

# Fix permissions
sudo chmod 644 /etc/prometheus/file_sd/*.json

# Check Prometheus can read
sudo -u prometheus cat /etc/prometheus/file_sd/node_exporters.json
```

## Advanced Topics

### Topic 1: Consul Service Discovery (Reference)

```yaml
scrape_configs:
  - job_name: 'consul'
    consul_sd_configs:
      - server: 'localhost:8500'
        services: ['node-exporter', 'app']
```

### Topic 2: Kubernetes Service Discovery (Reference)

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
```

### Topic 3: DNS Service Discovery

```yaml
scrape_configs:
  - job_name: 'dns'
    dns_sd_configs:
      - names:
          - 'tasks.prometheus.local'
        type: 'A'
        port: 9090
```

## Tiếp Theo

Sau khi hoàn thành lab này, bạn đã có:
- ✅ File-based service discovery configured
- ✅ Dynamic target updates working
- ✅ Relabeling rules applied
- ✅ Understanding of service discovery patterns
- ✅ Automated target management

Tiếp tục với: [Verification](./verification.md) để kiểm tra kết quả, sau đó chuyển sang [PCA Preparation](../../08-pca-preparation/README.md)
