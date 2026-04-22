# Kế Hoạch Học Prometheus - Hướng Dẫn Toàn Diện

Chào mừng bạn đến với hệ thống kế hoạch học Prometheus! Đây là bộ tài liệu hướng dẫn toàn diện được thiết kế cho người mới bắt đầu chưa có kinh nghiệm với Prometheus. Hệ thống cung cấp lộ trình học tập có cấu trúc từ cơ bản đến nâng cao, bao gồm các thành phần chi tiết của Prometheus, bài tập thực hành, và chuẩn bị cho chứng chỉ Prometheus Certified Associate (PCA).

## Giới Thiệu

Prometheus là một hệ thống monitoring và alerting mã nguồn mở được thiết kế đặc biệt cho môi trường cloud-native và microservices. Với kiến trúc pull-based và ngôn ngữ truy vấn PromQL mạnh mẽ, Prometheus đã trở thành tiêu chuẩn de facto cho monitoring trong hệ sinh thái Kubernetes và CNCF.

## Cấu Trúc Hệ Thống

Hệ thống học tập được tổ chức thành 11 module chính:

### [00. Getting Started](./00-getting-started/README.md)
**Cấp độ**: Beginner | **Thời gian**: 2 giờ

Điểm khởi đầu cho người học mới, bao gồm giới thiệu về Prometheus, lộ trình học tập tổng quan, và các yêu cầu tiên quyết.

### [01. Fundamentals](./01-fundamentals/README.md)
**Cấp độ**: Beginner | **Thời gian**: 8 giờ

Xây dựng nền tảng kiến thức cơ bản về monitoring, observability, time-series databases, các loại metrics, kiến trúc Prometheus, và thuật ngữ chuyên ngành.

### [02. Components](./02-components/README.md)
**Cấp độ**: Intermediate | **Thời gian**: 12 giờ

Tìm hiểu chi tiết các thành phần của Prometheus: Server, Exporters, Alertmanager, Pushgateway, Service Discovery, và Storage.

### [03. PromQL](./03-promql/README.md)
**Cấp độ**: Intermediate | **Thời gian**: 10 giờ

Làm chủ ngôn ngữ truy vấn PromQL từ cơ bản đến nâng cao, bao gồm selectors, operators, functions, aggregations, và advanced queries.

### [04. Instrumentation](./04-instrumentation/README.md)
**Cấp độ**: Intermediate | **Thời gian**: 8 giờ

Học cách tạo custom metrics với client libraries, naming conventions, và best practices cho instrumentation.

### [05. Alerting](./05-alerting/README.md)
**Cấp độ**: Advanced | **Thời gian**: 8 giờ

Cấu hình alerting rules, Alertmanager, notification channels, và tích hợp với Grafana để tạo dashboards.

### [06. Integrations](./06-integrations/README.md)
**Cấp độ**: Advanced | **Thời gian**: 10 giờ

Tích hợp Prometheus với Kubernetes, Docker, cloud providers (AWS, GCP, Azure), và các monitoring stacks phổ biến.

### [07. Labs](./07-labs/README.md)
**Cấp độ**: All Levels | **Thời gian**: 15 giờ

Bài tập thực hành hands-on với 6 labs: Installation, Custom Metrics, PromQL Queries, Alertmanager, Exporters, và Service Discovery.

### [08. PCA Preparation](./08-pca-preparation/README.md)
**Cấp độ**: Advanced | **Thời gian**: 20 giờ

Chuẩn bị cho chứng chỉ Prometheus Certified Associate với nội dung theo exam domains, practice questions, study schedule, và exam tips.

### [09. Reference](./09-reference/README.md)
**Cấp độ**: All Levels | **Thời gian**: As needed

Tài liệu tham khảo nhanh bao gồm PromQL cheat sheet, configuration reference, exporters catalog, troubleshooting guide, và best practices checklist.

### [10. Resources](./10-resources/README.md)
**Cấp độ**: All Levels | **Thời gian**: As needed

Tài nguyên bổ sung với links đến official documentation, community resources, recommended books/courses, và advanced topics.

## Lộ Trình Học Tập Đề Xuất

