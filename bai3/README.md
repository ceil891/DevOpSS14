# Bài 3: Vận dụng chuyên sâu - "Đứng trên vai người khổng lồ (Import Template)"

## 1. Mục tiêu kiến thức
- Sử dụng kho tài nguyên Grafana Dashboards cộng đồng để Import nhanh Dashboard thông qua Dashboard ID.
- Nắm bắt các chỉ số giám sát cốt lõi của máy ảo Java (JVM): Heap Memory, Garbage Collection, JVM Threads.
- Khắc phục lỗi phổ biến "No Data" do thiếu cấu hình Common Metrics Tags trong Spring Boot.

---

## 2. Tình huống nghiệp vụ
Đội ngũ phát triển cần giám sát 5 chỉ số sống còn của ứng dụng Spring Boot trước ngày Launch sản phẩm:
1. Heap Memory
2. Non-Heap Memory
3. Garbage Collection Time (Thời gian dừng GC)
4. Thread Count
5. HTTP Request Latency

Việc tự thiết kế và vẽ thủ công từng biểu đồ, từng câu lệnh PromQL phức tạp sẽ tiêu tốn rất nhiều thời gian (vài ngày đến cả tuần). Giải pháp chuẩn DevOps là "Đứng trên vai người khổng lồ": Tận dụng các Dashboard mẫu chuẩn mực đã được cộng đồng Grafana Labs tối ưu hóa và kiểm định.

---

## 3. Thao tác Import Dashboard bằng Dashboard ID (ID: 4701)

