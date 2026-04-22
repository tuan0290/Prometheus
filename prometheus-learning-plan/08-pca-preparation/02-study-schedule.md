# PCA Study Schedule

## Tổng Quan

Lịch học 5 tuần được thiết kế để chuẩn bị toàn diện cho kỳ thi PCA. Mỗi tuần tập trung vào một hoặc hai domains, với thời gian ôn tập và practice questions.

**Tổng thời gian**: ~40 giờ (8 giờ/tuần)

## Tuần 1: Nền Tảng (Domain 1 + 2)

**Mục tiêu**: Nắm vững Observability Concepts và Prometheus Fundamentals

### Ngày 1-2: Observability Concepts (Domain 1 - 18%)

| Thời gian | Nội dung | Tài liệu |
|-----------|----------|----------|
| 2 giờ | Metrics, Logs, Events, Traces | [01-fundamentals/01-monitoring-observability.md](../01-fundamentals/01-monitoring-observability.md) |
| 1 giờ | Push vs Pull model | [01-fundamentals/04-prometheus-architecture.md](../01-fundamentals/04-prometheus-architecture.md) |
| 1 giờ | SLIs, SLOs, SLAs | [08-pca-preparation/03-domain-observability.md](./03-domain-observability.md) |

**Checklist**:
- [ ] Hiểu sự khác biệt giữa metrics, logs, traces
- [ ] Giải thích được push vs pull model
- [ ] Định nghĩa được SLI, SLO, SLA

### Ngày 3-5: Prometheus Fundamentals (Domain 2 - 20%)

| Thời gian | Nội dung | Tài liệu |
|-----------|----------|----------|
| 2 giờ | Architecture và data model | [01-fundamentals/04-prometheus-architecture.md](../01-fundamentals/04-prometheus-architecture.md) |
| 1 giờ | Metric types | [01-fundamentals/03-metrics-types.md](../01-fundamentals/03-metrics-types.md) |
| 1 giờ | Service discovery | [02-components/05-service-discovery.md](../02-components/05-service-discovery.md) |
| 1 giờ | Storage và retention | [02-components/06-storage-retention.md](../02-components/06-storage-retention.md) |

**Checklist**:
- [ ] Vẽ được Prometheus architecture diagram
- [ ] Phân biệt được 4 metric types
- [ ] Hiểu các service discovery mechanisms
- [ ] Cấu hình được retention policy

### Cuối Tuần: Review + Practice

- [ ] Làm 20 practice questions Domain 1 + 2
- [ ] Review weak areas
- [ ] Ghi chú key concepts

---

## Tuần 2: PromQL (Domain 3 - 28%)

**Mục tiêu**: Thành thạo PromQL - domain quan trọng nhất

### Ngày 1-2: PromQL Basics

| Thời gian | Nội dung | Tài liệu |
|-----------|----------|----------|
| 2 giờ | Selectors, matchers, data types | [03-promql/01-promql-basics.md](../03-promql/01-promql-basics.md) |
| 1 giờ | Instant vs range vectors | [03-promql/02-data-types.md](../03-promql/02-data-types.md) |
| 1 giờ | Operators | [03-promql/03-operators.md](../03-promql/03-operators.md) |

### Ngày 3-4: Functions và Aggregations

| Thời gian | Nội dung | Tài liệu |
|-----------|----------|----------|
| 2 giờ | rate(), increase(), histogram_quantile() | [03-promql/04-functions.md](../03-promql/04-functions.md) |
| 2 giờ | sum, avg, topk, by, without | [03-promql/05-aggregations.md](../03-promql/05-aggregations.md) |

### Ngày 5: Advanced PromQL

| Thời gian | Nội dung | Tài liệu |
|-----------|----------|----------|
| 2 giờ | Subqueries, offset, recording rules | [03-promql/06-advanced-queries.md](../03-promql/06-advanced-queries.md) |

**Checklist**:
- [ ] Viết được instant và range vector queries
- [ ] Sử dụng được rate() và increase()
- [ ] Tính được histogram quantiles
- [ ] Viết được aggregations với by/without
- [ ] Hiểu subqueries và offset

### Cuối Tuần: PromQL Practice

- [ ] Làm Lab 03 - PromQL Queries
- [ ] Làm 30 practice questions Domain 3
- [ ] Review tất cả PromQL functions

---

## Tuần 3: Instrumentation (Domain 4 - 20%)

**Mục tiêu**: Hiểu cách instrument applications và sử dụng exporters

### Ngày 1-2: Client Libraries

| Thời gian | Nội dung | Tài liệu |
|-----------|----------|----------|
| 2 giờ | Go, Python, Java client libraries | [04-instrumentation/01-client-libraries.md](../04-instrumentation/01-client-libraries.md) |
| 2 giờ | Custom metrics implementation | [04-instrumentation/02-custom-metrics.md](../04-instrumentation/02-custom-metrics.md) |

### Ngày 3-4: Exporters

