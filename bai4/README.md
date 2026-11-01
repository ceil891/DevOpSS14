# Bài 4: Vận dụng chuyên sâu - "Bảng điều khiển động (Variables)"

## 1. Mục tiêu kiến thức
- Hiểu và áp dụng cơ chế **Template Variables** trong Grafana để biến dashboard tĩnh thành dashboard động đa năng.
- Sử dụng hàm truy vấn `label_values()` trong Grafana Variable để tự động lấy danh sách máy chủ từ Prometheus.
- Xóa bỏ việc hardcode cứng địa chỉ IP/instance trong câu lệnh PromQL, hỗ trợ lọc đơn hoặc lọc nhiều máy chủ.

---

## 2. Tình huống nghiệp vụ
Doanh nghiệp mở rộng hạ tầng, hiện đang vận hành 3 máy chủ độc lập chạy Node Exporter:
- `app-server-1` (Máy chủ chạy Web/API Application)
- `db-server-1` (Máy chủ cơ sở dữ liệu)
- `cache-server-1` (Máy chủ Redis/Memcached)

Sếp phàn nàn rằng mỗi lần muốn kiểm tra tài nguyên của từng máy, ông phải mở 3 tab trình duyệt với 3 dashboard riêng biệt, vừa tốn thời gian vừa khó so sánh. Sếp muốn **duy nhất 1 Dashboard tập trung**, có **menu sổ xuống (dropdown)** để khi chọn máy chủ nào thì toàn bộ biểu đồ (CPU, RAM, Disk, Network) tự động cập nhật theo máy chủ đó.

---

## 3. Các bước tạo Variable `instance` loại `Query`

### Bước 1: Mở cài đặt Dashboard Variables
1. Mở Dashboard bạn muốn cấu hình (hoặc tạo dashboard mới).
2. Nhấp vào biểu tượng bánh răng **Settings** (góc trên bên phải màn hình).
3. Ở menu bên trái của trang Settings, chọn thẻ **Variables**.
4. Nhấn nút **Add variable** (hoặc **New variable** nếu chưa có biến nào).

---

### Bước 2: Thiết lập thông số cho Variable
Điền các giá trị vào form cấu hình như sau:

| Thuộc tính | Giá trị cấu hình | Ý nghĩa |
|---|---|---|
| **Name** | `instance` | Tên biến sẽ dùng trong PromQL (gọi bằng cú pháp `$instance`) |
| **Label** | `Server / Instance` | Tên hiển thị thân thiện trên thanh dropdown cho người dùng xem |
| **Description** | `Chọn máy chủ cần giám sát` | Chú thích hướng dẫn |
| **Type** | `Query` | Lấy dữ liệu động từ Data Source qua câu lệnh truy vấn |
| **Data source** | `Prometheus` | Nguồn dữ liệu chứa nhãn các máy chủ |
| **Refresh** | `On dashboard load` | Tự động làm mới danh sách máy chủ khi tải lại dashboard |

---

### Bước 3: Viết câu truy vấn lấy danh sách máy chủ
Tại ô **Query** (hoặc Label values query), nhập một trong các câu lệnh PromQL sau:

#### Câu truy vấn tối ưu nhất:
```promql
label_values(node_uname_info, instance)
```

*(Hoặc nếu chỉ muốn lấy các máy thuộc job `node-exporter`:)*
```promql
label_values(up{job="node-exporter"}, instance)
```

*(Hoặc dựa trên metric RAM):*
```promql
label_values(node_memory_MemFree_bytes, instance)
```

#### Giải thích hàm `label_values(metric, label)`:
- `label_values()` là hàm nội tại của Grafana dành riêng cho Prometheus Data Source.
- Hàm này gửi truy vấn tới Prometheus API để trích xuất toàn bộ danh sách các giá trị duy nhất (unique values) của nhãn `instance` gắn liền với metric `node_uname_info` (hoặc `node_memory_MemFree_bytes`).
- Kết quả thu được sẽ là mảng các máy chủ đang hoạt động: `["app-server-1:9100", "db-server-1:9100", "cache-server-1:9100"]`.
- Khi có một server mới được thêm vào Prometheus, Grafana sẽ **tự động cập nhật vào dropdown** mà không cần sửa code dashboard!

---

