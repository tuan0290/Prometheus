# Client Libraries

## Mục Lục
- [Giới Thiệu](#giới-thiệu)
- [Prometheus Client Libraries](#prometheus-client-libraries)
- [Go Client Library](#go-client-library)
- [Python Client Library](#python-client-library)
- [Java Client Library](#java-client-library)
- [Các Ngôn Ngữ Khác](#các-ngôn-ngữ-khác)
- [Tài Liệu Liên Quan](#tài-liệu-liên-quan)
- [Tài Liệu Tham Khảo](#tài-liệu-tham-khảo)

## Giới Thiệu

Prometheus client libraries cho phép bạn instrument application code để expose metrics. Các thư viện này cung cấp API để tạo và quản lý các loại metrics (counter, gauge, histogram, summary) và tự động expose chúng qua HTTP endpoint.

**Lợi ích của việc sử dụng client libraries:**
- API chuẩn hóa cho tất cả metric types
- Tự động xử lý thread-safety và concurrency
- Built-in HTTP server để expose metrics
- Hỗ trợ labels và metric naming conventions
- Tích hợp dễ dàng với Prometheus ecosystem

## Prometheus Client Libraries

Prometheus cung cấp official client libraries cho nhiều ngôn ngữ lập trình phổ biến:

| Ngôn Ngữ | Library | Repository |
|----------|---------|------------|
| Go | `prometheus/client_golang` | github.com/prometheus/client_golang |
| Python | `prometheus_client` | github.com/prometheus/client_python |
| Java | `simpleclient` | github.com/prometheus/client_java |
| Ruby | `prometheus-client` | github.com/prometheus/client_ruby |

**Unofficial client libraries** cũng có sẵn cho: Node.js, PHP, Rust, C++, .NET, và nhiều ngôn ngữ khác.

## Go Client Library

### Cài Đặt

```bash
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promhttp
```

### Counter Example

Counter là metric tăng đơn điệu, thường dùng để đếm requests, errors, tasks completed.

```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
)

func init() {
    // Register metrics with Prometheus
    prometheus.MustRegister(httpRequestsTotal)
}

func handler(w http.ResponseWriter, r *http.Request) {
    // Increment counter with labels
    httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, "200").Inc()
    w.Write([]byte("Hello, World!"))
}

func main() {
    http.HandleFunc("/", handler)
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

### Gauge Example

Gauge là metric có thể tăng hoặc giảm, thường dùng cho temperature, memory usage, concurrent requests.

```go
var (
    activeConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )
)

func init() {
    prometheus.MustRegister(activeConnections)
}

// Increase gauge
func onConnect() {
    activeConnections.Inc()
}

// Decrease gauge
func onDisconnect() {
    activeConnections.Dec()
}

// Set to specific value
func updateConnections(count float64) {
    activeConnections.Set(count)
}
```

### Histogram Example

Histogram đo phân phối của observations (request duration, response size).

```go
var (
    httpDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets, // [0.005, 0.01, 0.025, 0.05, ...]
        },
        []string{"method", "endpoint"},
    )
)

func init() {
    prometheus.MustRegister(httpDuration)
}

func timedHandler(w http.ResponseWriter, r *http.Request) {
    timer := prometheus.NewTimer(httpDuration.WithLabelValues(r.Method, r.URL.Path))
    defer timer.ObserveDuration()
    
    // Your handler logic here
    processRequest(r)
    w.WriteHeader(http.StatusOK)
}
```

## Python Client Library

### Cài Đặt

```bash
pip install prometheus-client
```

### Counter Example

```python
from prometheus_client import Counter, start_http_server
import time

# Create counter metric
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

def handle_request(method, endpoint):
    # Increment counter with labels
    http_requests_total.labels(
        method=method,
        endpoint=endpoint,
        status='200'
    ).inc()

if __name__ == '__main__':
    # Start HTTP server to expose metrics on port 8000
    start_http_server(8000)
    
    # Simulate requests
    while True:
        handle_request('GET', '/api/users')
        time.sleep(1)
```

### Gauge Example

```python
from prometheus_client import Gauge

# Create gauge metric
active_connections = Gauge(
    'active_connections',
    'Number of active connections'
)

# Increase gauge
def on_connect():
    active_connections.inc()

# Decrease gauge
def on_disconnect():
    active_connections.dec()

# Set to specific value
def update_connections(count):
    active_connections.set(count)

# Use as decorator to track in-progress requests
in_progress = Gauge('requests_in_progress', 'Requests in progress')

@in_progress.track_inprogress()
def process_request():
    # Your processing logic
    pass
```

### Histogram Example

```python
from prometheus_client import Histogram
import time

# Create histogram metric
http_request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['method', 'endpoint']
)

def handle_request(method, endpoint):
    # Measure duration
    with http_request_duration.labels(method=method, endpoint=endpoint).time():
        # Your request processing logic
        time.sleep(0.1)  # Simulate work
```

### Summary Example

```python
from prometheus_client import Summary

# Create summary metric
request_latency = Summary(
    'request_latency_seconds',
    'Request latency in seconds',
    ['service']
)

def call_service(service_name):
    with request_latency.labels(service=service_name).time():
        # Call external service
        pass
```

### Flask Integration

```python
from flask import Flask
from prometheus_client import Counter, Histogram, make_wsgi_app
from werkzeug.middleware.dispatcher import DispatcherMiddleware
import time

app = Flask(__name__)

# Metrics
request_count = Counter(
    'flask_request_count',
    'Flask Request Count',
    ['method', 'endpoint', 'http_status']
)

request_duration = Histogram(
    'flask_request_duration_seconds',
    'Flask Request Duration',
    ['method', 'endpoint']
)

@app.route('/')
def hello():
    start = time.time()
    
    # Your logic here
    result = "Hello, World!"
    
    # Record metrics
    duration = time.time() - start
    request_duration.labels(method='GET', endpoint='/').observe(duration)
    request_count.labels(method='GET', endpoint='/', http_status=200).inc()
    
    return result

# Add prometheus wsgi middleware to route /metrics requests
app.wsgi_app = DispatcherMiddleware(app.wsgi_app, {
    '/metrics': make_wsgi_app()
})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

## Java Client Library

### Cài Đặt (Maven)

```xml
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient</artifactId>
    <version>0.16.0</version>
</dependency>
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_httpserver</artifactId>
    <version>0.16.0</version>
</dependency>
```

### Counter Example

```java
import io.prometheus.client.Counter;
import io.prometheus.client.exporter.HTTPServer;
import java.io.IOException;

public class PrometheusExample {
    // Create counter metric
    static final Counter requests = Counter.build()
        .name("http_requests_total")
        .help("Total HTTP requests")
        .labelNames("method", "endpoint", "status")
        .register();
    
    public static void handleRequest(String method, String endpoint) {
        // Increment counter with labels
        requests.labels(method, endpoint, "200").inc();
    }
    
    public static void main(String[] args) throws IOException {
        // Start HTTP server on port 8080
        HTTPServer server = new HTTPServer(8080);
        
        // Simulate requests
        while (true) {
            handleRequest("GET", "/api/users");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

### Gauge Example

```java
import io.prometheus.client.Gauge;

public class GaugeExample {
    static final Gauge activeConnections = Gauge.build()
        .name("active_connections")
        .help("Number of active connections")
        .register();
    
    public void onConnect() {
        activeConnections.inc();
    }
    
    public void onDisconnect() {
        activeConnections.dec();
    }
    
    public void updateConnections(double count) {
        activeConnections.set(count);
    }
}
```

### Histogram Example

```java
import io.prometheus.client.Histogram;

public class HistogramExample {
    static final Histogram requestDuration = Histogram.build()
        .name("http_request_duration_seconds")
        .help("HTTP request duration in seconds")
        .labelNames("method", "endpoint")
        .buckets(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10)
        .register();
    
    public void handleRequest(String method, String endpoint) {
        Histogram.Timer timer = requestDuration
            .labels(method, endpoint)
            .startTimer();
        
        try {
            // Your request processing logic
            processRequest();
        } finally {
            timer.observeDuration();
        }
    }
    
    private void processRequest() {
        // Processing logic
    }
}
```

### Spring Boot Integration

```java
import io.prometheus.client.spring.boot.SpringBootMetricsCollector;
import io.prometheus.client.exporter.MetricsServlet;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class Application {
    
    @Bean
    public ServletRegistrationBean<MetricsServlet> metricsServlet() {
        return new ServletRegistrationBean<>(new MetricsServlet(), "/metrics");
    }
    
    @Bean
    public SpringBootMetricsCollector springBootMetricsCollector() {
        return new SpringBootMetricsCollector().register();
    }
    
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## Các Ngôn Ngữ Khác

### Node.js (prom-client)

```bash
npm install prom-client
```

```javascript
const client = require('prom-client');
const express = require('express');

// Create counter
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status']
});

// Create histogram
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route']
});

const app = express();

app.get('/', (req, res) => {
  const end = httpRequestDuration.startTimer();
  
  // Your logic
  res.send('Hello World');
  
  end({ method: req.method, route: req.route.path });
  httpRequestsTotal.inc({ method: req.method, route: req.route.path, status: 200 });
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

app.listen(3000);
```

### Ruby (prometheus-client)

```bash
gem install prometheus-client
```

```ruby
require 'prometheus/client'
require 'rack'

# Create registry
prometheus = Prometheus::Client.registry

# Create counter
http_requests = Prometheus::Client::Counter.new(
  :http_requests_total,
  docstring: 'Total HTTP requests',
  labels: [:method, :path]
)
prometheus.register(http_requests)

# Increment counter
http_requests.increment(labels: { method: 'GET', path: '/' })

# Expose metrics
use Prometheus::Client::Rack::Exporter
```

## Tài Liệu Liên Quan

- [Custom Metrics](./02-custom-metrics.md) - Tạo custom metrics
- [Best Practices](./04-best-practices.md) - Naming conventions và cardinality
- [Metrics Types](../01-fundamentals/03-metrics-types.md) - Các loại metrics
- [Lab 02 - Custom Metrics](../07-labs/lab-02-custom-metrics/README.md) - Bài tập thực hành

## Tài Liệu Tham Khảo

- [Prometheus Client Libraries](https://prometheus.io/docs/instrumenting/clientlibs/) - Official documentation
- [Writing Client Libraries](https://prometheus.io/docs/instrumenting/writing_clientlibs/) - Guidelines for library authors
- [Go Client GitHub](https://github.com/prometheus/client_golang) - Go client repository
- [Python Client GitHub](https://github.com/prometheus/client_python) - Python client repository
- [Java Client GitHub](https://github.com/prometheus/client_java) - Java client repository
