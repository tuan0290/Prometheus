# Cloud Providers Integration

## Mục Lục
- [Tổng Quan](#tổng-quan)
- [Kiến Trúc](#kiến-trúc)
- [Cách Hoạt Động](#cách-hoạt-động)
- [Use Cases](#use-cases)
- [Cấu Hình](#cấu-hình)
  - [AWS CloudWatch Exporter](#aws-cloudwatch-exporter)
  - [GCP Stackdriver Exporter](#gcp-stackdriver-exporter)
  - [Azure Monitor Exporter](#azure-monitor-exporter)
- [Best Practices](#best-practices)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

---

## Tổng Quan

Các cloud providers (AWS, GCP, Azure) đều có hệ thống monitoring riêng (CloudWatch, Stackdriver, Azure Monitor). Để tích hợp với Prometheus, chúng ta sử dụng các exporters chuyên dụng để chuyển đổi cloud metrics sang định dạng Prometheus.

**Lợi ích của việc tích hợp cloud metrics vào Prometheus:**
- Unified monitoring: Xem tất cả metrics (cloud + on-prem + containers) trong một nơi
- Grafana dashboards: Tạo dashboards kết hợp nhiều nguồn dữ liệu
- Alerting nhất quán: Dùng cùng một hệ thống alerting cho mọi infrastructure
- Cost visibility: Theo dõi cloud costs qua metrics

---

## Kiến Trúc

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloud Providers                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │     AWS      │  │     GCP      │  │     Azure        │  │
│  │  CloudWatch  │  │ Stackdriver  │  │  Azure Monitor   │  │
│  │  Metrics     │  │  Metrics     │  │  Metrics         │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
└─────────┼─────────────────┼──────────────────┼─────────────┘
          │                 │                  │
          ▼                 ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐
│  CloudWatch  │  │  Stackdriver │  │  Azure Monitor       │
│  Exporter   │  │  Exporter    │  │  Exporter            │
│  :9106      │  │  :9255       │  │  :9276               │
└──────┬───────┘  └──────┬───────┘  └────────┬─────────────┘
       │                 │                   │
       └─────────────────┼───────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │   Prometheus Server  │
              │   scrape all         │
              └──────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │       Grafana        │
              └──────────────────────┘
```

---

## Cách Hoạt Động

### Pull Model với Cloud APIs

Cloud exporters hoạt động theo mô hình:
1. Exporter gọi Cloud API (CloudWatch API, Stackdriver API, Azure Monitor API)
2. Lấy metrics từ cloud provider
3. Chuyển đổi sang định dạng Prometheus
4. Expose qua HTTP endpoint `/metrics`
5. Prometheus scrape endpoint định kỳ

### Authentication

Mỗi cloud provider có cơ chế authentication riêng:
- **AWS**: IAM roles, access keys, hoặc instance profiles
- **GCP**: Service accounts với JSON key files
- **Azure**: Service principals với client credentials

---

## Use Cases

1. **Multi-cloud monitoring**: Theo dõi infrastructure trên nhiều cloud providers
2. **Hybrid monitoring**: Kết hợp on-premises và cloud metrics
3. **Cost monitoring**: Theo dõi cloud spending qua metrics
4. **SLA tracking**: Đo lường availability của cloud services
5. **Capacity planning**: Phân tích usage trends để tối ưu costs
6. **Compliance**: Audit trail cho cloud resource usage

---

## Cấu Hình

### AWS CloudWatch Exporter

**Cài đặt:**

```bash
# Chạy bằng Docker
docker run -d \
  --name cloudwatch-exporter \
  -p 9106:9106 \
  -v $(pwd)/cloudwatch-config.yml:/config/config.yml \
  -e AWS_ACCESS_KEY_ID=your_access_key \
  -e AWS_SECRET_ACCESS_KEY=your_secret_key \
  -e AWS_DEFAULT_REGION=us-east-1 \
  prom/cloudwatch-exporter:latest

# Hoặc dùng IAM role (khuyến nghị cho EC2/ECS)
docker run -d \
  --name cloudwatch-exporter \
  -p 9106:9106 \
  -v $(pwd)/cloudwatch-config.yml:/config/config.yml \
  prom/cloudwatch-exporter:latest
```

**IAM Policy cần thiết:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "cloudwatch:GetMetricData",
        "tag:GetResources"
      ],
      "Resource": "*"
    }
  ]
}
```

**Cấu hình cloudwatch-config.yml:**

```yaml
# cloudwatch-config.yml
region: us-east-1
period_seconds: 300
set_timestamp: false

metrics:
  # EC2 Instance metrics
  - aws_namespace: AWS/EC2
    aws_metric_name: CPUUtilization
    aws_dimensions: [InstanceId]
    aws_statistics: [Average, Maximum]
    aws_tag_select:
      tag_selections:
        Environment: [production]

  - aws_namespace: AWS/EC2
    aws_metric_name: NetworkIn
    aws_dimensions: [InstanceId]
    aws_statistics: [Sum]

  - aws_namespace: AWS/EC2
    aws_metric_name: NetworkOut
    aws_dimensions: [InstanceId]
    aws_statistics: [Sum]

  # RDS metrics
  - aws_namespace: AWS/RDS
    aws_metric_name: CPUUtilization
    aws_dimensions: [DBInstanceIdentifier]
    aws_statistics: [Average]

  - aws_namespace: AWS/RDS
    aws_metric_name: DatabaseConnections
    aws_dimensions: [DBInstanceIdentifier]
    aws_statistics: [Average]

  - aws_namespace: AWS/RDS
    aws_metric_name: FreeStorageSpace
    aws_dimensions: [DBInstanceIdentifier]
    aws_statistics: [Average]

  - aws_namespace: AWS/RDS
    aws_metric_name: ReadLatency
    aws_dimensions: [DBInstanceIdentifier]
    aws_statistics: [Average]

  # ELB/ALB metrics
  - aws_namespace: AWS/ApplicationELB
    aws_metric_name: RequestCount
    aws_dimensions: [LoadBalancer]
    aws_statistics: [Sum]

  - aws_namespace: AWS/ApplicationELB
    aws_metric_name: TargetResponseTime
    aws_dimensions: [LoadBalancer]
    aws_statistics: [Average, p99]

  - aws_namespace: AWS/ApplicationELB
    aws_metric_name: HTTPCode_Target_5XX_Count
    aws_dimensions: [LoadBalancer]
    aws_statistics: [Sum]

  # SQS metrics
  - aws_namespace: AWS/SQS
    aws_metric_name: NumberOfMessagesSent
    aws_dimensions: [QueueName]
    aws_statistics: [Sum]

  - aws_namespace: AWS/SQS
    aws_metric_name: ApproximateNumberOfMessagesVisible
    aws_dimensions: [QueueName]
    aws_statistics: [Average]

  # Lambda metrics
  - aws_namespace: AWS/Lambda
    aws_metric_name: Invocations
    aws_dimensions: [FunctionName]
    aws_statistics: [Sum]

  - aws_namespace: AWS/Lambda
    aws_metric_name: Errors
    aws_dimensions: [FunctionName]
    aws_statistics: [Sum]

  - aws_namespace: AWS/Lambda
    aws_metric_name: Duration
    aws_dimensions: [FunctionName]
    aws_statistics: [Average, p99]

  # ElastiCache metrics
  - aws_namespace: AWS/ElastiCache
    aws_metric_name: CPUUtilization
    aws_dimensions: [CacheClusterId]
    aws_statistics: [Average]

  - aws_namespace: AWS/ElastiCache
    aws_metric_name: CacheHits
    aws_dimensions: [CacheClusterId]
    aws_statistics: [Sum]

  - aws_namespace: AWS/ElastiCache
    aws_metric_name: CacheMisses
    aws_dimensions: [CacheClusterId]
    aws_statistics: [Sum]
```

**Cấu hình Prometheus để scrape CloudWatch Exporter:**

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'cloudwatch'
    static_configs:
      - targets: ['cloudwatch-exporter:9106']
    scrape_interval: 5m  # CloudWatch có độ trễ ~5 phút
    scrape_timeout: 2m
```

**Queries PromQL cho AWS:**

```promql
# EC2 CPU utilization trung bình
avg(aws_ec2_cpuutilization_average) by (instance_id)

# RDS connections
aws_rds_database_connections_average{dbinstance_identifier="my-db"}

# ALB error rate
rate(aws_applicationelb_httpcode_target_5xx_count_sum[5m]) /
rate(aws_applicationelb_request_count_sum[5m]) * 100

# SQS queue depth
aws_sqs_approximate_number_of_messages_visible_average{queue_name="my-queue"}

# Lambda error rate
rate(aws_lambda_errors_sum[5m]) / rate(aws_lambda_invocations_sum[5m]) * 100
```

---

### GCP Stackdriver Exporter

**Cài đặt:**

```bash
# Tạo service account
gcloud iam service-accounts create prometheus-stackdriver \
  --display-name="Prometheus Stackdriver Exporter"

# Gán roles
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:prometheus-stackdriver@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/monitoring.viewer"

# Tạo key file
gcloud iam service-accounts keys create stackdriver-key.json \
  --iam-account=prometheus-stackdriver@YOUR_PROJECT_ID.iam.gserviceaccount.com

# Chạy exporter
docker run -d \
  --name stackdriver-exporter \
  -p 9255:9255 \
  -v $(pwd)/stackdriver-key.json:/key.json \
  -e GOOGLE_APPLICATION_CREDENTIALS=/key.json \
  prometheuscommunity/stackdriver-exporter:latest \
  --google.project-id=YOUR_PROJECT_ID \
  --monitoring.metrics-type-prefixes="compute.googleapis.com/instance/cpu,compute.googleapis.com/instance/memory"
```

**Cấu hình chi tiết với nhiều metrics:**

```bash
docker run -d \
  --name stackdriver-exporter \
  -p 9255:9255 \
  -v $(pwd)/stackdriver-key.json:/key.json \
  -e GOOGLE_APPLICATION_CREDENTIALS=/key.json \
  prometheuscommunity/stackdriver-exporter:latest \
  --google.project-id=YOUR_PROJECT_ID \
  --monitoring.metrics-type-prefixes="\
    compute.googleapis.com/instance/cpu,\
    compute.googleapis.com/instance/memory,\
    compute.googleapis.com/instance/disk,\
    compute.googleapis.com/instance/network,\
    cloudsql.googleapis.com/database/cpu,\
    cloudsql.googleapis.com/database/memory,\
    cloudsql.googleapis.com/database/disk,\
    pubsub.googleapis.com/subscription/num_undelivered_messages,\
    pubsub.googleapis.com/topic/send_message_operation_count,\
    storage.googleapis.com/api/request_count,\
    run.googleapis.com/request_count,\
    run.googleapis.com/request_latencies" \
  --monitoring.metrics-interval=5m \
  --monitoring.metrics-offset=0s
```

**Cấu hình Prometheus:**

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'stackdriver'
    static_configs:
      - targets: ['stackdriver-exporter:9255']
    scrape_interval: 5m
    scrape_timeout: 4m
```

**Queries PromQL cho GCP:**

```promql
# GCE instance CPU utilization
stackdriver_gce_instance_compute_googleapis_com_instance_cpu_utilization{
  project_id="my-project"
}

# Cloud SQL CPU
stackdriver_cloudsql_database_cloudsql_googleapis_com_database_cpu_utilization

# Pub/Sub undelivered messages
stackdriver_pubsub_subscription_pubsub_googleapis_com_subscription_num_undelivered_messages

# Cloud Run request count
rate(stackdriver_cloud_run_revision_run_googleapis_com_request_count[5m])

# Cloud Storage request count
rate(stackdriver_gcs_bucket_storage_googleapis_com_api_request_count[5m])
```

---

### Azure Monitor Exporter

**Cài đặt:**

```bash
# Tạo Service Principal
az ad sp create-for-rbac \
  --name "prometheus-azure-monitor" \
  --role "Monitoring Reader" \
  --scopes /subscriptions/YOUR_SUBSCRIPTION_ID

# Output sẽ có: appId, password, tenant
# Lưu lại để dùng trong cấu hình

# Chạy exporter
docker run -d \
  --name azure-monitor-exporter \
  -p 9276:9276 \
  -e AZURE_SUBSCRIPTION_ID=your_subscription_id \
  -e AZURE_TENANT_ID=your_tenant_id \
  -e AZURE_CLIENT_ID=your_client_id \
  -e AZURE_CLIENT_SECRET=your_client_secret \
  -v $(pwd)/azure-config.yml:/etc/azure-metrics-exporter/config.yml \
  webdevops/azure-metrics-exporter:latest
```

**Cấu hình azure-config.yml:**

```yaml
# azure-config.yml
targets:
  # Virtual Machines
  - resource: /subscriptions/YOUR_SUB_ID/resourceGroups/my-rg/providers/Microsoft.Compute/virtualMachines/my-vm
    metrics:
      - name: Percentage CPU
        aggregations:
          - Average
          - Maximum
      - name: Network In Total
        aggregations:
          - Total
      - name: Network Out Total
        aggregations:
          - Total
      - name: Disk Read Bytes
        aggregations:
          - Total
      - name: Disk Write Bytes
        aggregations:
          - Total

  # Azure SQL Database
  - resource: /subscriptions/YOUR_SUB_ID/resourceGroups/my-rg/providers/Microsoft.Sql/servers/my-server/databases/my-db
    metrics:
      - name: cpu_percent
        aggregations:
          - Average
      - name: dtu_consumption_percent
        aggregations:
          - Average
      - name: storage_percent
        aggregations:
          - Average
      - name: connection_successful
        aggregations:
          - Total
      - name: connection_failed
        aggregations:
          - Total

  # Azure App Service
  - resource: /subscriptions/YOUR_SUB_ID/resourceGroups/my-rg/providers/Microsoft.Web/sites/my-app
    metrics:
      - name: Requests
        aggregations:
          - Total
      - name: Http5xx
        aggregations:
          - Total
      - name: AverageResponseTime
        aggregations:
          - Average
      - name: CpuTime
        aggregations:
          - Total
      - name: MemoryWorkingSet
        aggregations:
          - Average

  # Azure Storage Account
  - resource: /subscriptions/YOUR_SUB_ID/resourceGroups/my-rg/providers/Microsoft.Storage/storageAccounts/mystorage
    metrics:
      - name: Transactions
        aggregations:
          - Total
      - name: Ingress
        aggregations:
          - Total
      - name: Egress
        aggregations:
          - Total
      - name: SuccessE2ELatency
        aggregations:
          - Average
```

**Cấu hình Prometheus:**

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'azure-monitor'
    static_configs:
      - targets: ['azure-monitor-exporter:9276']
    scrape_interval: 5m
    scrape_timeout: 4m
```

**Queries PromQL cho Azure:**

```promql
# VM CPU percentage
azure_vm_percentage_cpu_average{resource_group="my-rg"}

# SQL Database DTU consumption
azure_sql_dtu_consumption_percent_average

# App Service request count
rate(azure_app_service_requests_total[5m])

# App Service 5xx errors
rate(azure_app_service_http5xx_total[5m])

# Storage transactions
rate(azure_storage_transactions_total[5m])
```

---

## Best Practices

1. **Dùng IAM roles/service accounts**: Không hardcode credentials, dùng IAM roles cho EC2/GCE/Azure VMs
2. **Scrape interval phù hợp**: Cloud metrics thường có độ trễ 1-5 phút, đặt scrape_interval >= 5m
3. **Giới hạn metrics**: Chỉ collect metrics cần thiết để tránh API rate limits và costs
4. **Caching**: Một số exporters hỗ trợ caching để giảm API calls
5. **Multi-region**: Deploy exporter gần với cloud region để giảm latency
6. **Error handling**: Cấu hình alerts khi exporter không thể kết nối cloud API
7. **Cost awareness**: CloudWatch API calls có chi phí, tối ưu số metrics và frequency
8. **Tagging**: Dùng cloud resource tags để filter và group metrics
9. **Namespace isolation**: Deploy exporters trong namespace riêng
10. **Secret management**: Dùng Kubernetes Secrets hoặc Vault để quản lý credentials

---

## Tài Liệu Liên Quan

- [Kubernetes Integration](./01-kubernetes-integration.md) - Monitor Kubernetes trên cloud
- [Monitoring Stacks](./04-monitoring-stacks.md) - Complete monitoring stack
- [Common Exporters](../04-instrumentation/03-common-exporters.md) - Danh sách exporters
- [Service Discovery](../02-components/05-service-discovery.md) - Service discovery mechanisms

---

## Tài Liệu Tham Khảo

- [CloudWatch Exporter](https://github.com/prometheus/cloudwatch_exporter)
- [Stackdriver Exporter](https://github.com/prometheus-community/stackdriver_exporter)
- [Azure Monitor Exporter](https://github.com/webdevops/azure-metrics-exporter)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [GCP Monitoring API](https://cloud.google.com/monitoring/api/v3)
- [Azure Monitor REST API](https://docs.microsoft.com/en-us/azure/azure-monitor/essentials/rest-api-walkthrough)
