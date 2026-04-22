# Common Exporters

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Infrastructure Exporters](#infrastructure-exporters)
- [Database Exporters](#database-exporters)
- [Web Server Exporters](#web-server-exporters)
- [Message Queue Exporters](#message-queue-exporters)
- [Cloud Platform Exporters](#cloud-platform-exporters)
- [Application Exporters](#application-exporters)
- [Cách Sử Dụng Exporters](#cách-sử-dụng-exporters)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

## Giới Thiệu

Exporters là các service chuyển đổi metrics từ third-party systems sang định dạng Prometheus có thể scrape. Thay vì instrument từng application, bạn có thể sử dụng exporters có sẵn để monitor infrastructure, databases, và services.

**Lợi ích của exporters:**
- Không cần modify source code của third-party applications
- Cộng đồng maintain và update
- Standardized metrics cho common services
- Dễ dàng setup và configure

## Infrastructure Exporters

### Node Exporter

**Mục đích:** Monitor hardware và OS metrics (CPU, memory, disk, network)

**Platform:** Linux, macOS, BSD

**Metrics cung cấp:**
- CPU usage và load average
- Memory và swap usage
- Disk I/O và space
- Network traffic
- Filesystem statistics
- System uptime

**Cài đặt:**
```bash
# Download
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvfz node_exporter-*.tar.gz
cd node_exporter-*

# Run
./node_exporter
```

**Prometheus configuration:**
```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```

**Useful queries:**
```promql
# CPU usage percentage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk usage percentage
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100
```

**Repository:** https://github.com/prometheus/node_exporter

---

### Windows Exporter

**Mục đích:** Monitor Windows systems

**Platform:** Windows

**Metrics cung cấp:**
- CPU, memory, disk metrics
- Windows services status
- IIS metrics
- .NET CLR metrics

**Cài đặt:**
```powershell
# Download MSI installer
# https://github.com/prometheus-community/windows_exporter/releases

# Install and run as Windows service
msiexec /i windows_exporter.msi ENABLED_COLLECTORS="cpu,memory,disk,net"
```

**Repository:** https://github.com/prometheus-community/windows_exporter

---

### Blackbox Exporter

**Mục đích:** Probe endpoints qua HTTP, HTTPS, DNS, TCP, ICMP

**Use cases:**
- HTTP/HTTPS endpoint monitoring
- SSL certificate expiration
- DNS resolution checks
- TCP port availability
- ICMP ping checks

**Cài đặt:**
```bash
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.24.0/blackbox_exporter-0.24.0.linux-amd64.tar.gz
tar xvfz blackbox_exporter-*.tar.gz
cd blackbox_exporter-*
./blackbox_exporter
```

**Configuration (blackbox.yml):**
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: []
      method: GET
      preferred_ip_protocol: "ip4"
  
  http_post_2xx:
    prober: http
    http:
      method: POST
      headers:
        Content-Type: application/json
      body: '{"key":"value"}'
  
  tcp_connect:
    prober: tcp
    timeout: 5s
  
  icmp:
    prober: icmp
    timeout: 5s
```

**Prometheus configuration:**
```yaml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://example.com
        - https://api.example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9115
```

**Useful queries:**
```promql
# Probe success rate
avg_over_time(probe_success[5m])

# HTTP response time
probe_http_duration_seconds

# SSL certificate expiry (days)
(probe_ssl_earliest_cert_expiry - time()) / 86400
```

**Repository:** https://github.com/prometheus/blackbox_exporter

## Database Exporters

### MySQL Exporter

**Mục đích:** Monitor MySQL/MariaDB databases

**Metrics cung cấp:**
- Query performance
- Connection statistics
- InnoDB metrics
- Replication status
- Table statistics

**Cài đặt:**
```bash
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz
tar xvfz mysqld_exporter-*.tar.gz
cd mysqld_exporter-*
```

**Setup database user:**
```sql
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'password' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;
```

**Run exporter:**
```bash
export DATA_SOURCE_NAME='exporter:password@(localhost:3306)/'
./mysqld_exporter
```

**Useful queries:**
```promql
# Queries per second
rate(mysql_global_status_queries[5m])

# Slow queries
rate(mysql_global_status_slow_queries[5m])

# Connection usage
mysql_global_status_threads_connected / mysql_global_variables_max_connections

# Replication lag
mysql_slave_status_seconds_behind_master
```

**Repository:** https://github.com/prometheus/mysqld_exporter

---

### PostgreSQL Exporter

**Mục đích:** Monitor PostgreSQL databases

**Metrics cung cấp:**
- Database size và connections
- Query statistics
- Table và index statistics
- Replication metrics
- Lock statistics

**Cài đặt:**
```bash
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.15.0/postgres_exporter-0.15.0.linux-amd64.tar.gz
tar xvfz postgres_exporter-*.tar.gz
cd postgres_exporter-*
```

**Run exporter:**
```bash
export DATA_SOURCE_NAME="postgresql://postgres:password@localhost:5432/postgres?sslmode=disable"
./postgres_exporter
```

**Useful queries:**
```promql
# Active connections
pg_stat_activity_count

# Database size
pg_database_size_bytes

# Transaction rate
rate(pg_stat_database_xact_commit[5m])

# Cache hit ratio
rate(pg_stat_database_blks_hit[5m]) / 
  (rate(pg_stat_database_blks_hit[5m]) + rate(pg_stat_database_blks_read[5m]))
```

**Repository:** https://github.com/prometheus-community/postgres_exporter

---

### Redis Exporter

**Mục đích:** Monitor Redis instances

**Metrics cung cấp:**
- Memory usage
- Connected clients
- Command statistics
- Keyspace statistics
- Replication info

**Cài đặt:**
```bash
docker run -d --name redis_exporter \
  -p 9121:9121 \
  oliver006/redis_exporter \
  --redis.addr=redis://localhost:6379
```

**Useful queries:**
```promql
# Memory usage
redis_memory_used_bytes

# Connected clients
redis_connected_clients

# Commands per second
rate(redis_commands_processed_total[5m])

# Hit rate
rate(redis_keyspace_hits_total[5m]) / 
  (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
```

**Repository:** https://github.com/oliver006/redis_exporter

---

### MongoDB Exporter

**Mục đích:** Monitor MongoDB databases

**Metrics cung cấp:**
- Database operations
- Connection statistics
- Memory usage
- Replication status
- Collection statistics

**Cài đặt:**
```bash
docker run -d -p 9216:9216 \
  percona/mongodb_exporter:0.40 \
  --mongodb.uri=mongodb://localhost:27017
```

**Repository:** https://github.com/percona/mongodb_exporter

## Web Server Exporters

### NGINX Exporter

**Mục đích:** Monitor NGINX web server

**Requirements:** NGINX stub_status module enabled

**NGINX configuration:**
```nginx
server {
    location /stub_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

**Run exporter:**
```bash
docker run -p 9113:9113 nginx/nginx-prometheus-exporter:latest \
  -nginx.scrape-uri=http://localhost:8080/stub_status
```

**Metrics:**
- Active connections
- Requests per second
- Reading/writing/waiting connections

**Repository:** https://github.com/nginxinc/nginx-prometheus-exporter

---

### Apache Exporter

**Mục đích:** Monitor Apache HTTP Server

**Requirements:** mod_status enabled

**Apache configuration:**
```apache
<Location "/server-status">
    SetHandler server-status
    Require local
</Location>
```

**Run exporter:**
```bash
docker run -p 9117:9117 \
  lusotycoon/apache-exporter \
  --scrape_uri=http://localhost/server-status?auto
```

**Repository:** https://github.com/Lusitaniae/apache_exporter

## Message Queue Exporters

### RabbitMQ Exporter

**Mục đích:** Monitor RabbitMQ message broker

**Metrics cung cấp:**
- Queue depth và rates
- Consumer statistics
- Connection và channel counts
- Node metrics

**Run exporter:**
```bash
docker run -d -p 9419:9419 \
  kbudde/rabbitmq-exporter \
  -rabbit-url=http://localhost:15672 \
  -rabbit-user=guest \
  -rabbit-password=guest
```

**Useful queries:**
```promql
# Messages in queue
rabbitmq_queue_messages

# Message rate
rate(rabbitmq_queue_messages_published_total[5m])

# Consumer count
rabbitmq_queue_consumers
```

**Repository:** https://github.com/kbudde/rabbitmq_exporter

---

### Kafka Exporter

**Mục đích:** Monitor Apache Kafka

**Metrics cung cấp:**
- Topic và partition metrics
- Consumer lag
- Broker metrics

**Run exporter:**
```bash
docker run -p 9308:9308 \
  danielqsj/kafka-exporter \
  --kafka.server=kafka:9092
```

**Repository:** https://github.com/danielqsj/kafka_exporter

## Cloud Platform Exporters

### AWS CloudWatch Exporter

**Mục đích:** Export AWS CloudWatch metrics to Prometheus

**Services supported:**
- EC2, RDS, ELB, S3
- Lambda, DynamoDB
- CloudFront, API Gateway

**Configuration (config.yml):**
```yaml
region: us-east-1
metrics:
  - aws_namespace: AWS/EC2
    aws_metric_name: CPUUtilization
    aws_dimensions: [InstanceId]
    aws_statistics: [Average]
  
  - aws_namespace: AWS/RDS
    aws_metric_name: DatabaseConnections
    aws_dimensions: [DBInstanceIdentifier]
    aws_statistics: [Average]
```

**Run exporter:**
```bash
docker run -p 9106:9106 \
  -v $(pwd)/config.yml:/config/config.yml \
  prom/cloudwatch-exporter
```

**Repository:** https://github.com/prometheus/cloudwatch_exporter

---

### Azure Monitor Exporter

**Mục đích:** Export Azure Monitor metrics

**Repository:** https://github.com/RobustPerception/azure_metrics_exporter

---

### GCP Stackdriver Exporter

**Mục đích:** Export Google Cloud metrics

**Repository:** https://github.com/prometheus-community/stackdriver_exporter

## Application Exporters

### JMX Exporter

**Mục đích:** Export JMX metrics từ Java applications

**Use cases:**
- Java application monitoring
- Tomcat, Kafka, Cassandra
- Custom JMX beans

**Configuration (config.yml):**
```yaml
lowercaseOutputName: true
lowercaseOutputLabelNames: true
rules:
  - pattern: 'java.lang<type=Memory><HeapMemoryUsage>(\w+)'
    name: jvm_memory_heap_$1
    type: GAUGE
```

**Run as Java agent:**
```bash
java -javaagent:jmx_prometheus_javaagent.jar=8080:config.yml -jar yourapp.jar
```

**Repository:** https://github.com/prometheus/jmx_exporter

---

### Elasticsearch Exporter

**Mục đích:** Monitor Elasticsearch clusters

**Metrics:**
- Cluster health
- Node statistics
- Index statistics
- JVM metrics

**Run exporter:**
```bash
docker run -p 9114:9114 \
  quay.io/prometheuscommunity/elasticsearch-exporter:latest \
  --es.uri=http://localhost:9200
```

**Repository:** https://github.com/prometheus-community/elasticsearch_exporter

## Cách Sử Dụng Exporters

### Bước 1: Cài Đặt Exporter

**Option 1: Binary**
```bash
wget <exporter-url>
tar xvfz <exporter>.tar.gz
cd <exporter>
./<exporter>
```

**Option 2: Docker**
```bash
docker run -d -p <port>:<port> <exporter-image>
```

**Option 3: Systemd Service**
```bash
# Create systemd service file
sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

### Bước 2: Configure Prometheus

**Thêm scrape config:**
```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
    
  - job_name: 'mysql'
    static_configs:
      - targets: ['localhost:9104']
    
  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']
```

### Bước 3: Verify Metrics

**Check exporter endpoint:**
```bash
curl http://localhost:9100/metrics
```

**Query in Prometheus:**
```promql
up{job="node"}
```

### Bước 4: Create Dashboards

Import pre-built Grafana dashboards:
- Node Exporter: Dashboard ID 1860
- MySQL: Dashboard ID 7362
- PostgreSQL: Dashboard ID 9628

## Tài Liệu Liên Quan

- [Exporters](../02-components/02-exporters.md) - Tổng quan về exporters
- [Client Libraries](./01-client-libraries.md) - Tạo custom exporters
- [Service Discovery](../02-components/05-service-discovery.md) - Auto-discover exporters
- [Lab 05 - Exporters](../07-labs/lab-05-exporters/README.md) - Bài tập thực hành

## Tài Liệu Tham Khảo

- [Exporters and Integrations](https://prometheus.io/docs/instrumenting/exporters/) - Official list
- [Prometheus Community](https://github.com/prometheus-community) - Community exporters
- [Awesome Prometheus](https://github.com/roaldnefs/awesome-prometheus) - Curated list of exporters
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/) - Pre-built dashboards
