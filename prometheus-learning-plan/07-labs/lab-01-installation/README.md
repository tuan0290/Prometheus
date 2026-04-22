# Lab 01 - Installation

## Tổng Quan

Lab này hướng dẫn bạn cài đặt Prometheus và node_exporter từ đầu. Bạn sẽ học cách download, configure, và chạy Prometheus server cùng với node_exporter để collect system metrics.

**Cấp độ**: Beginner  
**Thời gian ước tính**: 60 phút  
**Yêu cầu trước**: Hoàn thành Fundamentals module

## Mục Tiêu

Trong bài lab này, bạn sẽ:
- Download và cài đặt Prometheus server
- Cài đặt node_exporter để collect system metrics
- Cấu hình prometheus.yml với scrape targets
- Verify Prometheus đang collect metrics
- Truy cập Prometheus web UI

## Nội Dung

- [Instructions](./instructions.md) - Hướng dẫn từng bước
- [Verification](./verification.md) - Cách kiểm tra kết quả

## Kết Quả Mong Đợi

Sau khi hoàn thành lab này, bạn sẽ có:
- Prometheus server chạy trên port 9090
- Node_exporter chạy trên port 9100
- Prometheus đang scrape metrics từ node_exporter
- Có thể query metrics qua web UI

## Tiếp Theo

Sau khi hoàn thành lab này, tiếp tục với: [Lab 02 - Custom Metrics](../lab-02-custom-metrics/README.md)