### Beginner Level (Tuần 1-2)
1. Bắt đầu với [Getting Started](./00-getting-started/README.md)
2. Học [Fundamentals](./01-fundamentals/README.md)
3. Tìm hiểu cơ bản về [Components](./02-components/README.md)
4. Thực hành [Lab 01 - Installation](./07-labs/lab-01-installation/README.md)

### Intermediate Level (Tuần 3-4)
1. Hoàn thành [Components](./02-components/README.md) nâng cao
2. Làm chủ [PromQL](./03-promql/README.md)
3. Học [Instrumentation](./04-instrumentation/README.md)
4. Thực hành [Lab 02](./07-labs/lab-02-custom-metrics/README.md), [Lab 03](./07-labs/lab-03-promql-queries/README.md), [Lab 05](./07-labs/lab-05-exporters/README.md)

### Advanced Level (Tuần 5-6)
1. Học [Alerting](./05-alerting/README.md)
2. Tìm hiểu [Integrations](./06-integrations/README.md)
3. Thực hành [Lab 04](./07-labs/lab-04-alertmanager/README.md), [Lab 06](./07-labs/lab-06-service-discovery/README.md)
4. Bắt đầu [PCA Preparation](./08-pca-preparation/README.md)

### Reference (Ongoing)
- Sử dụng [Reference](./09-reference/README.md) khi cần tra cứu nhanh
- Khám phá [Resources](./10-resources/README.md) để mở rộng kiến thức

## Yêu Cầu Trước

- Kiến thức cơ bản về Linux/Unix command line
- Hiểu biết về networking cơ bản (HTTP, TCP/IP)
- Kinh nghiệm với Docker (khuyến nghị)
- Kinh nghiệm với YAML configuration files
- Text editor hoặc IDE (VS Code, Vim, etc.)

## Cách Sử Dụng Hệ Thống Này

1. **Học Tuần Tự**: Bắt đầu từ module 00 và tiến dần theo thứ tự
2. **Thực Hành Ngay**: Làm labs ngay sau khi học lý thuyết tương ứng
3. **Sử Dụng Checkpoints**: Đánh dấu checkpoints trong mỗi module để theo dõi tiến độ
4. **Tham Khảo Thường Xuyên**: Sử dụng module Reference khi cần tra cứu
5. **Mở Rộng Kiến Thức**: Khám phá Resources để học sâu hơn

## Thời Gian Học Ước Tính

**Tổng thời gian**: 60-80 giờ

- **Beginner Level**: 20-25 giờ
- **Intermediate Level**: 25-30 giờ
- **Advanced Level**: 15-20 giờ
- **PCA Preparation**: 20 giờ

## Mục Tiêu Học Tập

Sau khi hoàn thành hệ thống học này, bạn sẽ có thể:

- ✅ Hiểu kiến trúc và các thành phần của Prometheus
- ✅ Cài đặt và cấu hình Prometheus trong môi trường production
- ✅ Viết PromQL queries để truy vấn và phân tích metrics
- ✅ Tạo custom metrics và instrument applications
- ✅ Cấu hình alerting rules và notification channels
- ✅ Tích hợp Prometheus với Kubernetes, Docker, và cloud platforms
- ✅ Xây dựng dashboards với Grafana
- ✅ Sẵn sàng cho kỳ thi Prometheus Certified Associate (PCA)

## Đóng Góp và Phản Hồi

Hệ thống này được thiết kế để liên tục cải thiện. Nếu bạn phát hiện lỗi, có đề xuất cải tiến, hoặc muốn đóng góp nội dung, vui lòng tạo issue hoặc pull request.

## Giấy Phép

Tài liệu này được cung cấp cho mục đích học tập. Vui lòng tham khảo official Prometheus documentation cho thông tin chính thức và cập nhật nhất.

## Bắt Đầu Ngay

Sẵn sàng bắt đầu hành trình học Prometheus? Hãy bắt đầu với [Getting Started](./00-getting-started/README.md)!

---

**Phiên bản**: 1.0  
**Cập nhật lần cuối**: 2025  
**Trạng thái**: Active Learning System
