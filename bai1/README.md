# Bài 1: Vận dụng cơ bản - "Giao tiếp giữa 2 người khổng lồ"

## 1. Mục tiêu kiến thức
- Cài đặt Grafana qua Docker Compose cùng với cụm giám sát Prometheus.
- Cấu hình Data Source kết nối Grafana với Prometheus.
- Hiểu sâu về kiến trúc mạng trong Docker Compose và cơ chế phân giải tên miền nội bộ (Docker Embedded DNS).

---

## 2. Tình huống nghiệp vụ
Hệ thống giám sát Prometheus đã được cấu hình để thu thập metrics, tuy nhiên giao diện mặc định của Prometheus (`:9090`) khá thô sơ, khó trực quan hóa theo thời gian thực và không phù hợp để trình bày cho đội ngũ Business/Management. Tech Lead yêu cầu tích hợp Grafana vào hạ tầng hiện tại để dựng dashboard trực quan, sinh động.

---

## 3. Cấu hình Docker Compose

File [docker-compose.yml](file:///E:/BaiTapDeVop/SS14/bai1/docker-compose.yml):

```yaml
services:
  prometheus:
    image: prom/prometheus:v2.51.0
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--web.enable-lifecycle"
    networks:
      - monitoring-net

  grafana:
    image: grafana/grafana:10.4.0
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-storage:/var/lib/grafana
    depends_on:
      - prometheus
    networks:
      - monitoring-net

volumes:
  prometheus-data:
  grafana-storage:

networks:
  monitoring-net:
    driver: bridge
```

### Điểm kỹ thuật quan trọng trong Compose:
1. **Port `3000:3000`**: Expose cổng 3000 của Grafana ra máy Host để người dùng truy cập giao diện qua trình duyệt.
2. **Volume `grafana-storage`**: Mount vào `/var/lib/grafana` để lưu trữ cơ sở dữ liệu nội bộ của Grafana (SQLite chứa dashboards, data sources, users). Nếu không có volume này, mọi cấu hình sẽ mất sạch khi container bị xóa/tạo lại.
3. **Mạng chung `monitoring-net`**: Cả `prometheus` và `grafana` cùng tham gia bridge network do người dùng định nghĩa để có thể giao tiếp qua service name.

---

## 4. Các bước thêm Prometheus làm Data Source trên Grafana

Sau khi chạy `docker compose up -d`:

1. **Truy cập Grafana**:
   - Mở trình duyệt vào địa chỉ: `http://localhost:3000`
   - Đăng nhập với tài khoản:
     - **Username**: `admin`
     - **Password**: `admin`
   - (Grafana có thể yêu cầu đổi mật khẩu mới, chọn *Skip* hoặc đặt mật khẩu mới).

2. **Điều hướng đến Data Sources**:
   - Ở thanh điều hướng bên trái, nhấp vào biểu tượng menu hoặc icon bánh răng **Connections** (hoặc **Administration**) > **Data sources**.
   - Nhấn nút **Add data source**.

3. **Chọn Prometheus**:
   - Trong danh sách các loại Data Source, tìm và chọn **Prometheus**.

4. **Cấu hình Connection**:
   - Tại mục **Prometheus server URL**, nhập:
     ```text
     http://prometheus:9090
     ```
   - Các thông số khác để mặc định (Access: *Server (default)*, HTTP Method: *POST* hoặc *GET*).

5. **Lưu và kiểm tra kết nối**:
   - Cuộn xuống dưới cùng và nhấn nút **Save & test**.
   - Thông báo màu xanh lá xuất hiện: **"Data source is working"** (hoặc "Successfully queried the Prometheus API") chứng tỏ kết nối thành công.

---

## 5. Giải thích Bẫy: `http://localhost:9090` hay `http://prometheus:9090`?

### Câu trả lời chính xác:
> **BẮT BUỘC phải điền `http://prometheus:9090`**, KHÔNG ĐƯỢC điền `http://localhost:9090`.

---

### Phân tích chi tiết dựa trên kiến trúc mạng Docker Compose:

#### 1. Góc nhìn Network Namespace của Container
- Mỗi container trong Docker là một môi trường bị cô lập hoàn toàn (Network Namespace độc lập), sở hữu một card mạng ảo, một dải IP riêng và một loopback interface riêng (`127.0.0.1` hay `localhost`).
- Khi backend của Grafana nhận lệnh truy vấn dữ liệu từ người dùng, nó phải tự mình gửi HTTP request đến server Prometheus.
- Nếu bạn nhập `http://localhost:9090`:
  - Grafana sẽ gửi request tới chính **loopback của container Grafana** (`127.0.0.1:9090`).
  - Trong container Grafana, cổng 9090 hoàn toàn không có tiến trình nào lắng nghe (chỉ có tiến trình Grafana đang lắng nghe tại port 3000).
  - Kết quả: Grafana báo lỗi `HTTP Error Bad Gateway (502)` hoặc `dial tcp 127.0.0.1:9090: connect: connection refused`.

#### 2. Cơ chế Docker Embedded DNS trong User-Defined Bridge Network
- Khi các service cùng khai báo trong một `docker-compose.yml` (hoặc cùng join một custom network), Docker tự động khởi chạy một máy chủ DNS nội bộ tại địa chỉ `127.0.0.11`.
- DNS này cho phép các container **phân giải tên service thành IP nội bộ** của container tương ứng.
- Khi Grafana gọi tới `http://prometheus:9090`:
  1. Grafana hỏi Docker DNS: *"Tên miền `prometheus` có IP là gì?"*
  2. Docker DNS trả về địa chỉ IP nội bộ của container Prometheus (ví dụ: `172.20.0.2`).
  3. Grafana kết nối trực tiếp đến `172.20.0.2:9090` thành công qua mạng bridge nội bộ.

#### 3. Bảng so sánh 2 góc nhìn mạng

| Góc nhìn kết nối | Đối tượng gửi request | URL đúng | Lý do |
|---|---|---|---|
| **Người dùng (Browser máy Host)** | Web Browser trên Laptop/PC | `http://localhost:9090` | Máy Host map port `9090` của container ra cổng `9090` của máy Host |
| **Grafana gọi Prometheus** | Container Grafana | `http://prometheus:9090` | Hai container giao tiếp nội bộ trong mạng Docker qua Service Name |

---

## 6. Sơ đồ luồng giao tiếp

```text
[ Trình duyệt người dùng (Host) ]
       │                    │
       │ :3000              │ :9090 (để debug trực tiếp)
       ▼                    ▼
┌────────────────── Docker Bridge Network ──────────────────┐
│                                                           │
│   ┌─────────────────────┐         ┌───────────────────┐   │
│   │  Container Grafana  │         │ Container         │   │
│   │  (:3000)            │────────▶│ Prometheus (:9090)│   │
│   └─────────────────────┘         └───────────────────┘   │
│              http://prometheus:9090                       │
│       (Docker Embedded DNS phân giải tên)                 │
└───────────────────────────────────────────────────────────┘
```
