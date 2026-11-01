# SS14 - Grafana & Advanced Visualization

Thực hành chuyên sâu về trực quan hóa dữ liệu giám sát hệ thống với **Grafana**, kết nối **Prometheus Data Source**, **PromQL Visualization**, **Community Templates (JVM Micrometer)**, và **Dynamic Variables**.

---

## Cấu trúc thư mục

```text
SS14/
├── bai1/
│   ├── docker-compose.yml ← Cụm Prometheus + Grafana với volume và network
│   ├── prometheus.yml     ← Cấu hình scrape Prometheus cơ bản
│   └── README.md          ← Phân tích bẫy Docker DNS: http://prometheus:9090 vs localhost
├── bai2/
│   ├── dashboard-vps.json ← File JSON Dashboard mẫu gồm Time series CPU & Stat Free RAM
│   └── README.md          ← Công thức PromQL % CPU, Stat RAM, và cách dùng Standard Options Unit
├── bai3/
│   ├── application.yml    ← Cấu hình Spring Boot Actuator thêm common tag application
│   └── README.md          ← Hướng dẫn Import Dashboard 4701, giải thích 3 biểu đồ JVM & fix lỗi No Data
└── bai4/
    ├── docker-compose.yml ← Mô phỏng cụm 3 server Node Exporter (app, db, cache) + Prometheus + Grafana
    ├── prometheus.yml     ← Scrape targets cho 3 máy chủ
    └── README.md          ← Hướng dẫn tạo Dynamic Variable với label_values(), giải quyết bẫy hardcode
```

---

## Tóm tắt kiến thức cốt lõi

| Bài | Tên bài | Điểm mấu chốt & Bẫy kỹ thuật |
|---|---|---|
| **Bài 1** | Giao tiếp giữa 2 người khổng lồ | **Bẫy URL:** Grafana kết nối Prometheus qua Docker network phải dùng `http://prometheus:9090` (nhờ Docker Embedded DNS). Dùng `localhost:9090` sẽ trỏ vào chính loopback của Grafana container $\rightarrow$ lỗi `Connection refused`. |
| **Bài 2** | Vẽ biểu đồ sức khỏe VPS | **Bẫy Đơn vị:** Metric `node_memory_MemFree_bytes` trả về Bytes thô. KHÔNG chia `/ 1024 / 1024` trong PromQL; thay vào đó cấu hình **Standard options > Unit: Data > bytes (IEC)** để Grafana tự động scale thông minh (MiB, GiB). CPU usage dùng công thức: `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`. |
| **Bài 3** | Đứng trên vai người khổng lồ (Import Template) | **Bẫy No Data:** Import JVM Dashboard ID `4701`. Bị lỗi rỗng biến `$application` do Spring Boot thiếu `management.metrics.tags.application: ${spring.application.name}`. Ba biểu đồ JVM cốt lõi: Heap Memory (phát hiện OOM/Memory leak), GC Pauses (ảnh hưởng Stop-the-world/Latency), JVM Threads (phát hiện Thread leak/Deadlock). |
| **Bài 4** | Bảng điều khiển động (Variables) | **Bẫy Hardcode:** Thay thế chuỗi cứng `{instance="192.168.1.10:9100"}` bằng biến `{instance=~"$instance"}`. Tạo biến Query dùng hàm `label_values(node_uname_info, instance)` để tự động quét danh sách server trong cụm. |
