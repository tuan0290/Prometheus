# Lab 01 - Installation Instructions

## Mục Tiêu

Trong lab này, bạn sẽ cài đặt Prometheus server và node_exporter từ đầu, cấu hình scraping, và verify metrics collection.

## Yêu Cầu

- Linux system (Ubuntu/Debian hoặc CentOS/RHEL)
- Root hoặc sudo access
- Internet connection để download binaries
- Port 9090 và 9100 available

## Phần 1: Cài Đặt Prometheus Server

### Bước 1: Download Prometheus

```bash
# Tạo user cho Prometheus
sudo useradd --no-create-home --shell /bin/false prometheus

# Tạo directories
sudo mkdir -p /etc/prometheus /var/lib/prometheus

# Download Prometheus (check latest version tại https://prometheus.io/download/)
# Prometheus 3.5.2 là bản LTS (Long Term Support)
cd /tmp
PROM_VERSION="3.5.2"
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz

# Extract
tar -xzf prometheus-${PROM_VERSION}.linux-amd64.tar.gz
cd prometheus-${PROM_VERSION}.linux-amd64
```

### Bước 2: Install Binaries

```bash
# Copy binaries
sudo cp prometheus promtool /usr/local/bin/

# Set ownership
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

### Bước 3: Tạo Configuration File

Tạo file `/etc/prometheus/prometheus.yml`:

```bash
sudo vim /etc/prometheus/prometheus.yml
```

Nội dung:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

Set ownership:

```bash
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

### Bước 4: Tạo Systemd Service

Tạo file `/etc/systemd/system/prometheus.service`:

```bash
sudo vim /etc/systemd/system/prometheus.service
```

Nội dung:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.enable-lifecycle

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

> **Lưu ý Prometheus 3.x**: Các flags `--web.console.templates` và `--web.console.libraries` đã bị xóa hoàn toàn. Flags `--storage.tsdb.retention.time` và `--storage.tsdb.retention.size` đã deprecated — nên cấu hình retention trong `prometheus.yml` thay thế.

### Bước 5: Start Prometheus

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable và start service
sudo systemctl enable prometheus
sudo systemctl start prometheus

# Check status
sudo systemctl status prometheus
```

### Bước 6: Verify Prometheus Web UI

Mở browser và truy cập:

```
http://<your-server-ip>:9090
```

Bạn sẽ thấy Prometheus web interface.

## Phần 2: Cài Đặt node_exporter

### Bước 1: Download node_exporter

```bash
cd /tmp
NE_VERSION="1.9.1"
wget https://github.com/prometheus/node_exporter/releases/download/v${NE_VERSION}/node_exporter-${NE_VERSION}.linux-amd64.tar.gz

# Extract
tar -xzf node_exporter-${NE_VERSION}.linux-amd64.tar.gz
cd node_exporter-${NE_VERSION}.linux-amd64
```

### Bước 2: Install Binary

```bash
# Copy binary
sudo cp node_exporter /usr/local/bin/

# Set ownership
sudo chown prometheus:prometheus /usr/local/bin/node_exporter
```

### Bước 3: Tạo Systemd Service

Tạo file `/etc/systemd/system/node_exporter.service`:

```bash
sudo vim /etc/systemd/system/node_exporter.service
```

Nội dung:

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/node_exporter

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### Bước 4: Start node_exporter

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable và start service
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

# Check status
sudo systemctl status node_exporter
```

### Bước 5: Verify node_exporter Metrics

```bash
# Check metrics endpoint
curl http://localhost:9100/metrics
```

Bạn sẽ thấy nhiều metrics về system (CPU, memory, disk, network).

## Phần 3: Verify Prometheus Scraping

### Bước 1: Check Targets trong Prometheus UI

1. Truy cập `http://<your-server-ip>:9090`
2. Click vào **Status** → **Targets**
3. Bạn sẽ thấy 2 targets:
   - `prometheus` (localhost:9090) - State: UP
   - `node_exporter` (localhost:9100) - State: UP

### Bước 2: Query Metrics

