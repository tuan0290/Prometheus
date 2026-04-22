# Lộ Trình Học Tập Prometheus

## Mục Lục

- [Tổng Quan](#tổng-quan)
- [Cấu Trúc Lộ Trình](#cấu-trúc-lộ-trình)
- [Cấp Độ Beginner (Tuần 1-2)](#cấp-độ-beginner-tuần-1-2)
- [Cấp Độ Intermediate (Tuần 3-4)](#cấp-độ-intermediate-tuần-3-4)
- [Cấp Độ Advanced (Tuần 5-6)](#cấp-độ-advanced-tuần-5-6)
- [Tài Liệu Tham Khảo (Liên Tục)](#tài-liệu-tham-khảo-liên-tục)
- [Lộ Trình Chi Tiết Theo Module](#lộ-trình-chi-tiết-theo-module)
- [Hướng Dẫn Sử Dụng Lộ Trình](#hướng-dẫn-sử-dụng-lộ-trình)
- [Chuẩn Bị Cho Chứng Chỉ PCA](#chuẩn-bị-cho-chứng-chỉ-pca)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Tổng Quan

Lộ trình học tập Prometheus được thiết kế cho người mới bắt đầu chưa có kinh nghiệm với Prometheus. Lộ trình này sẽ đưa bạn từ những khái niệm cơ bản nhất đến kiến thức nâng cao, bao gồm cả chuẩn bị cho chứng chỉ Prometheus Certified Associate (PCA).

**Tổng thời gian ước tính**: 60-80 giờ (6-8 tuần với 10-12 giờ/tuần)

**Mục tiêu cuối cùng**:
- Hiểu sâu về Prometheus và monitoring concepts
- Có khả năng triển khai và vận hành Prometheus trong production
- Viết được PromQL queries phức tạp
- Cấu hình alerting và integration với các công cụ khác
- Sẵn sàng cho kỳ thi PCA certification

## Cấu Trúc Lộ Trình

Lộ trình được chia thành 3 cấp độ chính:

```
┌─────────────────────────────────────────────────────────────┐
│                    LEARNING PATH STRUCTURE                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────┐      ┌────────────────┐                │
│  │   BEGINNER     │      │ INTERMEDIATE   │                │
│  │   (Weeks 1-2)  │  →   │  (Weeks 3-4)   │                │
│  │   20-25 hours  │      │  25-30 hours   │                │
│  └────────────────┘      └────────────────┘                │
│          │                       │                          │
│          │                       ▼                          │
│          │              ┌────────────────┐                  │
│          │              │   ADVANCED     │                  │
│          └──────────→   │  (Weeks 5-6)   │                  │
│                         │  25-30 hours   │                  │
│                         └────────────────┘                  │
│                                 │                           │
│                                 ▼                           │
│                         ┌────────────────┐                  │
│                         │  PCA READY     │                  │
│                         │  (Optional)    │                  │
│                         └────────────────┘                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Cấp Độ Beginner (Tuần 1-2)

**Thời gian**: 20-25 giờ  
**Mục tiêu**: Xây dựng nền tảng kiến thức về monitoring và Prometheus

### Tuần 1: Nền Tảng và Khái Niệm Cơ Bản

**Thời gian**: 10-12 giờ

#### Module 00: Getting Started (2 giờ)
- [ ] Đọc Introduction - Hiểu Prometheus là gì và tại sao cần học
- [ ] Đọc Learning Path Overview (tài liệu này)
- [ ] Đọc Prerequisites - Chuẩn bị môi trường và kiến thức

**Kết quả mong đợi**: Hiểu tổng quan về Prometheus và có môi trường sẵn sàng

#### Module 01: Fundamentals (8 giờ)
- [ ] Monitoring and Observability (2 giờ) - Khái niệm cơ bản về monitoring
- [ ] Time-Series Databases (1.5 giờ) - Hiểu cách lưu trữ time-series data
- [ ] Metrics Types (2 giờ) - Counter, Gauge, Histogram, Summary
- [ ] Prometheus Architecture (2 giờ) - Kiến trúc tổng thể
- [ ] Glossary (0.5 giờ) - Thuật ngữ chuyên ngành

**Kết quả mong đợi**: Hiểu rõ các khái niệm nền tảng và kiến trúc Prometheus

### Tuần 2: Thành Phần Cơ Bản và Thực Hành Đầu Tiên

**Thời gian**: 10-13 giờ

#### Module 02: Components - Phần Cơ Bản (6 giờ)
- [ ] Prometheus Server (2 giờ) - Thành phần core
- [ ] Exporters (2 giờ) - Cách thu thập metrics
- [ ] Storage and Retention (2 giờ) - Lưu trữ dữ liệu

**Kết quả mong đợi**: Hiểu các thành phần chính của Prometheus

#### Lab 01: Installation (4-7 giờ)
- [ ] Cài đặt Prometheus Server
- [ ] Cài đặt Node Exporter
- [ ] Cấu hình scrape targets
- [ ] Truy cập Prometheus UI
- [ ] Kiểm tra metrics đầu tiên

**Kết quả mong đợi**: Có Prometheus chạy thành công và thu thập metrics

**Checkpoint Beginner**:
- [ ] Giải thích được Prometheus là gì và tại sao dùng pull model
- [ ] Phân biệt được 4 loại metrics
- [ ] Cài đặt được Prometheus và node_exporter
- [ ] Xem được metrics trong Prometheus UI

## Cấp Độ Intermediate (Tuần 3-4)

**Thời gian**: 25-30 giờ  
**Mục tiêu**: Thành thạo PromQL và instrumentation

### Tuần 3: PromQL và Query Mastery

**Thời gian**: 12-15 giờ

#### Module 03: PromQL (10 giờ)
- [ ] PromQL Basics (2 giờ) - Cú pháp và selectors
- [ ] Data Types (1.5 giờ) - Scalar, instant vector, range vector
- [ ] Operators (2 giờ) - Arithmetic, comparison, logical
- [ ] Functions (2.5 giờ) - rate(), increase(), sum(), avg(), etc.
- [ ] Aggregations (1.5 giờ) - by, without, topk, bottomk
- [ ] Advanced Queries (0.5 giờ) - Subqueries, recording rules

**Kết quả mong đợi**: Viết được PromQL queries từ cơ bản đến nâng cao

#### Lab 03: PromQL Queries (2-5 giờ)
- [ ] Thực hành basic selectors
- [ ] Tính toán rates và increases
- [ ] Sử dụng aggregation operators
- [ ] Viết queries phức tạp

**Kết quả mong đợi**: Tự tin viết queries để phân tích metrics

### Tuần 4: Instrumentation và Custom Metrics

**Thời gian**: 13-15 giờ

#### Module 04: Instrumentation (8 giờ)
- [ ] Client Libraries (2 giờ) - Go, Python, Java
- [ ] Custom Metrics (3 giờ) - Tạo metrics cho application
- [ ] Common Exporters (2 giờ) - Danh sách exporters phổ biến
- [ ] Best Practices (1 giờ) - Naming conventions, cardinality

**Kết quả mong đợi**: Tạo được custom metrics cho application

#### Module 02: Components - Phần Nâng Cao (3 giờ)
- [ ] Alertmanager (1 giờ) - Giới thiệu alerting
- [ ] Pushgateway (1 giờ) - Short-lived jobs
- [ ] Service Discovery (1 giờ) - Tự động phát hiện targets

**Kết quả mong đợi**: Hiểu đầy đủ các thành phần Prometheus

#### Lab 02: Custom Metrics (2-4 giờ)
- [ ] Instrument application với client library
- [ ] Expose /metrics endpoint
- [ ] Configure Prometheus scrape
- [ ] Query custom metrics

**Kết quả mong đợi**: Application của bạn expose metrics cho Prometheus

#### Lab 05: Exporters (2-3 giờ)
- [ ] Cài đặt multiple exporters
- [ ] Configure scrape configs
- [ ] Verify metrics collection

**Kết quả mong đợi**: Tích hợp được nhiều exporters

**Checkpoint Intermediate**:
- [ ] Viết được PromQL queries phức tạp với aggregations
- [ ] Tạo được custom metrics cho application
- [ ] Hiểu và sử dụng được các exporters phổ biến
- [ ] Giải thích được service discovery

## Cấp Độ Advanced (Tuần 5-6)

**Thời gian**: 25-30 giờ  
**Mục tiêu**: Alerting, integrations, và production readiness

### Tuần 5: Alerting và Integrations

**Thời gian**: 13-15 giờ

#### Module 05: Alerting (8 giờ)
- [ ] Alerting Rules (2 giờ) - Viết alerting rules
- [ ] Alertmanager Config (2 giờ) - Routing, grouping, inhibition
- [ ] Notification Channels (2 giờ) - Email, Slack, PagerDuty
- [ ] Grafana Integration (2 giờ) - Dashboards và visualization

**Kết quả mong đợi**: Cấu hình được alerting system hoàn chỉnh

#### Lab 04: Alertmanager (3-5 giờ)
- [ ] Viết alerting rules
- [ ] Cấu hình Alertmanager
- [ ] Setup notification channels
- [ ] Test alerts

**Kết quả mong đợi**: Nhận được alerts qua Slack/Email

#### Module 06: Integrations - Phần 1 (2-2 giờ)
- [ ] Grafana Integration (chi tiết)
- [ ] Kubernetes Integration (giới thiệu)

**Kết quả mong đợi**: Tạo được Grafana dashboards

### Tuần 6: Production và Advanced Topics

**Thời gian**: 12-15 giờ

#### Module 06: Integrations - Phần 2 (8 giờ)
- [ ] Kubernetes Integration (3 giờ) - Service discovery, kube-state-metrics
- [ ] Docker Monitoring (2 giờ) - cAdvisor, container metrics
- [ ] Cloud Providers (2 giờ) - AWS, GCP, Azure
- [ ] Monitoring Stacks (1 giờ) - Complete stack examples

**Kết quả mong đợi**: Triển khai Prometheus trong môi trường production

#### Lab 06: Service Discovery (2-4 giờ)
- [ ] Setup file-based service discovery
- [ ] Configure Kubernetes service discovery (nếu có K8s)
- [ ] Test automatic target discovery

**Kết quả mong đợi**: Prometheus tự động phát hiện targets

#### Module 09: Reference (2 giờ)
- [ ] PromQL Quick Reference - Cheat sheet
- [ ] Configuration Reference - Parameters
- [ ] Troubleshooting Guide - Common issues
- [ ] Best Practices Checklist

**Kết quả mong đợi**: Có tài liệu tham khảo nhanh khi cần

**Checkpoint Advanced**:
- [ ] Cấu hình được alerting với routing phức tạp
- [ ] Tích hợp Prometheus với Grafana
- [ ] Triển khai Prometheus trong Kubernetes
- [ ] Hiểu best practices cho production

## Tài Liệu Tham Khảo (Liên Tục)

**Module 10: Resources** - Sử dụng khi cần

- Official Documentation - Links tài liệu chính thức
- Community Resources - Blogs, forums, Slack
- Books and Courses - Tài liệu học tập bổ sung
- Advanced Topics - Federation, remote storage, HA

**Cách sử dụng**: Tham khảo khi cần tìm hiểu sâu hơn về một chủ đề cụ thể

## Lộ Trình Chi Tiết Theo Module

### Sơ Đồ Dependencies

```
00-getting-started (Required first)
        │
        ▼
01-fundamentals (Required second)
        │
        ├──────────────┬──────────────┐
        ▼              ▼              ▼
02-components    03-promql    04-instrumentation
        │              │              │
        └──────┬───────┴──────┬───────┘
               ▼              ▼
        05-alerting    06-integrations
               │              │
               └──────┬───────┘
                      ▼
              07-labs (throughout)
                      │
                      ▼
          08-pca-preparation (optional)
                      │
                      ▼
        09-reference + 10-resources (as needed)
```

### Bảng Thời Gian Chi Tiết

| Module | Tên | Cấp Độ | Thời Gian | Tuần Đề Xuất |
|--------|-----|---------|-----------|--------------|
| 00 | Getting Started | Beginner | 2h | Tuần 1 |
| 01 | Fundamentals | Beginner | 8h | Tuần 1 |
| 02 | Components | Intermediate | 12h | Tuần 2, 4 |
| 03 | PromQL | Intermediate | 10h | Tuần 3 |
| 04 | Instrumentation | Intermediate | 8h | Tuần 4 |
| 05 | Alerting | Advanced | 8h | Tuần 5 |
| 06 | Integrations | Advanced | 10h | Tuần 5-6 |
| 07 | Labs | All levels | 15h | Throughout |
| 08 | PCA Preparation | Advanced | 20h | After completion |
| 09 | Reference | All levels | As needed | Throughout |
| 10 | Resources | All levels | As needed | Throughout |

### Chi Tiết Labs

| Lab | Tên | Prerequisites | Thời Gian | Tuần Đề Xuất |
|-----|-----|---------------|-----------|--------------|
| 01 | Installation | Module 01, 02 (basic) | 4-7h | Tuần 2 |
| 02 | Custom Metrics | Module 04 | 2-4h | Tuần 4 |
| 03 | PromQL Queries | Module 03 | 2-5h | Tuần 3 |
| 04 | Alertmanager | Module 05 | 3-5h | Tuần 5 |
| 05 | Exporters | Module 02, 04 | 2-3h | Tuần 4 |
| 06 | Service Discovery | Module 02 | 2-4h | Tuần 6 |

## Hướng Dẫn Sử Dụng Lộ Trình

### Lộ Trình Tiêu Chuẩn (6-8 tuần)

**Dành cho**: Người có thể dành 10-12 giờ/tuần

1. **Tuần 1**: Module 00, 01 - Nền tảng
2. **Tuần 2**: Module 02 (basic), Lab 01 - Thực hành đầu tiên
3. **Tuần 3**: Module 03, Lab 03 - PromQL mastery
4. **Tuần 4**: Module 04, 02 (advanced), Lab 02, 05 - Instrumentation
5. **Tuần 5**: Module 05, Lab 04, 06 (part 1) - Alerting
6. **Tuần 6**: Module 06 (part 2), Lab 06 - Integrations
7. **Tuần 7-8** (optional): Module 08 - PCA preparation

### Lộ Trình Nhanh (3-4 tuần)

**Dành cho**: Người có kinh nghiệm monitoring, có thể dành 20+ giờ/tuần

1. **Tuần 1**: Module 00, 01, 02, Lab 01
2. **Tuần 2**: Module 03, 04, Lab 02, 03, 05
3. **Tuần 3**: Module 05, 06, Lab 04, 06
4. **Tuần 4**: Review, practice, Module 08 (nếu cần PCA)

### Lộ Trình Linh Hoạt (Self-paced)

**Dành cho**: Người học theo tốc độ riêng

**Quy tắc**:
1. Luôn bắt đầu với Module 00 và 01
2. Hoàn thành Lab 01 trước khi tiếp tục
3. Module 03 (PromQL) là quan trọng nhất - dành thời gian đầy đủ
4. Labs nên làm ngay sau khi học module liên quan
5. Module 08 (PCA) chỉ cần nếu muốn lấy chứng chỉ

### Tips Học Tập Hiệu Quả

#### 1. Thực Hành Là Quan Trọng Nhất

```
Tỷ lệ đề xuất:
40% đọc lý thuyết
60% thực hành hands-on
```

- Đừng chỉ đọc, hãy làm theo từng bước
- Mỗi concept học được, hãy thử ngay trong lab
- Tạo môi trường test riêng để thử nghiệm

#### 2. Sử Dụng Checkpoints

Sau mỗi cấp độ, tự kiểm tra:
- [ ] Tôi có thể giải thích concept này cho người khác không?
- [ ] Tôi có thể làm lại lab mà không cần xem hướng dẫn không?
- [ ] Tôi có thể áp dụng kiến thức này vào project thực tế không?

#### 3. Ghi Chú và Tài Liệu Cá Nhân

Tạo một notebook riêng ghi lại:
- Các lệnh thường dùng
- PromQL queries hữu ích
- Troubleshooting tips từ kinh nghiệm
- Configuration snippets

#### 4. Tham Gia Community

- Join CNCF Slack #prometheus channel
- Đọc Prometheus blog và case studies
- Theo dõi GitHub issues để học từ vấn đề thực tế
- Chia sẻ kiến thức với người khác

#### 5. Build Real Projects

Sau khi hoàn thành cấp độ Intermediate, hãy:
- Monitor một application thực tế của bạn
- Setup Prometheus cho side project
- Tạo custom exporter cho một service
- Build Grafana dashboard cho use case cụ thể

### Xử Lý Khi Gặp Khó Khăn

#### Nếu Concept Quá Khó

1. Quay lại module trước đó để review
2. Đọc tài liệu tham khảo bổ sung trong Module 10
3. Tìm video tutorials hoặc blog posts giải thích khác
4. Hỏi trong community (Slack, Reddit, Stack Overflow)

#### Nếu Lab Không Chạy

1. Kiểm tra lại từng bước trong instructions
2. Xem phần Troubleshooting trong lab
3. Kiểm tra logs của Prometheus/Exporters
4. Tham khảo Module 09 - Troubleshooting Guide

#### Nếu Thiếu Thời Gian

1. Ưu tiên: Module 00, 01, 03, Lab 01, 03
2. Module 03 (PromQL) là quan trọng nhất - không nên skip
3. Có thể học Module 06 (Integrations) sau khi đã làm việc với Prometheus
4. Module 08 (PCA) chỉ cần nếu muốn certification

## Chuẩn Bị Cho Chứng Chỉ PCA

**Module 08: PCA Preparation** (20 giờ)

### Khi Nào Nên Bắt Đầu PCA Prep?

Bạn nên hoàn thành:
- [ ] Tất cả modules từ 00-06
- [ ] Tất cả 6 labs
- [ ] Có ít nhất 2-3 tuần kinh nghiệm thực tế với Prometheus

### Lộ Trình PCA (4-6 tuần)

#### Tuần 1-2: Review và Củng Cố (8-10 giờ)
- [ ] Exam Overview - Hiểu cấu trúc kỳ thi
- [ ] Study Schedule - Lập kế hoạch học
- [ ] Review tất cả modules đã học
- [ ] Làm lại các labs quan trọng

#### Tuần 3-4: Study Exam Domains (8-10 giờ)
- [ ] Domain 1: Observability Concepts (18%)
- [ ] Domain 2: Prometheus Fundamentals (20%)
- [ ] Domain 3: PromQL (28%) - Quan trọng nhất!
- [ ] Domain 4: Instrumentation & Exporters (20%)
- [ ] Domain 5: Alerting & Dashboarding (14%)

#### Tuần 5-6: Practice và Mock Exams (4-6 giờ)
- [ ] Practice Questions - Làm tất cả câu hỏi thực hành
- [ ] Exam Tips - Đọc chiến lược làm bài
- [ ] Mock exams (nếu có)
- [ ] Review các chủ đề còn yếu

### PCA Exam Domains Breakdown

| Domain | Weight | Focus Areas |
|--------|--------|-------------|
| Observability Concepts | 18% | Metrics, logs, traces; Push vs Pull; SLIs/SLOs |
| Prometheus Fundamentals | 20% | Architecture, components, service discovery |
| PromQL | 28% | Selectors, operators, functions, aggregations |
| Instrumentation & Exporters | 20% | Client libraries, metric naming, exporters |
| Alerting & Dashboarding | 14% | Alert rules, Alertmanager, Grafana |

**Lưu ý**: PromQL chiếm 28% - dành nhiều thời gian nhất cho domain này!

### Tài Nguyên PCA

- [Linux Foundation PCA Page](https://training.linuxfoundation.org/certification/prometheus-certified-associate/) - Thông tin chính thức
- Module 08 - PCA Preparation - Tài liệu chi tiết
- Module 09 - Reference - Quick reference cho exam
- Practice questions trong Module 08

## Tài Liệu Liên Quan

- [Introduction](./01-introduction.md) - Giới thiệu về Prometheus
- [Prerequisites](./03-prerequisites.md) - Chuẩn bị trước khi bắt đầu
- [Fundamentals Module](../01-fundamentals/README.md) - Bắt đầu học
- [Labs Overview](../07-labs/README.md) - Danh sách tất cả labs
- [PCA Preparation](../08-pca-preparation/README.md) - Chuẩn bị chứng chỉ

## Tài Liệu Tham Khảo

- [Prometheus Documentation](https://prometheus.io/docs/) - Tài liệu chính thức
- [CNCF Prometheus](https://www.cncf.io/projects/prometheus/) - Thông tin dự án
- [Prometheus GitHub](https://github.com/prometheus/prometheus) - Source code và issues
