# Exporters Reference

## Official Exporters

### node_exporter

**Mô tả**: Collect hardware và OS metrics từ Linux/Unix systems  
**Port**: 9100  
**GitHub**: https://github.com/prometheus/node_exporter

**Key Metrics**:
```promql
node_cpu_seconds_total          # CPU time by mode
node_memory_MemAvailable_bytes  # Available memory
node_filesystem_avail_bytes     # Available disk space
node_network_receive_bytes_total # Network received
node_load1                      # 1-minute load average
node_disk_io_time_seconds_total # Disk I/O time
```

### blackbox_exporter

**Mô tả**: Probe endpoints over HTTP, HTTPS, DNS, TCP, ICMP  
**Port**: 9115  
**GitHub**: https://github.com/prometheus/blackbox_exporter

**Key Metrics**:
```promql
probe_success                    # 1 if probe succeeded
probe_http_status_code           # HTTP status code
probe_http_duration_seconds      # Response time
probe_ssl_earliest_cert_expiry   # SSL cert expiry timestamp
probe_dns_lookup_time_seconds    # DNS lookup time
```

### pushgateway

**Mô tả**: Push metrics từ batch jobs  
**Port**: 9091  
**GitHub**: https://github.com/prometheus/pushgateway

### alertmanager

**Mô tả**: Handle alerts từ Prometheus  
**Port**: 9093  
**GitHub**: https://github.com/prometheus/alertmanager

---

## Database Exporters

### postgres_exporter

**Mô tả**: PostgreSQL metrics  
**Port**: 9187  
**GitHub**: https://github.com/prometheus-community/postgres_exporter

**Key Metrics**:
```promql
pg_up                           # PostgreSQL up
pg_stat_activity_count          # Active connections
pg_database_size_bytes          # Database size
pg_stat_database_xact_commit    # Committed transactions
pg_locks_count                  # Lock count
pg_replication_lag              # Replication lag
```

### mysql_exporter (mysqld_exporter)

**Mô tả**: MySQL/MariaDB metrics  
**Port**: 9104  
**GitHub**: https://github.com/prometheus/mysqld_exporter

**Key Metrics**:
```promql
mysql_up                        # MySQL up
mysql_global_status_connections # Total connections
mysql_global_status_queries     # Total queries
mysql_global_status_slow_queries # Slow queries
mysql_global_variables_max_connections # Max connections
```

### redis_exporter

**Mô tả**: Redis metrics  
**Port**: 9121  
**GitHub**: https://github.com/oliver006/redis_exporter

**Key Metrics**:
```promql
redis_up                        # Redis up
redis_connected_clients         # Connected clients
redis_memory_used_bytes         # Memory usage
redis_keyspace_hits_total       # Cache hits
redis_keyspace_misses_total     # Cache misses
```

### mongodb_exporter

**Mô tả**: MongoDB metrics  
**Port**: 9216  
**GitHub**: https://github.com/percona/mongodb_exporter

---

## Infrastructure Exporters

### haproxy_exporter

**Mô tả**: HAProxy metrics  
**Port**: 9101  
**GitHub**: https://github.com/prometheus/haproxy_exporter

### nginx_exporter (nginx-prometheus-exporter)

**Mô tả**: NGINX metrics  
**Port**: 9113  
**GitHub**: https://github.com/nginxinc/nginx-prometheus-exporter

### apache_exporter

**Mô tả**: Apache HTTP Server metrics  
**Port**: 9117  
**GitHub**: https://github.com/Lusitaniae/apache_exporter

### elasticsearch_exporter

**Mô tả**: Elasticsearch metrics  
**Port**: 9114  
**GitHub**: https://github.com/prometheus-community/elasticsearch_exporter

---

## Cloud Exporters

### aws_cloudwatch_exporter

**Mô tả**: AWS CloudWatch metrics  
**Port**: 9106  
**GitHub**: https://github.com/prometheus/cloudwatch_exporter

### azure_metrics_exporter

**Mô tả**: Azure Monitor metrics  
**GitHub**: https://github.com/RobustPerception/azure_metrics_exporter

---

## Kubernetes Exporters

### kube-state-metrics

**Mô tả**: Kubernetes object state metrics  
**Port**: 8080  
**GitHub**: https://github.com/kubernetes/kube-state-metrics

**Key Metrics**:
```promql
kube_pod_status_phase           # Pod phase
kube_deployment_status_replicas # Deployment replicas
kube_node_status_condition      # Node conditions
kube_namespace_status_phase     # Namespace phase
```

### cAdvisor

**Mô tả**: Container resource usage metrics  
**Port**: 8080 (built into kubelet)  
**GitHub**: https://github.com/google/cadvisor

**Key Metrics**:
```promql
container_cpu_usage_seconds_total    # Container CPU
container_memory_usage_bytes         # Container memory
container_network_receive_bytes_total # Container network
container_fs_usage_bytes             # Container filesystem
```

---

## JVM Exporters

### jmx_exporter

**Mô tả**: JMX metrics từ Java applications  
**Port**: Configurable  
**GitHub**: https://github.com/prometheus/jmx_exporter

### micrometer

**Mô tả**: Application metrics cho Spring Boot  
**GitHub**: https://micrometer.io/

---

## Exporter Ports Reference

| Port | Exporter |
|------|---------|
| 9090 | Prometheus |
| 9091 | Pushgateway |
| 9093 | Alertmanager |
| 9100 | node_exporter |
| 9101 | haproxy_exporter |
| 9104 | mysqld_exporter |
| 9106 | cloudwatch_exporter |
| 9113 | nginx_exporter |
| 9114 | elasticsearch_exporter |
| 9115 | blackbox_exporter |
| 9117 | apache_exporter |
| 9121 | redis_exporter |
| 9187 | postgres_exporter |
| 9216 | mongodb_exporter |

**Full list**: https://prometheus.io/docs/instrumenting/exporters/

---

## Choosing an Exporter

| Use Case | Exporter |
|----------|---------|
| Linux system metrics | node_exporter |
| HTTP endpoint monitoring | blackbox_exporter |
| PostgreSQL | postgres_exporter |
| MySQL | mysqld_exporter |
| Redis | redis_exporter |
| Kubernetes objects | kube-state-metrics |
| Container metrics | cAdvisor |
| Batch jobs | pushgateway |
| Custom metrics | prometheus_client library |