Trong Prometheus UI, thử các queries sau:

```promql
# Check Prometheus is up
up

# Check node_exporter metrics
node_cpu_seconds_total

# CPU usage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
```

## Phần 4: Configuration Management

### Reload Configuration

Khi bạn thay đổi `prometheus.yml`, reload config mà không cần restart:

```bash
# Method 1: Using systemctl
sudo systemctl reload prometheus

# Method 2: Using API (nếu --web.enable-lifecycle enabled)
curl -X POST http://localhost:9090/-/reload
```

### View Logs

```bash
# Prometheus logs
sudo journalctl -u prometheus -f

# node_exporter logs
sudo journalctl -u node_exporter -f
```

## Troubleshooting

### Issue 1: Prometheus không start

**Triệu chứng**: `systemctl status prometheus` shows failed

**Giải pháp**:
```bash
# Check logs
sudo journalctl -u prometheus -n 50

# Common issues:
# - Config file syntax error: validate với promtool
sudo /usr/local/bin/promtool check config /etc/prometheus/prometheus.yml

# - Permission issues
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

### Issue 2: Target DOWN trong Prometheus UI

**Triệu chứng**: Target shows state "DOWN"

**Giải pháp**:
```bash
# Check service is running
sudo systemctl status node_exporter

# Check port is listening
sudo netstat -tlnp | grep 9100

# Check firewall
sudo ufw status
sudo firewall-cmd --list-all

# Test connection
curl http://localhost:9100/metrics
```

### Issue 3: Cannot access Prometheus UI

**Triệu chứng**: Browser không connect được

**Giải pháp**:
```bash
# Check Prometheus is running
sudo systemctl status prometheus

# Check port is listening
sudo netstat -tlnp | grep 9090

# Check firewall rules
sudo ufw allow 9090/tcp
# hoặc
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload

# Check from server
curl http://localhost:9090
```

### Issue 4: Metrics không hiển thị

**Triệu chứng**: Query không trả về data

**Giải pháp**:
```bash
# Check scrape config
cat /etc/prometheus/prometheus.yml

# Check targets status
curl http://localhost:9090/api/v1/targets

# Check if metrics exist
curl http://localhost:9100/metrics | grep node_cpu

# Wait for first scrape (15s default)
```

## Best Practices

1. **Security**:
   - Chạy Prometheus với dedicated user (không dùng root)
   - Restrict network access với firewall
   - Cân nhắc dùng reverse proxy (nginx) với authentication

2. **Storage**:
   - Monitor disk usage của `/var/lib/prometheus`
   - Set retention policy phù hợp (default 15 days)
   - Cân nhắc dùng remote storage cho long-term retention

3. **Configuration**:
   - Backup `prometheus.yml` trước khi thay đổi
   - Validate config với `promtool` trước khi reload
   - Version control configuration files

4. **Monitoring**:
   - Monitor Prometheus itself (self-monitoring)
   - Set up alerts cho Prometheus down
   - Monitor scrape duration và failures

## Cleanup (Optional)

Nếu muốn remove installation:

```bash
# Stop services
sudo systemctl stop prometheus node_exporter
sudo systemctl disable prometheus node_exporter

# Remove files
sudo rm -rf /etc/prometheus /var/lib/prometheus
sudo rm /usr/local/bin/prometheus /usr/local/bin/promtool /usr/local/bin/node_exporter
sudo rm /etc/systemd/system/prometheus.service /etc/systemd/system/node_exporter.service

# Remove user
sudo userdel prometheus

# Reload systemd
sudo systemctl daemon-reload
```

## Tiếp Theo

Sau khi hoàn thành lab này, bạn đã có:
- ✅ Prometheus server running
- ✅ node_exporter collecting system metrics
- ✅ Prometheus scraping metrics
- ✅ Có thể query metrics qua UI

Tiếp tục với: [Verification](./verification.md) để kiểm tra kết quả, sau đó chuyển sang [Lab 02 - Custom Metrics](../lab-02-custom-metrics/README.md)