| Thời gian | Nội dung | Tài liệu |
|-----------|----------|----------|
| 2 giờ | Common exporters (node, blackbox, mysql, postgres) | [04-instrumentation/03-common-exporters.md](../04-instrumentation/03-common-exporters.md) |
| 1 giờ | Pushgateway use cases | [02-components/04-pushgateway.md](../02-components/04-pushgateway.md) |

### Ngày 5: Best Practices

| Thời gian | Nội dung | Tài liệu |
|-----------|----------|----------|
| 2 giờ | Naming conventions, cardinality | [04-instrumentation/04-best-practices.md](../04-instrumentation/04-best-practices.md) |

**Checklist**:
- [ ] Implement Counter, Gauge, Histogram, Summary
- [ ] Biết khi nào dùng mỗi metric type
- [ ] Cấu hình được node_exporter
- [ ] Hiểu Pushgateway use cases và limitations
- [ ] Áp dụng naming conventions

### Cuối Tuần: Hands-on Practice

- [ ] Làm Lab 02 - Custom Metrics
- [ ] Làm Lab 05 - Exporters
- [ ] Làm 20 practice questions Domain 4

---

## Tuần 4: Alerting (Domain 5 - 14%)

**Mục tiêu**: Cấu hình alerting và dashboarding

### Ngày 1-2: Alerting Rules

| Thời gian | Nội dung | Tài liệu |
|-----------|----------|----------|
| 2 giờ | Alert rules syntax, states | [05-alerting/01-alerting-rules.md](../05-alerting/01-alerting-rules.md) |
| 2 giờ | Alertmanager configuration | [05-alerting/02-alertmanager-config.md](../05-alerting/02-alertmanager-config.md) |

### Ngày 3-4: Notifications và Grafana

| Thời gian | Nội dung | Tài liệu |
|-----------|----------|----------|
| 2 giờ | Notification channels | [05-alerting/03-notification-channels.md](../05-alerting/03-notification-channels.md) |
| 2 giờ | Grafana integration | [05-alerting/04-grafana-integration.md](../05-alerting/04-grafana-integration.md) |

**Checklist**:
- [ ] Viết được alerting rules với PromQL
- [ ] Hiểu alert lifecycle (inactive → pending → firing)
- [ ] Cấu hình được Alertmanager routing
- [ ] Tạo được Grafana dashboard

### Cuối Tuần: Alerting Practice

- [ ] Làm Lab 04 - Alertmanager
- [ ] Làm 15 practice questions Domain 5
- [ ] Review tất cả domains

---

## Tuần 5: Ôn Tập Tổng Hợp

**Mục tiêu**: Consolidate knowledge và final preparation

### Ngày 1-2: Full Review

| Thời gian | Nội dung |
|-----------|----------|
| 2 giờ | Review Domain 1 + 2 |
| 2 giờ | Review Domain 3 (PromQL focus) |
| 2 giờ | Review Domain 4 + 5 |

### Ngày 3-4: Practice Exam

- [ ] Làm full practice exam (60 câu, 90 phút)
- [ ] Review tất cả câu sai
- [ ] Identify remaining weak areas
- [ ] Targeted review của weak areas

### Ngày 5: Final Preparation

- [ ] Light review của key concepts
- [ ] Review PromQL cheat sheet
- [ ] Test technical setup
- [ ] Đăng ký lịch thi

### Cuối Tuần: Rest

- Nghỉ ngơi, không học quá nhiều
- Nhẹ nhàng review notes
- Chuẩn bị tâm lý

---

## Daily Study Tips

### Buổi Sáng (30 phút)
- Review notes từ hôm trước
- Làm 5-10 practice questions

### Buổi Tối (1-2 giờ)
- Học nội dung mới
- Hands-on practice
- Ghi chú key concepts

### Cuối Tuần (3-4 giờ)
- Deep dive vào topics khó
- Làm practice tests
- Review và consolidate

## Tracking Progress

Sử dụng bảng này để track tiến độ:

| Domain | Tuần | Tự đánh giá (1-5) | Practice Score |
|--------|------|-------------------|----------------|
| Domain 1: Observability | 1 | | |
| Domain 2: Fundamentals | 1 | | |
| Domain 3: PromQL | 2 | | |
| Domain 4: Instrumentation | 3 | | |
| Domain 5: Alerting | 4 | | |
| Full Review | 5 | | |

**Thang điểm tự đánh giá**:
- 1: Chưa hiểu
- 2: Hiểu cơ bản
- 3: Hiểu tốt
- 4: Tự tin
- 5: Thành thạo

## Tài Nguyên Bổ Sung

- [Prometheus Documentation](https://prometheus.io/docs/)
- [PromQL Cheat Sheet](../09-reference/01-promql-quick-reference.md)
- [Practice Questions](./08-practice-questions.md)
- [Exam Tips](./09-exam-tips.md)

## Tiếp Theo

- [Domain 1: Observability](./03-domain-observability.md)