### Bước 4: Tùy chọn lựa chọn (Selection Options)
- **Multi-value**: `BẬT (Checked)` $\rightarrow$ Cho phép người dùng chọn cùng lúc nhiều server (ví dụ: vừa xem `app-server-1` vừa so sánh với `db-server-1`).
- **Include All option**: `BẬT (Checked)` $\rightarrow$ Thêm tùy chọn "All" trên dropdown để xem toàn bộ server cùng lúc.

Kiểm tra phần **Preview of values** ở cuối trang: Thấy danh sách các instance hiện ra đầy đủ $\rightarrow$ Nhấn **Apply** (hoặc **Save dashboard**).

---

## 4. Sửa đổi biểu đồ đồ thị RAM nhận biến `$instance`

### 1. Hiện trạng ban đầu (Bị bẫy do Hardcode):
Biểu đồ cũ đang gắn chết địa chỉ IP của một server cố định:
```promql
node_memory_MemFree_bytes{instance="192.168.1.10:9100"}
```
> **Hậu quả**: Dù người dùng có chọn bất kỳ server nào trên menu dropdown ở trên, biểu đồ vẫn "trơ trơ" chỉ vẽ dữ liệu của máy `192.168.1.10`.

---

### 2. Viết lại câu PromQL có chứa biến động:

#### Trường hợp 1: Chỉ chọn 1 server đơn lẻ (Single Select)
```promql
node_memory_MemFree_bytes{instance="$instance"}
```
- Khi người dùng chọn `app-server-1:9100`, Grafana tự động thay thế chuỗi `$instance` thành:
  `node_memory_MemFree_bytes{instance="app-server-1:9100"}`.

#### Trường hợp 2: Hỗ trợ Multi-value (Chọn nhiều máy hoặc chọn "All") - Khuyên dùng chuẩn DevOps
Nếu đã bật tùy chọn **Multi-value** hoặc **Include All option**, khi người dùng chọn nhiều máy chủ (hoặc chọn All), Grafana sẽ nội suy `$instance` thành một chuỗi biểu thức chính quy (Regex) nối bằng dấu `|`:
`app-server-1:9100|db-server-1:9100`

Nếu dùng dấu so sánh bằng tuyệt đối (`instance="$instance"`), Prometheus sẽ báo lỗi hoặc không tìm thấy nhãn nào có giá trị ghép đó.

Do đó, bắt buộc phải sử dụng **toán tử so khớp Regex `=~`**:
```promql
node_memory_MemFree_bytes{instance=~"^$instance$"}
```
hoặc viết gọn:
```promql
node_memory_MemFree_bytes{instance=~"$instance"}
```

---

## 5. Mở rộng nâng cao: Tính năng Panel Repeat (Nhân bản biểu đồ tự động)

Nếu Sếp muốn khi chọn "All", Grafana không gom chung các đường vào 1 biểu đồ rối mắt mà tự động tách thành 3 biểu đồ riêng biệt:
1. Mở Panel Edit của biểu đồ RAM.
2. Tìm đến mục **Panel options** > **Repeat by variable** (hoặc **Repeat options**).
3. Chọn:
   - **Repeat by variable**: `instance`
   - **Direction**: `Horizontal` hoặc `Vertical`
   - **Max per row**: `2` hoặc `3`
4. Nhấn **Apply**.
5. Bây giờ, khi Sếp tick chọn 3 máy chủ trên dropdown, Grafana sẽ tự động render ra đúng 3 ô biểu đồ RAM tương ứng cho từng máy chủ cực kỳ trực quan và chuyên nghiệp.

---

## 6. Mô phỏng thử nghiệm thực tế với Docker Compose

File [docker-compose.yml](file:///E:/BaiTapDeVop/SS14/bai4/docker-compose.yml) và [prometheus.yml](file:///E:/BaiTapDeVop/SS14/bai4/prometheus.yml) đi kèm đã khai báo sẵn cụm 3 Node Exporters:
```bash
# Khởi động cụm 3 server giả lập và Prometheus + Grafana:
docker compose up -d

# Kiểm tra trạng thái các container:
docker compose ps
```
Truy cập Grafana tại `http://localhost:3000`, tạo biến như hướng dẫn trên và kiểm nghiệm menu dropdown hoạt động mượt mà.
