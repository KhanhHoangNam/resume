# 4. Giải thích từng khối trong sơ đồ

## 4.1 Bảng tổng hợp các khối chính

| Khối | Thành phần | Vai trò chính | Cách triển khai điển hình | Ghi chú |
|---|---|---|---|---|
| **Application Layer** | Java / Spring Boot services, API, Worker, Batch | Sinh ra telemetry (**trace / metric / log**) | Chạy trên **Kubernetes** hoặc **VM** | Đây là **nguồn phát dữ liệu observability** |
| **Instrumentation Layer** | OpenTelemetry SDK / Java Agent | Instrument ứng dụng để phát telemetry theo chuẩn OTel | Java Agent (`-javaagent`) hoặc SDK trong code | Giúp app **không phụ thuộc trực tiếp** vào backend observability |
| **Agent Layer** | OTel Agent Collector | Thu thập telemetry gần nguồn phát, enrich metadata cơ bản, forward lên Gateway | **DaemonSet** trên K8s, **systemd service** trên VM | Nên giữ **nhẹ**, không xử lý quá nặng |
| **Gateway Layer** | OTel Gateway Collector + Internal Load Balancer / Service | Xử lý telemetry tập trung: batch, retry, sampling, enrich, routing | **Deployment** trên K8s, scale nhiều replicas | Đây là **trung tâm xử lý pipeline** |
| **Backend Layer** | Tempo / Jaeger, Prometheus / Mimir, Loki / ELK, Grafana | Lưu trữ, truy vấn, hiển thị telemetry | Monitoring cluster / Platform cluster | Có thể thay backend mà **không cần sửa app** |

---

## 4.2 Chi tiết từng khối

### A. Application Layer

| Thuộc tính | Mô tả |
|---|---|
| **Vai trò** | Là nơi phát sinh request / transaction và tạo ra telemetry ban đầu |
| **Thành phần điển hình** | API service, microservice, worker, batch job, scheduler |
| **Telemetry sinh ra** | Trace spans, application metrics, application logs |
| **Mục tiêu** | Giúp theo dõi hành vi và hiệu năng của ứng dụng |
| **Ví dụ** | `billing-service`, `payment-service`, `notification-worker` |

> **Tóm gọn:** Đây là nơi **phát sinh dữ liệu quan sát** của hệ thống.

---

### B. Instrumentation Layer

| Thuộc tính | Mô tả |
|---|---|
| **Vai trò** | Gắn khả năng observability vào ứng dụng |
| **Thành phần chính** | OpenTelemetry SDK hoặc OpenTelemetry Java Agent |
| **Mục tiêu** | Tạo trace / metric / log theo chuẩn OpenTelemetry |
| **Cách dùng phổ biến** | Với Java nên ưu tiên **OpenTelemetry Java Agent** |
| **Lợi ích** | Không cần viết quá nhiều code custom, triển khai nhanh, dễ chuẩn hóa |

> **Tóm gọn:** Đây là lớp giúp ứng dụng **biết cách phát telemetry chuẩn OTel**.

---

### C. Agent Layer

| Thuộc tính | Mô tả |
|---|---|
| **Vai trò** | Thu telemetry gần nguồn phát và forward về Gateway |
| **Mục tiêu chính** | Giảm coupling giữa app và backend, giảm network hop |
| **Triển khai trên K8s** | **DaemonSet** – mỗi node có 1 Collector |
| **Triển khai trên VM** | Chạy như **systemd service** |
| **Nhiệm vụ chính** | Nhận OTLP, gắn metadata local (node / host / env), batch nhẹ, forward |
| **Nên tránh** | Không nên làm transform hoặc xử lý quá nặng ở layer này |

> **Tóm gọn:** Đây là lớp **collector gần ứng dụng nhất**.

---

### D. Gateway Layer

| Thuộc tính | Mô tả |
|---|---|
| **Vai trò** | Xử lý telemetry tập trung trước khi gửi tới backend |
| **Triển khai điển hình** | Chạy dạng **Deployment** trên K8s |
| **Khả năng HA** | Nên có **>= 2 replicas** + Load Balancer / K8s Service |
| **Tác vụ phù hợp** | Batch, retry, queue, enrich, sampling, routing |
| **Mục tiêu chính** | Tăng reliability, giảm tải backend, chuẩn hóa observability pipeline |
| **Lợi ích lớn nhất** | App / Agent không cần biết backend cuối là gì |

> **Tóm gọn:** Đây là **bộ não của observability pipeline**.

---

### E. Backend Layer

| Thuộc tính | Mô tả |
|---|---|
| **Vai trò** | Lưu trữ, truy vấn, hiển thị telemetry |
| **Thành phần phổ biến** | Tempo / Jaeger (Trace), Prometheus / Mimir (Metric), Loki / ELK (Log), Grafana (Dashboard) |
| **Mục tiêu** | Giúp operator / DevOps / developer quan sát hệ thống |
| **Tính linh hoạt** | Có thể đổi backend mà không ảnh hưởng ứng dụng nếu pipeline OTel được chuẩn hóa |
| **Use case** | Điều tra incident, phân tích latency, kiểm tra health, xây dashboard |

> **Tóm gọn:** Đây là lớp **đích đến cuối cùng** của telemetry.

---

## 4.3 Luồng tương tác giữa các khối

| Bước | Luồng dữ liệu | Ý nghĩa |
|---|---|---|
| **1** | Application → Instrumentation | Ứng dụng được gắn khả năng phát telemetry |
| **2** | Instrumentation → Agent | App gửi trace / metric / log tới collector gần nhất |
| **3** | Agent → Gateway | Agent forward telemetry về lớp xử lý tập trung |
| **4** | Gateway → Backend | Gateway xử lý và gửi telemetry tới backend tương ứng |
| **5** | Grafana / UI → Backend | Người dùng truy vấn và quan sát dữ liệu |

---

## 4.4 Tóm tắt vai trò từng khối theo tư duy dễ nhớ

| Khối | Vai trò dễ nhớ |
|---|---|
| **Application** | Nơi **phát sinh dữ liệu** |
| **Instrumentation** | Nơi **gắn khả năng observability** |
| **Agent** | Nơi **thu dữ liệu gần nguồn** |
| **Gateway** | Nơi **xử lý tập trung** |
| **Backend** | Nơi **lưu và hiển thị dữ liệu** |

---

## 4.5 Kết luận

> Kiến trúc OpenTelemetry production nên được hiểu theo mô hình:

```text
Application → Instrumentation → Agent → Gateway → Backend