Template nổi tiếng và phổ biến nhất thế giới dành cho Spring Boot Micrometer là:
- **Tên Dashboard**: `JVM (Micrometer)`
- **Dashboard ID**: `4701` (Link: https://grafana.com/grafana/dashboards/4701-jvm-micrometer/)

### Các bước Import từng bước:
1. Mở Grafana (`http://localhost:3000`).
2. Trên thanh menu bên trái, nhấp vào **Dashboards**.
3. Ở góc trên bên phải, nhấn nút **New** > chọn **Import** (hoặc truy cập thẳng đường dẫn `http://localhost:3000/dashboard/import`).
4. Tại ô **Import via grafana.com**, điền số ID:
   ```text
   4701
   ```
5. Nhấn nút **Load**.
6. Giao diện cấu hình Import hiện ra:
   - **Name**: Giữ nguyên hoặc đổi tên (ví dụ: `Production JVM Monitor`).
   - **Folder**: Chọn thư mục lưu (mặc định là *General*).
   - **Prometheus**: Ở mục chọn Data Source dưới đáy, bấm chọn data source **Prometheus** (đã thiết lập ở Bài 1).
7. Nhấn nút **Import**. Ngay lập tức toàn bộ hệ thống dashboard chuyên nghiệp với hàng chục panel trực quan sẽ xuất hiện.

---

## 4. Ba biểu đồ quan trọng nhất đối với ứng dụng Java & Ý nghĩa

### Biểu đồ 1: JVM Heap Memory (Committed vs Used vs Max)
- **Metric sử dụng**:
  - `jvm_memory_used_bytes{area="heap"}` (Bộ nhớ thực tế đang dùng)
  - `jvm_memory_committed_bytes{area="heap"}` (Bộ nhớ OS đã bàn giao cho JVM)
  - `jvm_memory_max_bytes{area="heap"}` (Ngưỡng trần tối đa cấu hình qua tham số `-Xmx`)
- **Ý nghĩa & Tầm quan trọng**:
  - Theo dõi mức tiêu thụ vùng nhớ đối tượng của Java runtime.
  - **Dấu hiệu cảnh báo**: Khi lượng `Used Heap` liên tục tăng theo dạng "răng cưa dốc đứng" và sau các đợt Garbage Collection vẫn không hạ xuống, đây là bằng chứng rõ ràng của hiện tượng **Memory Leak** (rò rỉ bộ nhớ do giữ tham chiếu đối tượng không giải phóng).
  - Nếu `Used` chạm trần `Max`, ứng dụng sẽ crash lập tức với lỗi chí mạng:
    ```text
    java.lang.OutOfMemoryError: Java heap space
    ```

---

### Biểu đồ 2: Garbage Collection (GC Time / Stop-the-World Pauses)
- **Metric sử dụng**:
  - `jvm_gc_pause_seconds_sum` / `jvm_gc_pause_seconds_count`
  - `jvm_gc_pause_seconds_max`
- **Ý nghĩa & Tầm quan trọng**:
  - Khi JVM thực hiện Garbage Collection (đặc biệt là Major/Full GC), nó phải tạm dừng mọi luồng thực thi của ứng dụng (**Stop-the-World**).
  - Biểu đồ này đo tổng thời gian và thời gian dừng lớn nhất mà ứng dụng phải "đứng hình" để dọn rác bộ nhớ.
  - **Dấu hiệu cảnh báo**: Thời gian GC pause kéo dài (vượt quá 500ms - vài giây) hoặc tần suất GC quá dày đặc (GC Thrashing). Người dùng bên ngoài sẽ cảm nhận thấy hệ thống bị chậm đột ngột (high latency spike) hoặc timeout giao dịch.

---

### Biểu đồ 3: JVM Threads (Live, Daemon, Peak Threads)
- **Metric sử dụng**:
  - `jvm_threads_live_threads`
  - `jvm_threads_daemon_threads`
  - `jvm_threads_peak_threads`
- **Ý nghĩa & Tầm quan trọng**:
  - Giám sát số lượng tiểu trình (thread) đang chạy đồng thời bên trong JVM, bao gồm worker threads của Web Server (Tomcat/Netty) và các luồng tác vụ nền.
  - **Dấu hiệu cảnh báo**:
    - **Thread Leak**: Số lượng luồng tăng liên tục không ngừng theo thời gian (ví dụ do code tự tạo `new Thread()` hoặc sử dụng ExecutorService sai cách không shutdown).
    - **Cạn kiệt Thread Pool (Starvation)**: Khi số live threads chạm kịch kim cấu hình `max-threads` của Tomcat, server sẽ không thể tiếp nhận thêm bất kỳ HTTP request nào mới, dẫn đến treo hàng đợi (504 Gateway Timeout).

---

## 5. Giải thích Bẫy: Lỗi "No Data" và biến `$application` bị rỗng

### 1. Hiện tượng:
Sau khi import thành công dashboard 4701, toàn bộ các panel hiển thị thông báo **"No data"**. Nhìn lên thanh filter đầu trang, dropdown biến `application` hoàn toàn trống rỗng không có giá trị để chọn.

### 2. Nguyên nhân cốt lõi:
- Trong mã nguồn của Dashboard 4701, biến `application` được truy vấn bằng PromQL:
  ```promql
  label_values(jvm_classes_loaded_classes, application)
  ```
- Các panel biểu đồ đều lọc dữ liệu có điều kiện nhãn:
  `jvm_memory_used_bytes{application="$application", ...}`
- Mặc định, thư viện Spring Boot Actuator kết hợp Micrometer **KHÔNG tự động chèn tag `application`** vào các metrics Prometheus.
- Do đó, trong cơ sở dữ liệu của Prometheus, các metrics thu thập được không hề có label `application`. Biến `$application` rỗng $\rightarrow$ Điều kiện lọc bị sai $\rightarrow$ Mọi panel đều báo **No Data**.

---

### 3. Giải pháp khắc phục trong file cấu hình Spring Boot:

Mở file [application.yml](file:///E:/BaiTapDeVop/SS14/bai3/application.yml) của dự án Spring Boot và bổ sung cấu hình:

```yaml
spring:
  application:
    name: ecommerce-api   # Định danh tên ứng dụng

management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
  metrics:
    tags:
      # KHẮC PHỤC TRIỆT ĐỂ LỖI NO DATA:
      # Định nghĩa common tag "application" cho toàn bộ metrics
      application: ${spring.application.name}
```

### 4. Cơ chế hoạt động:
Khi khai báo `management.metrics.tags.application: ${spring.application.name}`, thư viện Micrometer sẽ tự động tiêm thêm nhãn `application="ecommerce-api"` vào **tất cả** các metric phơi bày qua endpoint `/actuator/prometheus`:

```text
# Dữ liệu xuất ra sau khi cấu hình:
jvm_memory_used_bytes{area="heap", application="ecommerce-api", id="G1 Eden Space"} 1.834928E7
```

Khi đó, biến `$application` trên Grafana sẽ tự động nhận diện giá trị `ecommerce-api`, và tất cả biểu đồ sẽ hiển thị dữ liệu đầy đủ.
