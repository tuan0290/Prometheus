# Yêu Cầu Tiên Quyết

## Mục Lục

- [Giới Thiệu](#giới-thiệu)
- [Kiến Thức Nền Tảng](#kiến-thức-nền-tảng)
- [Công Cụ và Môi Trường](#công-cụ-và-môi-trường)
- [Môi Trường Thực Hành](#môi-trường-thực-hành)
- [Kiểm Tra Sự Sẵn Sàng](#kiểm-tra-sự-sẵn-sàng)
- [Tài Nguyên Học Tập Bổ Sung](#tài-nguyên-học-tập-bổ-sung)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)

## Giới Thiệu

Tài liệu này liệt kê các kiến thức nền tảng và công cụ cần thiết để bắt đầu học Prometheus một cách hiệu quả. Mặc dù Prometheus được thiết kế để dễ tiếp cận, việc có một số kiến thức cơ bản sẽ giúp bạn học nhanh hơn và hiểu sâu hơn.

**Lưu ý quan trọng**: Bạn không cần phải là chuyên gia về tất cả các chủ đề dưới đây. Tài liệu này chỉ ra những gì sẽ hữu ích, nhưng bạn có thể học Prometheus và bổ sung kiến thức thiếu trong quá trình học.

## Kiến Thức Nền Tảng

### 1. Linux Command Line (Bắt Buộc)

**Mức độ cần thiết**: Cơ bản đến Trung bình

**Tại sao cần**: Prometheus chủ yếu chạy trên Linux, và bạn sẽ cần sử dụng command line để cài đặt, cấu hình, và troubleshoot.

**Kiến thức cần có**:
- Điều hướng file system: `cd`, `ls`, `pwd`
- Quản lý files: `cat`, `less`, `nano`, `vim`
- Permissions: `chmod`, `chown`
- Processes: `ps`, `top`, `kill`
- Networking: `curl`, `wget`, `netstat`, `ss`
- System services: `systemctl` (cho systemd)

**Ví dụ thực tế**:
```bash
# Kiểm tra Prometheus có đang chạy không
ps aux | grep prometheus

# Xem metrics endpoint
curl http://localhost:9090/metrics

# Kiểm tra port đang listen
ss -tulpn | grep 9090
```

**Nếu chưa biết**: Dành 1-2 tuần học Linux basics trước khi bắt đầu Prometheus.

**Tài nguyên học**:
- [Linux Journey](https://linuxjourney.com/) - Interactive tutorials
- [The Linux Command Line book](http://linuxcommand.org/tlcl.php) - Free ebook

### 2. Networking Basics (Bắt Buộc)

**Mức độ cần thiết**: Cơ bản

**Tại sao cần**: Prometheus pull metrics qua HTTP, bạn cần hiểu networking để configure và troubleshoot.

**Kiến thức cần có**:
- HTTP protocol basics (GET requests, status codes)
- IP addresses và ports
- DNS basics
- Localhost và 127.0.0.1
- Firewalls và port forwarding (cơ bản)

**Ví dụ thực tế**:
```
Prometheus scrapes metrics từ:
http://192.168.1.10:9100/metrics
│      │            │     │
│      │            │     └─ Path
│      │            └─ Port (node_exporter default)
│      └─ IP address của target
└─ Protocol
```

**Nếu chưa biết**: Đọc về HTTP và networking basics, mất khoảng 2-3 ngày.

### 3. YAML Syntax (Bắt Buộc)

**Mức độ cần thiết**: Cơ bản

**Tại sao cần**: Prometheus configuration files sử dụng YAML format.

**Kiến thức cần có**:
- YAML syntax basics
- Indentation (spaces, not tabs!)
- Lists và dictionaries
- Comments

**Ví dụ Prometheus config**:
```yaml
# Prometheus configuration example
global:
  scrape_interval: 15s  # Indent với 2 spaces
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'  # List item bắt đầu với -
    static_configs:
      - targets: ['localhost:9090']  # Nested structure
```

**Nếu chưa biết**: YAML rất đơn giản, học trong 1-2 giờ là đủ.

**Tài nguyên học**:
- [Learn YAML in Y minutes](https://learnxinyminutes.com/docs/yaml/)
- [YAML Tutorial](https://www.cloudbees.com/blog/yaml-tutorial-everything-you-need-get-started)

### 4. Basic Programming/Scripting (Khuyến Nghị)

**Mức độ cần thiết**: Cơ bản (một ngôn ngữ bất kỳ)

**Tại sao cần**: Để tạo custom metrics và hiểu instrumentation examples.

**Ngôn ngữ đề xuất** (chọn một):
- **Python**: Dễ học, client library tốt
- **Go**: Ngôn ngữ Prometheus được viết bằng
- **Java**: Nếu bạn làm việc với Java applications
- **JavaScript/Node.js**: Cho web applications

**Kiến thức cần có**:
- Variables và data types
- Functions
- HTTP requests (làm sao gọi API)
- Basic error handling

**Ví dụ Python instrumentation**:
```python
from prometheus_client import Counter, start_http_server

# Tạo một counter metric
requests_total = Counter('http_requests_total', 'Total HTTP requests')

# Increment counter
requests_total.inc()

# Expose metrics
start_http_server(8000)
```

**Nếu chưa biết**: Không bắt buộc cho beginner level, nhưng cần cho Module 04 (Instrumentation).

### 5. Docker Basics (Khuyến Nghị Mạnh)

**Mức độ cần thiết**: Cơ bản

**Tại sao cần**: Cách dễ nhất để chạy Prometheus và các components. Nhiều production deployments dùng containers.

**Kiến thức cần có**:
- Docker là gì
- Chạy containers: `docker run`
- Quản lý containers: `docker ps`, `docker stop`, `docker logs`
- Docker Compose basics (bonus)
- Port mapping: `-p 9090:9090`
- Volume mounting: `-v`

**Ví dụ chạy Prometheus với Docker**:
```bash
# Chạy Prometheus container
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus

# Xem logs
docker logs prometheus

# Stop container
docker stop prometheus
```

**Nếu chưa biết**: Dành 2-3 ngày học Docker basics trước Lab 01.

**Tài nguyên học**:
- [Docker Getting Started](https://docs.docker.com/get-started/)
- [Play with Docker](https://labs.play-with-docker.com/) - Free online playground

### 6. Kubernetes Basics (Tùy Chọn)

**Mức độ cần thiết**: Không bắt buộc (chỉ cần cho Module 06)

**Tại sao cần**: Prometheus thường được dùng để monitor Kubernetes clusters.

**Kiến thức cần có**:
- Kubernetes architecture basics
- Pods, Services, Deployments
- kubectl commands
- YAML manifests

**Khi nào cần**: Chỉ cần khi học Module 06 (Integrations) - Kubernetes Integration.

**Nếu chưa biết**: Có thể skip phần Kubernetes trong Module 06, hoặc học K8s basics sau.

### 7. Monitoring Concepts (Sẽ Học Trong Khóa)

**Mức độ cần thiết**: Không cần biết trước

**Tại sao không cần**: Module 01 (Fundamentals) sẽ dạy tất cả monitoring concepts từ đầu.

**Nếu đã biết**: Bạn sẽ học nhanh hơn, nhưng không bắt buộc.

## Công Cụ và Môi Trường

### 1. Operating System

**Khuyến nghị**: Linux (Ubuntu, Debian, CentOS, hoặc Fedora)

**Lý do**:
- Prometheus được thiết kế cho Linux
- Hầu hết production deployments chạy trên Linux
- Tài liệu và examples chủ yếu cho Linux

**Alternatives**:
- **macOS**: Hoàn toàn OK, hầu hết commands tương tự Linux
- **Windows**: Có thể dùng, nhưng khuyến nghị dùng WSL2 (Windows Subsystem for Linux)

**Setup cho Windows users**:
```bash
# Cài WSL2
wsl --install

# Hoặc dùng Docker Desktop for Windows
# Download từ: https://www.docker.com/products/docker-desktop
```

### 2. Text Editor hoặc IDE

**Bắt buộc**: Một text editor để edit configuration files

**Khuyến nghị**:
- **VS Code**: Free, nhiều extensions, YAML support tốt
- **Vim/Neovim**: Nếu bạn thích terminal-based
- **Sublime Text**: Nhẹ và nhanh
- **IntelliJ IDEA**: Nếu bạn đã dùng cho Java

**VS Code extensions hữu ích**:
- YAML (Red Hat)
- Prometheus (Prometheus extension)
- Docker
- Remote - SSH (để edit files trên remote servers)

### 3. Terminal Emulator

**Bắt buộc**: Để chạy commands

**Khuyến nghị**:
- **Linux**: Terminal mặc định là OK
- **macOS**: iTerm2 hoặc Terminal mặc định
- **Windows**: Windows Terminal, hoặc terminal trong WSL2

### 4. Docker và Docker Compose

**Khuyến nghị mạnh**: Cài đặt Docker

**Tại sao**:
- Cách dễ nhất để chạy Prometheus và exporters
- Không cần compile từ source
- Dễ cleanup và restart
- Giống production environment

**Cài đặt**:
```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Thêm user vào docker group (không cần sudo)
sudo usermod -aG docker $USER

# Verify installation
docker --version
docker-compose --version
```

**Alternatives**: Có thể cài Prometheus binary trực tiếp, nhưng phức tạp hơn.

### 5. Web Browser

**Bắt buộc**: Để truy cập Prometheus UI và Grafana

**Khuyến nghị**:
- Chrome hoặc Firefox (modern browsers)
- Developer tools enabled (F12)

### 6. curl hoặc wget

**Bắt buộc**: Để test HTTP endpoints

**Cài đặt**:
```bash
# Ubuntu/Debian
sudo apt-get install curl wget

# macOS (curl đã có sẵn)
brew install wget

# Verify
curl --version
```

### 7. Git (Tùy Chọn)

**Khuyến nghị**: Để clone examples và configuration repos

**Cài đặt**:
```bash
# Ubuntu/Debian
sudo apt-get install git

# macOS
brew install git

# Verify
git --version
```

## Môi Trường Thực Hành

### Option 1: Local Machine (Khuyến Nghị cho Beginners)

**Ưu điểm**:
- Không tốn tiền
- Không cần internet connection (sau khi setup)
- Học được cách setup từ đầu

**Yêu cầu tối thiểu**:
- **CPU**: 2 cores
- **RAM**: 4GB (8GB khuyến nghị)
- **Disk**: 20GB free space
- **OS**: Linux, macOS, hoặc Windows với WSL2

**Setup**:
```bash
# Kiểm tra resources
# CPU cores
nproc

# RAM
free -h

# Disk space
df -h
```

### Option 2: Cloud VM (Khuyến Nghị cho Advanced Labs)

**Ưu điểm**:
- Giống production environment
- Có thể access từ mọi nơi
- Dễ cleanup (xóa VM khi xong)

**Providers đề xuất**:
- **DigitalOcean**: $5/month droplet là đủ
- **AWS EC2**: Free tier (t2.micro) cho 12 tháng đầu
- **Google Cloud**: $300 free credits
- **Azure**: $200 free credits

**Yêu cầu VM**:
- **Instance type**: 2 vCPU, 4GB RAM
- **OS**: Ubuntu 22.04 LTS
- **Storage**: 20GB

### Option 3: Kubernetes Cluster (Chỉ Cho Module 06)

**Khi nào cần**: Chỉ cần cho Module 06 - Kubernetes Integration

**Options**:
- **Minikube**: Local Kubernetes cluster
- **Kind**: Kubernetes in Docker
- **k3s**: Lightweight Kubernetes
- **Cloud managed**: GKE, EKS, AKS (tốn tiền)

**Setup Minikube**:
```bash
# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start --cpus=2 --memory=4096

# Verify
kubectl get nodes
```

**Lưu ý**: Không cần setup ngay, chỉ cần khi đến Module 06.

### Option 4: Docker Compose Stack (Khuyến Nghị cho Labs)

**Ưu điểm**:
- Chạy nhiều components cùng lúc
- Dễ start/stop toàn bộ stack
- Configuration as code

**Ví dụ docker-compose.yml**:
```yaml
version: '3'
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
  
  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
  
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
```

## Kiểm Tra Sự Sẵn Sàng

### Checklist Kiến Thức

Đánh dấu những gì bạn đã biết:

**Bắt buộc**:
- [ ] Tôi có thể sử dụng Linux command line cơ bản
- [ ] Tôi hiểu HTTP và networking basics
- [ ] Tôi có thể đọc và edit YAML files
- [ ] Tôi biết cách sử dụng text editor

**Khuyến nghị mạnh**:
- [ ] Tôi có thể chạy Docker containers
- [ ] Tôi biết cách sử dụng curl để test APIs
- [ ] Tôi có kinh nghiệm với ít nhất một ngôn ngữ lập trình

**Nice to have**:
- [ ] Tôi có kinh nghiệm với Kubernetes
- [ ] Tôi đã làm việc với monitoring tools trước đây
- [ ] Tôi hiểu về microservices architecture

### Checklist Môi Trường

Đánh dấu những gì bạn đã cài đặt:

**Bắt buộc**:
- [ ] Linux/macOS/Windows với WSL2
- [ ] Text editor (VS Code, Vim, etc.)
- [ ] Terminal emulator
- [ ] Web browser (Chrome/Firefox)
- [ ] curl hoặc wget

**Khuyến nghị mạnh**:
- [ ] Docker và Docker Compose
- [ ] Git
- [ ] Ít nhất 4GB RAM available
- [ ] 20GB disk space free

**Tùy chọn**:
- [ ] Cloud VM account (DigitalOcean, AWS, etc.)
- [ ] Kubernetes cluster (Minikube, Kind, etc.)

### Test Commands

Chạy các commands sau để verify setup:

```bash
# 1. Check OS
uname -a

# 2. Check Docker
docker --version
docker-compose --version
docker run hello-world

# 3. Check curl
curl --version
curl https://prometheus.io

# 4. Check disk space
df -h

# 5. Check RAM
free -h

# 6. Check network
curl http://localhost:9090 || echo "Prometheus not running yet (OK)"

# 7. Check text editor
code --version  # VS Code
# hoặc
vim --version
```

**Kết quả mong đợi**: Tất cả commands chạy thành công (trừ Prometheus check - sẽ cài trong Lab 01).

## Tài Nguyên Học Tập Bổ Sung

### Nếu Thiếu Kiến Thức Linux

**Free courses**:
- [Linux Journey](https://linuxjourney.com/) - Interactive, beginner-friendly
- [The Linux Command Line](http://linuxcommand.org/tlcl.php) - Free ebook
- [OverTheWire Bandit](https://overthewire.org/wargames/bandit/) - Gamified learning

**Thời gian**: 1-2 tuần (2-3 giờ/ngày)

### Nếu Thiếu Kiến Thức Docker

**Free courses**:
- [Docker Getting Started](https://docs.docker.com/get-started/) - Official tutorial
- [Play with Docker](https://labs.play-with-docker.com/) - Online playground
- [Docker for Beginners](https://docker-curriculum.com/) - Comprehensive guide

**Thời gian**: 2-3 ngày (2-3 giờ/ngày)

### Nếu Thiếu Kiến Thức Networking

**Free resources**:
- [Computer Networking: A Top-Down Approach](https://gaia.cs.umass.edu/kurose_ross/online_lectures.htm) - Video lectures
- [HTTP Basics](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP) - MDN docs
- [Networking Fundamentals](https://www.youtube.com/watch?v=cNwEVYkx2Kk) - YouTube playlist

**Thời gian**: 3-5 ngày (1-2 giờ/ngày)

### Nếu Thiếu Kiến Thức Programming

**Python** (khuyến nghị cho beginners):
- [Python for Everybody](https://www.py4e.com/) - Free course
- [Automate the Boring Stuff](https://automatetheboringstuff.com/) - Practical Python

**Go** (nếu muốn hiểu Prometheus source code):
- [Tour of Go](https://go.dev/tour/) - Official interactive tutorial
- [Go by Example](https://gobyexample.com/) - Practical examples

**Thời gian**: 1-2 tuần (nếu chưa biết programming), hoặc 2-3 ngày (nếu đã biết ngôn ngữ khác)

## Tài Liệu Liên Quan

- [Introduction](./01-introduction.md) - Tìm hiểu về Prometheus
- [Learning Path Overview](./02-learning-path-overview.md) - Xem lộ trình học tập
- [Fundamentals Module](../01-fundamentals/README.md) - Bắt đầu học sau khi chuẩn bị xong
- [Lab 01: Installation](../07-labs/lab-01-installation/README.md) - Lab đầu tiên

## Tài Liệu Tham Khảo

- [Prometheus Documentation](https://prometheus.io/docs/) - Tài liệu chính thức
- [Docker Documentation](https://docs.docker.com/) - Hướng dẫn Docker
- [Linux Journey](https://linuxjourney.com/) - Học Linux miễn phí
- [YAML Tutorial](https://www.cloudbees.com/blog/yaml-tutorial-everything-you-need-get-started) - Học YAML
